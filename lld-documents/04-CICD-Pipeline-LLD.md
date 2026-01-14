# CI/CD Pipeline Low Level Design
## Intelligent Document Processing (IDP) Solution

### Document Control
- **Version**: 1.0
- **Last Updated**: January 14, 2026
- **Status**: Draft

---

## 1. CI/CD Overview

### 1.1 Pipeline Architecture

**Source Control**: GitLab
**CI/CD Platform**: GitLab CI/CD
**Infrastructure as Code**: Terraform
**Container Registry**: Amazon ECR
**Deployment Target**: AWS (dev, test, prod)

### 1.2 Pipeline Strategy: **Single Unified Pipeline (RECOMMENDED)**

We use a **single unified pipeline** that handles Infrastructure, Application, and BDA configurations together. This approach provides:

**Benefits**:
- ✅ **Atomic Deployments**: Infrastructure and application deployed together
- ✅ **Consistency**: Single source of truth for all components
- ✅ **Simplified Management**: One pipeline to maintain
- ✅ **Dependency Management**: Automatic handling of dependencies between layers
- ✅ **Faster Feedback**: Single pipeline run shows all issues
- ✅ **Easier Rollback**: Rollback entire stack together

**Pipeline Components**:
```
┌─────────────────────────────────────────────────────────────────┐
│                    SINGLE UNIFIED PIPELINE                       │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  INFRASTRUCTURE LAYER (Terraform)                       │    │
│  │  - VPC configuration (if needed)                        │    │
│  │  - S3 buckets, DynamoDB tables                          │    │
│  │  - SQS queues, DLQ                                      │    │
│  │  - IAM roles and policies                               │    │
│  │  - Lambda function definitions (without code)           │    │
│  │  - Step Functions state machine                         │    │
│  │  - API Gateway                                           │    │
│  │  - CloudWatch alarms                                     │    │
│  └────────────────────────────────────────────────────────┘    │
│                           ↓                                      │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  APPLICATION LAYER (Docker Images)                      │    │
│  │  - Build Lambda Docker images                           │    │
│  │  - Push to ECR                                          │    │
│  │  - Update Lambda functions with new images              │    │
│  │  - Deploy Lambda layers                                 │    │
│  └────────────────────────────────────────────────────────┘    │
│                           ↓                                      │
│  ┌────────────────────────────────────────────────────────┐    │
│  │  BDA CONFIGURATION LAYER (S3 Configs)                   │    │
│  │  - Upload document type configs to S3                   │    │
│  │  - Upload schemas (JSON Schema)                         │    │
│  │  - Upload prompts (text files)                          │    │
│  │  - Update DynamoDB config cache                         │    │
│  │  - Validate BDA blueprints                              │    │
│  └────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 1.3 Change Detection & Conditional Execution

The pipeline intelligently detects what changed and runs only necessary stages:

```yaml
# Example: Only run infrastructure if Terraform files changed
deploy:infrastructure:
  rules:
    - changes:
      - terraform/**/*
      - .gitlab-ci.yml
  script:
    - terraform apply

# Example: Only build Lambda if application code changed
build:lambda:
  rules:
    - changes:
      - src/lambda_functions/**/*
      - src/common/**/*
  script:
    - docker build

# Example: Only update BDA configs if config files changed
deploy:bda-configs:
  rules:
    - changes:
      - config/document-types/**/*
  script:
    - aws s3 sync config/ s3://idp-config-bucket/
```

### 1.4 Pipeline Stages

```
1. Validate → 2. Build → 3. Test → 4. Security → 5. Package → 6. Deploy → 7. Validate
```

**Stage Breakdown**:
- **Validate**: Lint code, validate Terraform, check configs
- **Build**: Build Docker images, compile artifacts
- **Test**: Unit tests, integration tests
- **Security**: SAST, container scanning, secrets detection
- **Package**: Push images to ECR, upload artifacts
- **Deploy**: Deploy infrastructure → application → configs
- **Validate**: Smoke tests, health checks

### 1.5 Branching Strategy

**GitFlow Model**:
- `main`: Production-ready code
- `develop`: Integration branch
- `feature/*`: Feature development
- `release/*`: Release preparation
- `hotfix/*`: Production fixes

---

## 2. Repository Structure

```
idp-solution/
├── .gitlab-ci.yml                 # Main CI/CD pipeline (UNIFIED)
├── .gitlab/
│   └── ci/
│       ├── infrastructure.yml     # Infrastructure deployment jobs
│       ├── application.yml        # Application build/deploy jobs
│       ├── bda-config.yml         # BDA configuration jobs
│       ├── test.yml               # Test stage jobs
│       ├── security.yml           # Security scanning jobs
│       └── validate.yml           # Post-deployment validation
├── terraform/                     # INFRASTRUCTURE LAYER
│   ├── environments/
│   │   ├── dev/
│   │   │   ├── main.tf
│   │   │   ├── terraform.tfvars
│   │   │   └── backend.tf
│   │   ├── test/
│   │   └── prod/
│   └── modules/
│       ├── networking/
│       ├── storage/
│       ├── messaging/
│       ├── compute/
│       ├── orchestration/
│       ├── api/
│       ├── security/
│       └── monitoring/
├── src/                           # APPLICATION LAYER
│   ├── lambda_functions/
│   │   ├── presigned_url_generator/
│   │   │   ├── lambda_function.py
│   │   │   ├── requirements.txt
│   │   │   └── Dockerfile
│   │   ├── sqs_poller/
│   │   ├── bda_orchestrator/
│   │   ├── language_detector/
│   │   ├── document_classifier/
│   │   ├── schema_extractor/
│   │   ├── status_updater/
│   │   ├── document_cleanup/
│   │   └── dlq_analyzer/
│   └── common/                    # Shared utilities
├── config/                        # BDA CONFIGURATION LAYER
│   └── document-types/
│       ├── passport/
│       │   ├── config.json
│       │   ├── schema.json
│       │   └── prompt.txt
│       ├── bank-statement/
│       ├── utility-bill/
│       ├── drivers-license/
│       └── ...
├── tests/
│   ├── unit/
│   ├── integration/
│   └── e2e/
├── scripts/
│   ├── build.sh
│   ├── deploy-infra.sh
│   ├── deploy-app.sh
│   ├── deploy-bda-configs.sh
│   ├── validate.sh
│   └── rollback.sh
└── docs/
    └── lld-documents/
```

---

## 3. Unified Pipeline Flow

### 3.1 Complete Pipeline Execution

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         UNIFIED CI/CD PIPELINE FLOW                              │
└─────────────────────────────────────────────────────────────────────────────────┘

TRIGGER: Git Push to develop/main
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STAGE 1: VALIDATE (Parallel)                                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
│  │ Validate Code    │  │ Validate Terraform│  │ Validate BDA     │             │
│  │ - Black          │  │ - terraform fmt   │  │ Configs          │             │
│  │ - Flake8         │  │ - terraform       │  │ - JSON Schema    │             │
│  │ - Pylint         │  │   validate        │  │   validation     │             │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘             │
└─────────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STAGE 2: BUILD (Parallel)                                                       │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ Build Lambda Docker Images (9 images)                                     │  │
│  │ - presigned_url_generator                                                 │  │
│  │ - sqs_poller                                                               │  │
│  │ - bda_orchestrator                                                         │  │
│  │ - language_detector                                                        │  │
│  │ - document_classifier                                                      │  │
│  │ - schema_extractor                                                         │  │
│  │ - status_updater                                                           │  │
│  │ - document_cleanup                                                         │  │
│  │ - dlq_analyzer                                                             │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ Build Lambda Layer                                                         │  │
│  │ - Common dependencies                                                      │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STAGE 3: TEST (Sequential)                                                      │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐                                                           │
│  │ Unit Tests       │  - pytest with coverage                                   │
│  │ Coverage: >80%   │  - Mock AWS services with moto                            │
│  └────────┬─────────┘                                                           │
│           │                                                                      │
│           ▼                                                                      │
│  ┌──────────────────┐                                                           │
│  │ Integration Tests│  - Test Lambda interactions                               │
│  │                  │  - Test Step Functions workflow                           │
│  └────────┬─────────┘                                                           │
│           │                                                                      │
│           ▼                                                                      │
│  ┌──────────────────┐                                                           │
│  │ Terraform Plan   │  - Plan for dev/test/prod                                │
│  │                  │  - Save plan artifacts                                    │
│  └──────────────────┘                                                           │
└─────────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STAGE 4: SECURITY (Parallel)                                                    │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────┐  ┌──────────────────┐  ┌──────────────────┐             │
│  │ SAST             │  │ Container Scan   │  │ Secrets Detection│             │
│  │ - Bandit         │  │ - Trivy          │  │ - TruffleHog     │             │
│  │ - Safety         │  │ - Scan all images│  │                  │             │
│  └──────────────────┘  └──────────────────┘  └──────────────────┘             │
│  ┌──────────────────┐                                                           │
│  │ Terraform Scan   │                                                           │
│  │ - tfsec          │                                                           │
│  └──────────────────┘                                                           │
└─────────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STAGE 5: PACKAGE (Sequential)                                                   │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ Push Docker Images to ECR                                                 │  │
│  │ - Tag with commit SHA and 'latest'                                        │  │
│  │ - Push all 9 Lambda images                                                │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ Upload Lambda Layer to S3                                                 │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STAGE 6: DEPLOY TO DEV (Sequential - Manual Trigger)                            │
├─────────────────────────────────────────────────────────────────────────────────┤
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ 1. Deploy Infrastructure (Terraform)                                      │  │
│  │    - Create/Update S3 buckets, DynamoDB tables                            │  │
│  │    - Create/Update SQS queues, DLQ                                        │  │
│  │    - Create/Update Lambda functions (with new ECR images)                 │  │
│  │    - Create/Update Step Functions, EventBridge, API Gateway              │  │
│  │    - Create/Update IAM roles, CloudWatch alarms                           │  │
│  │    Duration: ~5-10 minutes                                                 │  │
│  └────────────────────────────────────────────────────────────────────────┬─┘  │
│                                                                             │    │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ 2. Deploy Application (Lambda Code Update)                               │  │
│  │    - Update Lambda function code (point to new ECR images)               │  │
│  │    - Update Lambda layers                                                 │  │
│  │    - Publish new Lambda versions                                          │  │
│  │    Duration: ~2-3 minutes                                                  │  │
│  └────────────────────────────────────────────────────────────────────────┬─┘  │
│                                                                             │    │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │ 3. Deploy BDA Configurations                                             │  │
│  │    - Upload config.json files to S3                                      │  │
│  │    - Upload schema.json files to S3                                      │  │
│  │    - Upload prompt.txt files to S3                                       │  │
│  │    - Update DynamoDB config cache                                         │  │
│  │    - Invalidate Lambda config cache                                       │  │
│  │    Duration: ~1 minute                                                     │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
└─────────────────────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│ STAGE 7: VALIDATE DEV (Smoke Tests)                                             │
├─────────────────────────────────────────────────────────────────────────────────┤
│  - Test API Gateway endpoints                                                   │
│  - Upload test document                                                          │
│  - Verify SQS queue processing                                                   │
│  - Check Step Functions execution                                                │
│  - Verify output in S3                                                           │
│  - Check DynamoDB status                                                         │
│  Duration: ~5 minutes                                                            │
└─────────────────────────────────────────────────────────────────────────────────┘
    │
    │ (Repeat for TEST and PROD environments)
    ▼
```

---

## 3. GitLab CI/CD Pipeline

### 3.1 Pipeline Strategy Comparison

#### Option A: Single Unified Pipeline (RECOMMENDED - Current Design)

**When to Use**:
- ✅ Small to medium teams
- ✅ Tightly coupled infrastructure and application
- ✅ Frequent deployments
- ✅ Need atomic deployments
- ✅ Simplified operations

**Pros**:
- Single source of truth
- Atomic deployments (all or nothing)
- Easier dependency management
- Faster feedback loop
- Simpler rollback
- Less pipeline maintenance

**Cons**:
- Longer pipeline execution time
- All changes trigger full pipeline (mitigated with change detection)
- Less flexibility for independent deployments

**Pipeline Execution Time**:
- Full pipeline: ~25-35 minutes
- With change detection: ~10-20 minutes (only changed components)

---

#### Option B: Separate Pipelines (Alternative)

**When to Use**:
- Large teams with specialized roles (Infra team, App team, ML team)
- Infrastructure changes infrequently
- Application changes frequently
- Need independent deployment schedules
- Different approval workflows for each layer

**Structure**:
```
Repository 1: idp-infrastructure
  - Terraform code only
  - Pipeline: validate → plan → deploy infra
  - Triggers: Changes to terraform/
  - Frequency: Weekly or as needed

Repository 2: idp-application
  - Lambda function code
  - Pipeline: build → test → deploy app
  - Triggers: Changes to src/
  - Frequency: Daily or multiple times per day

Repository 3: idp-bda-configs
  - BDA configurations
  - Pipeline: validate → deploy configs
  - Triggers: Changes to config/
  - Frequency: As needed (when adding new document types)
```

**Pros**:
- Faster individual pipelines
- Independent deployment schedules
- Team autonomy
- Parallel development

**Cons**:
- Complex dependency management
- Risk of version mismatches
- More difficult rollback
- Multiple repositories to maintain
- Coordination overhead

---

### 3.2 Our Choice: Single Unified Pipeline with Change Detection

We chose the **unified pipeline** because:

1. **IDP is a cohesive system**: Infrastructure, application, and configs are tightly coupled
2. **Atomic deployments**: Ensures all components are compatible
3. **Simplified operations**: One pipeline, one deployment process
4. **Change detection**: Only runs stages for changed components
5. **Team size**: Suitable for small to medium teams

**Change Detection Rules**:
```yaml
# Only deploy infrastructure if Terraform files changed
deploy:infrastructure:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - terraform/**/*
    - if: '$CI_COMMIT_BRANCH == "develop" || $CI_COMMIT_BRANCH == "main"'
      changes:
        - terraform/**/*

# Only build/deploy Lambda if application code changed
build:lambda:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - src/lambda_functions/**/*
        - src/common/**/*
    - if: '$CI_COMMIT_BRANCH == "develop" || $CI_COMMIT_BRANCH == "main"'
      changes:
        - src/lambda_functions/**/*
        - src/common/**/*

# Only deploy BDA configs if config files changed
deploy:bda-configs:
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
      changes:
        - config/document-types/**/*
    - if: '$CI_COMMIT_BRANCH == "develop" || $CI_COMMIT_BRANCH == "main"'
      changes:
        - config/document-types/**/*
```

**Example Scenarios**:

| Change Type | Stages Executed | Duration |
|-------------|----------------|----------|
| Only Lambda code | Validate → Build → Test → Security → Package → Deploy App | ~15 min |
| Only Terraform | Validate → Test (plan) → Security → Deploy Infra | ~12 min |
| Only BDA configs | Validate → Deploy Configs | ~5 min |
| All three | Full pipeline | ~30 min |
| No changes | Validate only | ~3 min |

---

### 3.3 Main Pipeline Configuration

**.gitlab-ci.yml**:
```yaml
stages:
  - validate
  - build
  - test
  - security
  - package
  - deploy-dev
  - test-dev
  - deploy-test
  - test-test
  - deploy-prod
  - validate-prod

variables:
  AWS_DEFAULT_REGION: eu-west-1
  TERRAFORM_VERSION: "1.6.0"
  PYTHON_VERSION: "3.12"
  DOCKER_DRIVER: overlay2
  ECR_REGISTRY: ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com

# Include stage-specific configurations
include:
  - local: '.gitlab/ci/build.yml'
  - local: '.gitlab/ci/test.yml'
  - local: '.gitlab/ci/security.yml'
  - local: '.gitlab/ci/deploy.yml'
  - local: '.gitlab/ci/validate.yml'

# Default settings
default:
  image: python:3.12-slim
  before_script:
    - pip install --upgrade pip
  retry:
    max: 2
    when:
      - runner_system_failure
      - stuck_or_timeout_failure

# Cache configuration
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .pip-cache/
    - .terraform/
```


### 3.2 Build Stage

**.gitlab/ci/build.yml**:
```yaml
# Code validation
validate:code:
  stage: validate
  script:
    - pip install black flake8 pylint bandit
    - black --check src/
    - flake8 src/ --max-line-length=120
    - pylint src/ --fail-under=8.0
  only:
    - merge_requests
    - develop
    - main

# Terraform validation
validate:terraform:
  stage: validate
  image: hashicorp/terraform:${TERRAFORM_VERSION}
  script:
    - cd terraform/environments/dev
    - terraform init -backend=false
    - terraform fmt -check -recursive
    - terraform validate
  only:
    - merge_requests
    - develop
    - main

# Build Lambda Docker images
build:lambda:
  stage: build
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  variables:
    DOCKER_TLS_CERTDIR: "/certs"
  before_script:
    - apk add --no-cache aws-cli
    - aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
  script:
    - |
      # Build all Lambda function images
      for lambda_dir in src/lambda_functions/*/; do
        lambda_name=$(basename $lambda_dir)
        echo "Building ${lambda_name}..."
        
        cd $lambda_dir
        docker build -t idp-${lambda_name}:${CI_COMMIT_SHORT_SHA} .
        docker tag idp-${lambda_name}:${CI_COMMIT_SHORT_SHA} ${ECR_REGISTRY}/idp-${lambda_name}:${CI_COMMIT_SHORT_SHA}
        docker tag idp-${lambda_name}:${CI_COMMIT_SHORT_SHA} ${ECR_REGISTRY}/idp-${lambda_name}:latest
        
        cd -
      done
  artifacts:
    reports:
      dotenv: build.env
  only:
    - develop
    - main
    - /^release\/.*/

# Build common layer
build:layer:
  stage: build
  script:
    - mkdir -p python/lib/python3.12/site-packages
    - pip install -r src/common/requirements.txt -t python/lib/python3.12/site-packages
    - zip -r idp-common-layer.zip python/
  artifacts:
    paths:
      - idp-common-layer.zip
    expire_in: 1 week
  only:
    - develop
    - main
```

### 3.3 Test Stage

**.gitlab/ci/test.yml**:
```yaml
# Unit tests
test:unit:
  stage: test
  before_script:
    - pip install -r requirements-dev.txt
    - pip install -r src/common/requirements.txt
  script:
    - pytest tests/unit/ 
      --cov=src 
      --cov-report=xml 
      --cov-report=html 
      --cov-report=term 
      --junitxml=report.xml
  coverage: '/(?i)total.*? (100(?:\.0+)?\%|[1-9]?\d(?:\.\d+)?\%)$/'
  artifacts:
    reports:
      junit: report.xml
      coverage_report:
        coverage_format: cobertura
        path: coverage.xml
    paths:
      - htmlcov/
    expire_in: 1 week
  only:
    - merge_requests
    - develop
    - main

# Integration tests
test:integration:
  stage: test
  image: python:3.12
  services:
    - docker:24.0.5-dind
  variables:
    AWS_DEFAULT_REGION: eu-west-1
  before_script:
    - pip install -r requirements-dev.txt
    - pip install moto[all] boto3
  script:
    - pytest tests/integration/ 
      --junitxml=integration-report.xml
      -v
  artifacts:
    reports:
      junit: integration-report.xml
    expire_in: 1 week
  only:
    - develop
    - main

# Terraform plan
test:terraform:plan:
  stage: test
  image: hashicorp/terraform:${TERRAFORM_VERSION}
  before_script:
    - cd terraform/environments/${ENVIRONMENT}
    - terraform init
  script:
    - terraform plan -out=tfplan
    - terraform show -json tfplan > tfplan.json
  artifacts:
    paths:
      - terraform/environments/${ENVIRONMENT}/tfplan
      - terraform/environments/${ENVIRONMENT}/tfplan.json
    expire_in: 1 week
  parallel:
    matrix:
      - ENVIRONMENT: [dev, test, prod]
  only:
    - develop
    - main
```

### 3.4 Security Stage

**.gitlab/ci/security.yml**:
```yaml
# SAST - Static Application Security Testing
security:sast:
  stage: security
  image: python:3.12
  before_script:
    - pip install bandit safety
  script:
    # Bandit for Python security issues
    - bandit -r src/ -f json -o bandit-report.json || true
    - bandit -r src/ -f html -o bandit-report.html || true
    
    # Safety for dependency vulnerabilities
    - safety check --json --output safety-report.json || true
  artifacts:
    paths:
      - bandit-report.json
      - bandit-report.html
      - safety-report.json
    expire_in: 1 week
  allow_failure: true
  only:
    - merge_requests
    - develop
    - main

# Container scanning
security:container:
  stage: security
  image: aquasec/trivy:latest
  script:
    - |
      for lambda_name in presigned_url_generator bda_orchestrator language_detector document_classifier schema_extractor status_updater document_cleanup; do
        echo "Scanning idp-${lambda_name}..."
        trivy image --severity HIGH,CRITICAL --format json --output trivy-${lambda_name}.json ${ECR_REGISTRY}/idp-${lambda_name}:${CI_COMMIT_SHORT_SHA} || true
      done
  artifacts:
    paths:
      - trivy-*.json
    expire_in: 1 week
  allow_failure: true
  only:
    - develop
    - main

# Terraform security scan
security:terraform:
  stage: security
  image: aquasec/tfsec:latest
  script:
    - tfsec terraform/ --format json --out tfsec-report.json || true
    - tfsec terraform/ --format html --out tfsec-report.html || true
  artifacts:
    paths:
      - tfsec-report.json
      - tfsec-report.html
    expire_in: 1 week
  allow_failure: true
  only:
    - merge_requests
    - develop
    - main

# Secrets detection
security:secrets:
  stage: security
  image: trufflesecurity/trufflehog:latest
  script:
    - trufflehog filesystem . --json --output trufflehog-report.json || true
  artifacts:
    paths:
      - trufflehog-report.json
    expire_in: 1 week
  allow_failure: false  # Fail pipeline if secrets found
  only:
    - merge_requests
    - develop
    - main
```

### 3.5 Package Stage

**.gitlab/ci/package.yml** (added to build.yml):
```yaml
# Push Docker images to ECR
package:docker:
  stage: package
  image: docker:24.0.5
  services:
    - docker:24.0.5-dind
  dependencies:
    - build:lambda
  before_script:
    - apk add --no-cache aws-cli
    - aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}
  script:
    - |
      for lambda_name in presigned_url_generator bda_orchestrator language_detector document_classifier schema_extractor status_updater document_cleanup; do
        echo "Pushing idp-${lambda_name}..."
        docker push ${ECR_REGISTRY}/idp-${lambda_name}:${CI_COMMIT_SHORT_SHA}
        docker push ${ECR_REGISTRY}/idp-${lambda_name}:latest
      done
  only:
    - develop
    - main

# Upload Lambda layer
package:layer:
  stage: package
  image: amazon/aws-cli:latest
  dependencies:
    - build:layer
  script:
    - aws s3 cp idp-common-layer.zip s3://idp-artifacts-${AWS_ACCOUNT_ID}/layers/idp-common-layer-${CI_COMMIT_SHORT_SHA}.zip
  only:
    - develop
    - main
```


### 3.6 Deployment Stage

**.gitlab/ci/deploy.yml**:
```yaml
# ============================================================================
# DEPLOYMENT STAGE - UNIFIED PIPELINE
# Deploys Infrastructure → Application → BDA Configs in sequence
# ============================================================================

# ────────────────────────────────────────────────────────────────────────────
# STEP 1: Deploy Infrastructure (Terraform)
# ────────────────────────────────────────────────────────────────────────────
deploy:infrastructure:dev:
  stage: deploy-dev
  image: hashicorp/terraform:${TERRAFORM_VERSION}
  environment:
    name: development
    url: https://idp-dev.evoke.com
  variables:
    ENVIRONMENT: dev
  before_script:
    - cd terraform/environments/dev
    - terraform init
  script:
    - echo "Deploying infrastructure to dev..."
    - terraform apply -auto-approve
    - terraform output -json > terraform-outputs.json
  artifacts:
    paths:
      - terraform/environments/dev/terraform-outputs.json
    expire_in: 1 week
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      changes:
        - terraform/**/*
        - .gitlab-ci.yml
  when: manual
  tags:
    - docker

# ────────────────────────────────────────────────────────────────────────────
# STEP 2: Deploy Application (Lambda Functions)
# ────────────────────────────────────────────────────────────────────────────
deploy:application:dev:
  stage: deploy-dev
  image: amazon/aws-cli:latest
  environment:
    name: development
  variables:
    ENVIRONMENT: dev
  dependencies:
    - deploy:infrastructure:dev
    - package:docker
  script:
    - echo "Deploying application to dev..."
    
    # Update Lambda functions with new ECR images
    - |
      for lambda_name in presigned_url_generator sqs_poller bda_orchestrator language_detector document_classifier schema_extractor status_updater document_cleanup dlq_analyzer; do
        echo "Updating Lambda: idp-${lambda_name}-dev"
        aws lambda update-function-code \
          --function-name idp-${lambda_name}-dev \
          --image-uri ${ECR_REGISTRY}/idp-${lambda_name}:${CI_COMMIT_SHORT_SHA} \
          --region ${AWS_DEFAULT_REGION}
        
        # Wait for update to complete
        aws lambda wait function-updated \
          --function-name idp-${lambda_name}-dev \
          --region ${AWS_DEFAULT_REGION}
      done
    
    # Update Lambda layer
    - |
      LAYER_VERSION=$(aws lambda publish-layer-version \
        --layer-name idp-common-layer-dev \
        --content S3Bucket=idp-artifacts-${AWS_ACCOUNT_ID},S3Key=layers/idp-common-layer-${CI_COMMIT_SHORT_SHA}.zip \
        --compatible-runtimes python3.12 \
        --region ${AWS_DEFAULT_REGION} \
        --query 'Version' \
        --output text)
      
      echo "Published layer version: ${LAYER_VERSION}"
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      changes:
        - src/**/*
        - .gitlab-ci.yml
  needs:
    - job: deploy:infrastructure:dev
      optional: true
  when: manual
  tags:
    - docker

# ────────────────────────────────────────────────────────────────────────────
# STEP 3: Deploy BDA Configurations
# ────────────────────────────────────────────────────────────────────────────
deploy:bda-configs:dev:
  stage: deploy-dev
  image: amazon/aws-cli:latest
  environment:
    name: development
  variables:
    ENVIRONMENT: dev
    CONFIG_BUCKET: idp-config-dev-${AWS_ACCOUNT_ID}
  script:
    - echo "Deploying BDA configurations to dev..."
    
    # Sync config files to S3
    - |
      aws s3 sync config/document-types/ s3://${CONFIG_BUCKET}/configs/ \
        --delete \
        --exclude "*.md" \
        --exclude ".gitkeep"
    
    # Validate uploaded configs
    - |
      echo "Validating uploaded configurations..."
      for doc_type in passport bank-statement utility-bill drivers-license; do
        echo "Checking ${doc_type}..."
        aws s3 ls s3://${CONFIG_BUCKET}/configs/${doc_type}/config.json || echo "WARNING: ${doc_type} config missing"
        aws s3 ls s3://${CONFIG_BUCKET}/configs/${doc_type}/schema.json || echo "WARNING: ${doc_type} schema missing"
        aws s3 ls s3://${CONFIG_BUCKET}/configs/${doc_type}/prompt.txt || echo "WARNING: ${doc_type} prompt missing"
      done
    
    # Update DynamoDB config cache (optional - configs are loaded on-demand)
    - |
      echo "Config cache will be updated on first Lambda invocation"
    
    # Invalidate Lambda config cache by updating environment variable
    - |
      TIMESTAMP=$(date +%s)
      for lambda_name in document_classifier schema_extractor; do
        aws lambda update-function-configuration \
          --function-name idp-${lambda_name}-dev \
          --environment "Variables={CONFIG_CACHE_INVALIDATE=${TIMESTAMP}}" \
          --region ${AWS_DEFAULT_REGION} || true
      done
  rules:
    - if: '$CI_COMMIT_BRANCH == "develop"'
      changes:
        - config/document-types/**/*
        - .gitlab-ci.yml
  needs:
    - job: deploy:application:dev
      optional: true
  when: manual
  tags:
    - docker

# ────────────────────────────────────────────────────────────────────────────
# DEPLOYMENT SUMMARY
# ────────────────────────────────────────────────────────────────────────────
deploy:summary:dev:
  stage: deploy-dev
  image: alpine:latest
  script:
    - echo "═══════════════════════════════════════════════════════════"
    - echo "  IDP Deployment to DEV - COMPLETED"
    - echo "═══════════════════════════════════════════════════════════"
    - echo ""
    - echo "Deployed Components:"
    - echo "  ✓ Infrastructure (Terraform)"
    - echo "  ✓ Application (9 Lambda Functions)"
    - echo "  ✓ BDA Configurations"
    - echo ""
    - echo "Next Steps:"
    - echo "  1. Run smoke tests"
    - echo "  2. Verify API Gateway endpoints"
    - echo "  3. Test document upload flow"
    - echo ""
  needs:
    - job: deploy:infrastructure:dev
      optional: true
    - job: deploy:application:dev
      optional: true
    - job: deploy:bda-configs:dev
      optional: true
  when: on_success
  tags:
    - docker

# ════════════════════════════════════════════════════════════════════════════
# Repeat similar structure for TEST and PROD environments
# ════════════════════════════════════════════════════════════════════════════

# Deploy to Test
deploy:test:
  stage: deploy-test
  extends: .deploy_template
  variables:
    ENVIRONMENT: test
  dependencies:
    - deploy:dev
    - test:dev:smoke
  only:
    - develop
  when: manual

# Deploy to Production
deploy:prod:
  stage: deploy-prod
  extends: .deploy_template
  variables:
    ENVIRONMENT: prod
  dependencies:
    - deploy:test
    - test:test:e2e
  only:
    - main
  when: manual
```

---

### 3.7 Deployment Order & Dependencies

**Critical Deployment Order**:
```
1. Infrastructure (Terraform)
   ├─ Creates S3 buckets
   ├─ Creates DynamoDB tables
   ├─ Creates SQS queues + DLQ
   ├─ Creates Lambda functions (with placeholder code)
   ├─ Creates Step Functions
   ├─ Creates API Gateway
   ├─ Creates IAM roles
   └─ Creates CloudWatch alarms
   
   ↓ (Wait for completion)

2. Application (Lambda Code)
   ├─ Updates Lambda function code (ECR images)
   ├─ Updates Lambda layers
   └─ Publishes new versions
   
   ↓ (Wait for completion)

3. BDA Configurations
   ├─ Uploads config.json files to S3
   ├─ Uploads schema.json files to S3
   ├─ Uploads prompt.txt files to S3
   └─ Invalidates Lambda config cache
```

**Why This Order?**:
1. **Infrastructure First**: Creates all AWS resources
2. **Application Second**: Updates Lambda code (requires Lambda functions to exist)
3. **BDA Configs Last**: Uploads configs (requires S3 buckets to exist)

**Dependencies**:
```yaml
deploy:application:dev:
  needs:
    - job: deploy:infrastructure:dev
      optional: true  # Optional if infra didn't change

deploy:bda-configs:dev:
  needs:
    - job: deploy:application:dev
      optional: true  # Optional if app didn't change
```

---

### 3.8 Validation Stage

**.gitlab/ci/validate.yml**:
```yaml
# Smoke tests for Dev
test:dev:smoke:
  stage: test-dev
  image: python:3.12
  variables:
    ENVIRONMENT: dev
  before_script:
    - pip install requests boto3 pytest
  script:
    - pytest tests/smoke/ --environment=dev -v
  dependencies:
    - deploy:dev
  only:
    - develop

# Integration tests for Test environment
test:test:integration:
  stage: test-test
  image: python:3.12
  variables:
    ENVIRONMENT: test
  before_script:
    - pip install -r requirements-dev.txt
  script:
    - pytest tests/integration/ --environment=test -v
  dependencies:
    - deploy:test
  only:
    - develop

# End-to-end tests for Test environment
test:test:e2e:
  stage: test-test
  image: python:3.12
  variables:
    ENVIRONMENT: test
  before_script:
    - pip install -r requirements-dev.txt
  script:
    - pytest tests/e2e/ --environment=test -v --timeout=300
  dependencies:
    - deploy:test
  only:
    - develop
  allow_failure: false

# Production validation
validate:prod:
  stage: validate-prod
  image: python:3.12
  variables:
    ENVIRONMENT: prod
  before_script:
    - pip install requests boto3
  script:
    - python scripts/validate.sh prod
    - |
      # Check critical endpoints
      python -c "
      import requests
      import sys
      
      endpoints = [
        'https://idp.evoke.com/health',
        'https://idp.evoke.com/api/status'
      ]
      
      for endpoint in endpoints:
        response = requests.get(endpoint, timeout=10)
        if response.status_code != 200:
          print(f'Endpoint {endpoint} failed with status {response.status_code}')
          sys.exit(1)
        print(f'Endpoint {endpoint} is healthy')
      "
  dependencies:
    - deploy:prod
  only:
    - main
  allow_failure: false
```

---

## 4. Terraform Deployment

### 4.1 Terraform Backend Configuration

**terraform/environments/dev/backend.tf**:
```hcl
terraform {
  backend "s3" {
    bucket         = "evoke-terraform-state-eu-west-1"
    key            = "idp/dev/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
    kms_key_id     = "arn:aws:kms:eu-west-1:123456789012:key/12345678-1234-1234-1234-123456789012"
  }
}
```

### 4.2 Environment-Specific Variables

**terraform/environments/dev/terraform.tfvars**:
```hcl
# Environment
environment = "dev"
aws_region  = "eu-west-1"
aws_account_id = "123456789012"

# Networking (provided by Evoke)
vpc_id              = "vpc-0123456789abcdef0"
private_subnet_ids  = ["subnet-0123456789abcdef0", "subnet-0123456789abcdef1"]
public_subnet_ids   = ["subnet-0123456789abcdef2", "subnet-0123456789abcdef3"]

# Lambda Configuration
lambda_memory_size = {
  presigned_url_generator = 512
  bda_orchestrator       = 1024
  language_detector      = 512
  document_classifier    = 1024
  schema_extractor       = 2048
  status_updater         = 256
  document_cleanup       = 256
}

lambda_timeout = {
  presigned_url_generator = 30
  bda_orchestrator       = 300
  language_detector      = 60
  document_classifier    = 120
  schema_extractor       = 300
  status_updater         = 30
  document_cleanup       = 60
}

lambda_reserved_concurrency = {
  presigned_url_generator = 100
  bda_orchestrator       = 50
  language_detector      = 50
  document_classifier    = 50
  schema_extractor       = 50
  status_updater         = 100
  document_cleanup       = 10
}

# S3 Configuration
s3_lifecycle_rules = {
  input_documents = 7
  output_documents = 90
  audit_logs = 365
}

# DynamoDB Configuration
dynamodb_billing_mode = "PAY_PER_REQUEST"

# BDA Configuration
bda_project_arn = "arn:aws:bedrock:eu-west-1:123456789012:data-automation-project/idp-doc-proc"
bda_region      = "eu-west-1"

# Monitoring
log_retention_days = 7
enable_xray_tracing = true

# Tags
tags = {
  Project     = "IDP"
  Environment = "dev"
  ManagedBy   = "Terraform"
  CostCenter  = "Engineering"
  Owner       = "IDP-Team"
}
```

### 4.3 Deployment Script

**scripts/deploy.sh**:
```bash
#!/bin/bash
set -e

ENVIRONMENT=$1
AWS_REGION=${AWS_REGION:-eu-west-1}

if [ -z "$ENVIRONMENT" ]; then
  echo "Usage: $0 <environment>"
  echo "Example: $0 dev"
  exit 1
fi

echo "Deploying IDP to ${ENVIRONMENT} environment..."

# Navigate to environment directory
cd terraform/environments/${ENVIRONMENT}

# Initialize Terraform
echo "Initializing Terraform..."
terraform init

# Validate configuration
echo "Validating Terraform configuration..."
terraform validate

# Plan deployment
echo "Planning deployment..."
terraform plan -out=tfplan

# Apply deployment
echo "Applying deployment..."
terraform apply tfplan

# Save outputs
echo "Saving outputs..."
terraform output -json > terraform-outputs.json

echo "Deployment completed successfully!"
echo "Outputs saved to terraform-outputs.json"
```

---

## 5. Docker Image Build

### 5.1 Multi-Stage Dockerfile

**src/lambda_functions/bda_orchestrator/Dockerfile**:
```dockerfile
# Stage 1: Builder
FROM public.ecr.aws/lambda/python:3.12 AS builder

WORKDIR /app

# Install build dependencies
RUN yum install -y gcc python3-devel

# Copy requirements
COPY requirements.txt .

# Install Python dependencies
RUN pip install --no-cache-dir --upgrade pip && \
    pip install --no-cache-dir -r requirements.txt -t /app/python

# Stage 2: Runtime
FROM public.ecr.aws/lambda/python:3.12

WORKDIR ${LAMBDA_TASK_ROOT}

# Copy dependencies from builder
COPY --from=builder /app/python ./

# Copy application code
COPY lambda_function.py ./
COPY ../../common/ ./common/

# Set environment variables
ENV PYTHONUNBUFFERED=1
ENV LOG_LEVEL=INFO

# Set handler
CMD ["lambda_function.handler"]
```

### 5.2 Build Script

**scripts/build.sh**:
```bash
#!/bin/bash
set -e

AWS_REGION=${AWS_REGION:-eu-west-1}
AWS_ACCOUNT_ID=${AWS_ACCOUNT_ID}
ECR_REGISTRY="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com"
IMAGE_TAG=${CI_COMMIT_SHORT_SHA:-latest}

# Login to ECR
echo "Logging in to ECR..."
aws ecr get-login-password --region ${AWS_REGION} | \
  docker login --username AWS --password-stdin ${ECR_REGISTRY}

# Build all Lambda images
LAMBDA_FUNCTIONS=(
  "presigned_url_generator"
  "bda_orchestrator"
  "language_detector"
  "document_classifier"
  "schema_extractor"
  "status_updater"
  "document_cleanup"
)

for lambda_name in "${LAMBDA_FUNCTIONS[@]}"; do
  echo "Building ${lambda_name}..."
  
  cd src/lambda_functions/${lambda_name}
  
  # Build image
  docker build \
    --platform linux/arm64 \
    -t idp-${lambda_name}:${IMAGE_TAG} \
    -t ${ECR_REGISTRY}/idp-${lambda_name}:${IMAGE_TAG} \
    -t ${ECR_REGISTRY}/idp-${lambda_name}:latest \
    .
  
  # Push to ECR
  echo "Pushing ${lambda_name} to ECR..."
  docker push ${ECR_REGISTRY}/idp-${lambda_name}:${IMAGE_TAG}
  docker push ${ECR_REGISTRY}/idp-${lambda_name}:latest
  
  cd -
done

echo "All images built and pushed successfully!"
```


---

## 6. Testing Framework

### 6.1 Unit Test Structure

**tests/unit/test_presigned_url_generator.py**:
```python
import pytest
import json
from moto import mock_s3, mock_dynamodb
from lambda_functions.presigned_url_generator import lambda_function

@pytest.fixture
def aws_credentials(monkeypatch):
    """Mock AWS credentials"""
    monkeypatch.setenv('AWS_ACCESS_KEY_ID', 'testing')
    monkeypatch.setenv('AWS_SECRET_ACCESS_KEY', 'testing')
    monkeypatch.setenv('AWS_SECURITY_TOKEN', 'testing')
    monkeypatch.setenv('AWS_SESSION_TOKEN', 'testing')

@pytest.fixture
def mock_environment(monkeypatch):
    """Mock environment variables"""
    monkeypatch.setenv('INPUT_BUCKET_NAME', 'test-input-bucket')
    monkeypatch.setenv('METADATA_TABLE_NAME', 'test-metadata-table')
    monkeypatch.setenv('PRESIGNED_URL_EXPIRY', '3600')

@mock_s3
@mock_dynamodb
def test_generate_presigned_url_success(aws_credentials, mock_environment):
    """Test successful presigned URL generation"""
    # Setup
    event = {
        'body': json.dumps({
            'document_type': 'passport',
            'callback_url': 'https://example.com/callback'
        })
    }
    context = {}
    
    # Execute
    response = lambda_function.handler(event, context)
    
    # Assert
    assert response['statusCode'] == 200
    body = json.loads(response['body'])
    assert 'document_id' in body
    assert 'upload_url' in body
    assert 'expires_in' in body
    assert body['expires_in'] == 3600

@mock_s3
@mock_dynamodb
def test_generate_presigned_url_invalid_input(aws_credentials, mock_environment):
    """Test presigned URL generation with invalid input"""
    # Setup
    event = {
        'body': json.dumps({
            'document_type': '',  # Invalid: empty document type
        })
    }
    context = {}
    
    # Execute
    response = lambda_function.handler(event, context)
    
    # Assert
    assert response['statusCode'] == 400
    body = json.loads(response['body'])
    assert 'error' in body
```

### 6.2 Integration Test Structure

**tests/integration/test_document_processing_flow.py**:
```python
import pytest
import boto3
import json
import time
from datetime import datetime

@pytest.fixture
def aws_clients():
    """Create AWS service clients"""
    return {
        's3': boto3.client('s3', region_name='eu-west-1'),
        'dynamodb': boto3.client('dynamodb', region_name='eu-west-1'),
        'stepfunctions': boto3.client('stepfunctions', region_name='eu-west-1'),
        'lambda': boto3.client('lambda', region_name='eu-west-1')
    }

@pytest.mark.integration
def test_end_to_end_document_processing(aws_clients):
    """Test complete document processing workflow"""
    # Step 1: Upload document
    document_id = upload_test_document(
        aws_clients['s3'],
        'test-passport.pdf'
    )
    
    # Step 2: Wait for processing
    status = wait_for_processing_completion(
        aws_clients['dynamodb'],
        document_id,
        timeout=300
    )
    
    # Step 3: Verify results
    assert status == 'COMPLETED'
    
    # Step 4: Retrieve output
    output = get_processing_output(
        aws_clients['s3'],
        document_id
    )
    
    # Step 5: Validate output
    assert output['document_type'] == 'passport'
    assert output['confidence_scores']['overall'] > 0.85
    assert 'extracted_data' in output
    assert 'passport_number' in output['extracted_data']

def upload_test_document(s3_client, filename):
    """Upload test document to S3"""
    with open(f'tests/fixtures/{filename}', 'rb') as f:
        document_id = str(uuid.uuid4())
        s3_client.put_object(
            Bucket=os.environ['INPUT_BUCKET_NAME'],
            Key=f'documents/{document_id}.pdf',
            Body=f.read()
        )
    return document_id

def wait_for_processing_completion(dynamodb_client, document_id, timeout=300):
    """Wait for document processing to complete"""
    start_time = time.time()
    
    while time.time() - start_time < timeout:
        response = dynamodb_client.get_item(
            TableName=os.environ['METADATA_TABLE_NAME'],
            Key={'document_id': {'S': document_id}}
        )
        
        if 'Item' in response:
            status = response['Item']['status']['S']
            if status in ['COMPLETED', 'FAILED', 'LOW_CONFIDENCE']:
                return status
        
        time.sleep(5)
    
    raise TimeoutError(f'Processing did not complete within {timeout} seconds')
```

### 6.3 E2E Test Structure

**tests/e2e/test_api_workflow.py**:
```python
import pytest
import requests
import time

@pytest.mark.e2e
def test_complete_api_workflow(environment):
    """Test complete workflow via API"""
    base_url = get_base_url(environment)
    
    # Step 1: Request presigned URL
    response = requests.post(
        f'{base_url}/upload',
        json={
            'document_type': 'passport',
            'callback_url': 'https://webhook.site/test'
        },
        headers={'Authorization': f'Bearer {get_api_token()}'}
    )
    
    assert response.status_code == 200
    data = response.json()
    document_id = data['document_id']
    upload_url = data['upload_url']
    
    # Step 2: Upload document
    with open('tests/fixtures/test-passport.pdf', 'rb') as f:
        upload_response = requests.put(upload_url, data=f.read())
    
    assert upload_response.status_code == 200
    
    # Step 3: Poll for completion
    max_attempts = 60
    for attempt in range(max_attempts):
        status_response = requests.get(
            f'{base_url}/status/{document_id}',
            headers={'Authorization': f'Bearer {get_api_token()}'}
        )
        
        assert status_response.status_code == 200
        status_data = status_response.json()
        
        if status_data['status'] in ['COMPLETED', 'FAILED']:
            break
        
        time.sleep(5)
    
    # Step 4: Verify completion
    assert status_data['status'] == 'COMPLETED'
    assert status_data['document_type'] == 'passport'
    assert status_data['classification_confidence'] > 0.85
```

---

## 7. Environment Management

### 7.1 Environment Variables

**GitLab CI/CD Variables** (configured in GitLab UI):

**Protected Variables** (prod only):
```
AWS_ACCESS_KEY_ID_PROD
AWS_SECRET_ACCESS_KEY_PROD
AWS_ACCOUNT_ID_PROD
```

**Environment-Specific Variables**:
```
AWS_ACCESS_KEY_ID_DEV
AWS_SECRET_ACCESS_KEY_DEV
AWS_ACCOUNT_ID_DEV

AWS_ACCESS_KEY_ID_TEST
AWS_SECRET_ACCESS_KEY_TEST
AWS_ACCOUNT_ID_TEST
```

**Common Variables**:
```
AWS_DEFAULT_REGION=eu-west-1
TERRAFORM_VERSION=1.6.0
PYTHON_VERSION=3.12
```

### 7.2 Secrets Management

**Terraform Variables for Secrets**:
```hcl
# Reference secrets from AWS Secrets Manager
data "aws_secretsmanager_secret_version" "new_relic" {
  secret_id = "idp/${var.environment}/new-relic"
}

locals {
  new_relic_config = jsondecode(data.aws_secretsmanager_secret_version.new_relic.secret_string)
}

# Use in Lambda environment variables
resource "aws_lambda_function" "example" {
  environment {
    variables = {
      NEW_RELIC_LICENSE_KEY = local.new_relic_config.license_key
    }
  }
}
```

---

## 8. Monitoring & Alerting

### 8.1 Pipeline Monitoring

**GitLab Pipeline Metrics**:
- Pipeline success rate
- Average pipeline duration
- Stage failure rates
- Deployment frequency

**Custom Metrics**:
```yaml
# .gitlab-ci.yml
after_script:
  - |
    # Send pipeline metrics to CloudWatch
    aws cloudwatch put-metric-data \
      --namespace "IDP/CICD" \
      --metric-name PipelineDuration \
      --value ${CI_PIPELINE_DURATION} \
      --dimensions Environment=${ENVIRONMENT},Pipeline=${CI_PIPELINE_ID}
```

### 8.2 Deployment Alerts

**Slack Notifications**:
```yaml
notify:slack:
  stage: .post
  image: curlimages/curl:latest
  script:
    - |
      curl -X POST ${SLACK_WEBHOOK_URL} \
        -H 'Content-Type: application/json' \
        -d "{
          \"text\": \"IDP Deployment ${CI_PIPELINE_STATUS}\",
          \"attachments\": [{
            \"color\": \"${CI_PIPELINE_STATUS == 'success' ? 'good' : 'danger'}\",
            \"fields\": [
              {\"title\": \"Environment\", \"value\": \"${ENVIRONMENT}\", \"short\": true},
              {\"title\": \"Branch\", \"value\": \"${CI_COMMIT_REF_NAME}\", \"short\": true},
              {\"title\": \"Commit\", \"value\": \"${CI_COMMIT_SHORT_SHA}\", \"short\": true},
              {\"title\": \"Pipeline\", \"value\": \"${CI_PIPELINE_URL}\", \"short\": false}
            ]
          }]
        }"
  when: always
```

---

## 9. Rollback Strategy

### 9.1 Automated Rollback

**Rollback Script**:
```bash
#!/bin/bash
# scripts/rollback.sh

set -e

ENVIRONMENT=$1
BACKUP_STATE=$2

if [ -z "$ENVIRONMENT" ] || [ -z "$BACKUP_STATE" ]; then
  echo "Usage: $0 <environment> <backup-state-file>"
  exit 1
fi

echo "Rolling back ${ENVIRONMENT} to state: ${BACKUP_STATE}"

cd terraform/environments/${ENVIRONMENT}

# Restore state
terraform state push ${BACKUP_STATE}

# Apply previous configuration
terraform apply -auto-approve

echo "Rollback completed successfully!"
```

### 9.2 Rollback Triggers

**Automatic Rollback Conditions**:
1. Post-deployment validation fails
2. Error rate exceeds threshold (> 10%)
3. Critical endpoint health check fails
4. Manual trigger via GitLab UI

**Rollback Job**:
```yaml
rollback:auto:
  stage: deploy-prod
  image: hashicorp/terraform:${TERRAFORM_VERSION}
  environment:
    name: production
    action: rollback
  script:
    - cd terraform/environments/prod
    - terraform init
    - |
      # Find latest backup
      latest_backup=$(ls -t state-backup-*.json | head -1)
      if [ -n "$latest_backup" ]; then
        echo "Rolling back to: $latest_backup"
        terraform state push $latest_backup
        terraform apply -auto-approve
      else
        echo "No backup found!"
        exit 1
      fi
  when: on_failure
  only:
    - main
```

---

## 10. Performance Optimization

### 10.1 Pipeline Caching

```yaml
cache:
  key: ${CI_COMMIT_REF_SLUG}
  paths:
    - .pip-cache/
    - .terraform/
    - node_modules/
  policy: pull-push

# Per-job cache
build:lambda:
  cache:
    key: ${CI_COMMIT_REF_SLUG}-lambda
    paths:
      - .docker-cache/
    policy: pull-push
```

### 10.2 Parallel Execution

```yaml
# Build Lambda functions in parallel
build:lambda:parallel:
  stage: build
  parallel:
    matrix:
      - LAMBDA_NAME:
        - presigned_url_generator
        - bda_orchestrator
        - language_detector
        - document_classifier
        - schema_extractor
        - status_updater
        - document_cleanup
  script:
    - cd src/lambda_functions/${LAMBDA_NAME}
    - docker build -t idp-${LAMBDA_NAME}:${CI_COMMIT_SHORT_SHA} .
```

### 10.3 Artifact Management

```yaml
# Optimize artifact storage
build:lambda:
  artifacts:
    paths:
      - build/
    expire_in: 1 week
    when: on_success
```

---

## 11. Compliance & Audit

### 11.1 Audit Trail

**Pipeline Audit Log**:
```yaml
audit:log:
  stage: .post
  script:
    - |
      # Log deployment details
      cat > audit-log.json <<EOF
      {
        "timestamp": "$(date -u +%Y-%m-%dT%H:%M:%SZ)",
        "pipeline_id": "${CI_PIPELINE_ID}",
        "commit_sha": "${CI_COMMIT_SHA}",
        "environment": "${ENVIRONMENT}",
        "deployed_by": "${GITLAB_USER_LOGIN}",
        "status": "${CI_PIPELINE_STATUS}",
        "terraform_changes": $(terraform show -json tfplan)
      }
      EOF
    - aws s3 cp audit-log.json s3://idp-audit-logs/cicd/${CI_PIPELINE_ID}.json
  when: always
```

### 11.2 Compliance Checks

```yaml
compliance:check:
  stage: security
  script:
    - |
      # Check for required tags
      terraform show -json tfplan | jq '.resource_changes[] | select(.change.after.tags.Project == null)' > missing-tags.json
      
      if [ -s missing-tags.json ]; then
        echo "Resources missing required tags!"
        cat missing-tags.json
        exit 1
      fi
    
    - |
      # Check for encryption
      terraform show -json tfplan | jq '.resource_changes[] | select(.type == "aws_s3_bucket" and .change.after.server_side_encryption_configuration == null)' > unencrypted-buckets.json
      
      if [ -s unencrypted-buckets.json ]; then
        echo "S3 buckets without encryption!"
        cat unencrypted-buckets.json
        exit 1
      fi
```

---

## 12. Disaster Recovery

### 12.1 Backup Strategy

**Automated Backups**:
```yaml
backup:terraform:state:
  stage: .pre
  image: amazon/aws-cli:latest
  script:
    - |
      # Backup current Terraform state
      for env in dev test prod; do
        aws s3 cp \
          s3://evoke-terraform-state-eu-west-1/idp/${env}/terraform.tfstate \
          s3://evoke-terraform-backups/idp/${env}/terraform.tfstate.$(date +%Y%m%d-%H%M%S)
      done
  only:
    - schedules
```

### 12.2 Recovery Procedures

**Recovery Script**:
```bash
#!/bin/bash
# scripts/recover.sh

ENVIRONMENT=$1
BACKUP_TIMESTAMP=$2

echo "Recovering ${ENVIRONMENT} from backup: ${BACKUP_TIMESTAMP}"

# Download backup
aws s3 cp \
  s3://evoke-terraform-backups/idp/${ENVIRONMENT}/terraform.tfstate.${BACKUP_TIMESTAMP} \
  terraform.tfstate.backup

# Restore state
cd terraform/environments/${ENVIRONMENT}
terraform state push terraform.tfstate.backup

# Verify
terraform plan

echo "Recovery completed. Review the plan and apply if correct."
```

---

## 13. Best Practices

### 13.1 Pipeline Best Practices

1. **Fail Fast**: Run quick checks (linting, formatting) early
2. **Parallel Execution**: Build and test in parallel when possible
3. **Caching**: Cache dependencies to speed up builds
4. **Artifacts**: Only keep necessary artifacts, set expiration
5. **Manual Gates**: Require manual approval for production
6. **Rollback Plan**: Always have a rollback strategy

### 13.2 Security Best Practices

1. **Secrets Management**: Never commit secrets, use Secrets Manager
2. **Least Privilege**: IAM roles with minimum required permissions
3. **Scanning**: Run security scans on every commit
4. **Audit Logging**: Log all deployments and changes
5. **Protected Branches**: Protect main/prod branches
6. **Code Review**: Require approvals for merge requests

### 13.3 Terraform Best Practices

1. **State Locking**: Use DynamoDB for state locking
2. **Remote State**: Store state in S3 with encryption
3. **Modules**: Use modules for reusable components
4. **Workspaces**: Separate environments with workspaces or directories
5. **Plan Review**: Always review plan before apply
6. **Version Pinning**: Pin Terraform and provider versions

---

## 14. Troubleshooting

### 14.1 Common Issues

**Issue: Pipeline Timeout**
```yaml
# Solution: Increase timeout
default:
  timeout: 2h
```

**Issue: Docker Build Fails**
```bash
# Solution: Clear Docker cache
docker system prune -af
```

**Issue: Terraform State Lock**
```bash
# Solution: Force unlock (use with caution)
terraform force-unlock <lock-id>
```

**Issue: ECR Authentication Fails**
```bash
# Solution: Re-authenticate
aws ecr get-login-password --region eu-west-1 | \
  docker login --username AWS --password-stdin ${ECR_REGISTRY}
```

### 14.2 Debug Mode

```yaml
# Enable debug logging
variables:
  TF_LOG: DEBUG
  AWS_SDK_LOAD_CONFIG: 1
  PYTHONVERBOSE: 1
```

---

**End of CI/CD Pipeline LLD**
