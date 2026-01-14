# Intelligent Document Processing (IDP) - Low Level Design Documentation

## Overview

This directory contains comprehensive Low Level Design (LLD) documentation for the Intelligent Document Processing (IDP) solution built on AWS using Bedrock Data Automation (BDA), Terraform, and GitLab CI/CD.

### 🎯 **Start Here**: [FINAL-ARCHITECTURE-SUMMARY.md](FINAL-ARCHITECTURE-SUMMARY.md)
**Complete overview of the recommended simplified architecture**
- 5 Lambda functions (not 9)
- ~25 AWS resources (not 42)
- EventBridge → Step Functions (SQS optional)
- 40% simpler, same reliability

---

## Document Structure

### 0. [Architecture Diagrams](00-Architecture-Diagrams.md) 🎨
**Purpose**: Visual representation of the complete IDP architecture

**Contents**:
- High-level end-to-end architecture flow
- VPC and networking architecture
- Data flow diagram
- Component interaction diagram
- Error handling and retry flow
- SQS queue and DLQ integration

**Key Diagrams**:
- Complete system architecture with all components
- VPC layout with private/public subnets
- Message flow through SQS to Step Functions
- Error handling with DLQ and retry logic

**Also See**: [COMPLETE-INFRA-DIAGRAM.md](COMPLETE-INFRA-DIAGRAM.md) for detailed infrastructure with VPC, subnets, ENIs, and all AWS resources

---

### Quick References 📚

- **[SQS-DECISION-GUIDE.md](SQS-DECISION-GUIDE.md)**: ⚠️ **START HERE** - Should you use SQS? Decision guide and implementation options
- **[BDA-ONLY-ARCHITECTURE.md](BDA-ONLY-ARCHITECTURE.md)**: ⚠️ **IMPORTANT** - BDA handles ALL processing (language, classification, extraction)
- **[QUICK-REFERENCE.md](QUICK-REFERENCE.md)**: One-page cheat sheet with all key information
- **[CICD-STRATEGY-EXPLAINED.md](CICD-STRATEGY-EXPLAINED.md)**: Detailed explanation of single vs separate pipelines
- **[UPDATES-SUMMARY.md](UPDATES-SUMMARY.md)**: Summary of SQS integration and architecture updates

---

### 1. [Infrastructure LLD](01-Infrastructure-LLD.md) 🏗️
**Purpose**: Detailed infrastructure design and AWS resource specifications

**Contents**:
- Network architecture (VPC, subnets, security groups)
- Storage layer (S3 buckets, DynamoDB tables)
- Messaging layer (SQS queues, Dead Letter Queue)
- Compute layer (Lambda functions, configurations)
- Orchestration (Step Functions, EventBridge)
- Integration layer (API Gateway)
- Security & IAM (roles, policies, secrets)
- Monitoring & observability (CloudWatch, X-Ray, New Relic)
- Disaster recovery & backup strategies
- Cost optimization approaches
- Terraform module structure
- Scaling considerations

**Key Sections**:
- Section 2: Network Architecture
- Section 3: Storage Layer (S3 & DynamoDB)
- Section 4: Compute Layer (9 Lambda Functions including SQS Poller and DLQ Analyzer)
- Section 5: Orchestration Layer (SQS, DLQ, Step Functions)
- Section 7: Security & IAM
- Section 8: Monitoring & Observability (including SQS metrics and DLQ alarms)

---

### 2. [Application LLD](02-Application-LLD.md)
**Purpose**: Application architecture, Lambda function logic, and data flows

**Contents**:
- Application architecture overview
- Lambda function detailed specifications
- Step Functions workflow definition
- Configuration management
- Data models and schemas
- Error handling & logging strategies
- Testing strategy (unit, integration, E2E)
- Performance optimization
- Security implementation
- Application deployment structure

**Key Sections**:
- Section 2: Lambda Functions Detail (all 7 functions)
- Section 3: Step Functions Workflow
- Section 4: Configuration Management
- Section 5: Data Models
- Section 6: Error Handling & Logging
- Section 7: Testing Strategy

---

### 3. [BDA (Bedrock Data Automation) LLD](03-BDA-LLD.md)
**Purpose**: Bedrock Data Automation integration, blueprints, and AI/ML processing

**Contents**:
- BDA overview and architecture
- BDA project structure
- Blueprint design (classification & extraction)
- Document type blueprints (passports, bank statements, utility bills, legal documents)
- BDA API integration
- Blueprint management & versioning
- Model selection strategy
- Confidence scoring algorithms
- Performance optimization
- Multi-language support
- Quality assurance & validation
- Cost optimization
- Monitoring & troubleshooting
- Testing strategy

**Key Sections**:
- Section 3: Blueprint Design
- Section 4: BDA API Integration
- Section 6: Model Selection
- Section 7: Confidence Scoring
- Section 10: Quality Assurance

---

### 4. [CI/CD Pipeline LLD](04-CICD-Pipeline-LLD.md)
**Purpose**: Complete CI/CD pipeline design using GitLab and Terraform

**Contents**:
- CI/CD architecture overview
- Repository structure
- GitLab CI/CD pipeline configuration
- Build stage (code validation, Docker builds)
- Test stage (unit, integration, E2E tests)
- Security stage (SAST, container scanning, secrets detection)
- Package stage (ECR push)
- Deployment stage (dev, test, prod)
- Validation stage (smoke tests, health checks)
- Terraform deployment procedures
- Docker image build process
- Testing framework
- Environment management
- Monitoring & alerting
- Rollback strategy
- Performance optimization
- Compliance & audit
- Disaster recovery

**Key Sections**:
- Section 3: GitLab CI/CD Pipeline
- Section 4: Terraform Deployment
- Section 5: Docker Image Build
- Section 6: Testing Framework
- Section 9: Rollback Strategy
- Section 11: Compliance & Audit

---

## Quick Reference

### Architecture Components

| Component | Technology | Purpose |
|-----------|-----------|---------|
| **Compute** | AWS Lambda (Docker) | Serverless processing (5 functions) |
| **AI/ML** | Bedrock Data Automation | Document classification & extraction |
| **Orchestration** | Step Functions | Workflow management |
| **Messaging** | EventBridge (+ optional SQS) | Event routing (+ optional buffering) |
| **Storage** | S3, DynamoDB | Document & metadata storage |
| **Integration** | API Gateway | API endpoints |
| **IaC** | Terraform | Infrastructure provisioning |
| **CI/CD** | GitLab CI/CD | Automated deployment |
| **Monitoring** | CloudWatch, X-Ray, New Relic | Observability |
| **Container Registry** | Amazon ECR | Docker image storage |

### Document Types Supported

**Verification Documents**:
- Identity: National ID, Driver's License, Passport, Residence Permit, Citizenship Card
- Proof of Address: Utility bills, Bank statements, Lease agreements, Council tax bills
- Source of Funds: Payslips, Bank statements, Tax declarations
- Photo Holding ID: Identity verification with photo

**Legal Claims Documents**:
- UK: Claim forms, Judgements, Orders, Notices
- Malta: Cedola, Enforcement documents
- Sweden: Enforcement documents
- Spain: Court claims, Judgements, Applications
- Germany: Claim applications, Cost orders, Judgements, EPO documents

### Environments

| Environment | Purpose | Deployment Trigger | Approval Required |
|-------------|---------|-------------------|-------------------|
| **Dev** | Development testing | Push to `develop` | Manual |
| **Test** | Integration & E2E testing | After Dev validation | Manual |
| **Prod** | Production | Push to `main` | Manual |

### Key Metrics & Targets

| Metric | Target | Monitoring |
|--------|--------|------------|
| Document Processing Throughput | 100 docs/min | CloudWatch |
| API Response Time | < 500ms | CloudWatch |
| End-to-End Processing | < 2 min (avg) | CloudWatch |
| Classification Accuracy | > 95% | Custom metrics |
| Extraction Accuracy | > 90% | Custom metrics |
| Error Rate | < 5% | CloudWatch Alarms |
| Availability | 99.9% | CloudWatch |

---

## Getting Started

### Prerequisites

1. **AWS Account**: Access to Evoke AWS account with appropriate permissions
2. **VPC & Networking**: Existing VPC, subnets, NAT Gateway (provided by Evoke)
3. **GitLab Access**: Repository access and CI/CD runner configuration
4. **Terraform**: Version 1.6.0 or higher
5. **AWS CLI**: Configured with appropriate credentials
6. **Docker**: For local testing and image builds
7. **Python**: Version 3.12 for Lambda functions

### Initial Setup

1. **Clone Repository**:
   ```bash
   git clone <repository-url>
   cd idp-solution
   ```

2. **Configure AWS Credentials**:
   ```bash
   aws configure --profile idp-dev
   ```

3. **Initialize Terraform**:
   ```bash
   cd terraform/environments/dev
   terraform init
   ```

4. **Review Configuration**:
   - Update `terraform.tfvars` with Evoke-provided values
   - Review VPC and subnet IDs
   - Configure BDA project ARN

5. **Deploy Infrastructure**:
   ```bash
   terraform plan
   terraform apply
   ```

6. **Configure GitLab CI/CD**:
   - Set environment variables in GitLab UI
   - Configure AWS credentials
   - Enable GitLab runners

### Deployment Workflow

```
1. Feature Development
   ├── Create feature branch: feature/document-type-xyz
   ├── Develop & test locally
   ├── Push to GitLab
   └── Create Merge Request

2. CI/CD Pipeline (Automatic)
   ├── Code validation (linting, formatting)
   ├── Unit tests
   ├── Security scans
   ├── Build Docker images
   └── Push to ECR

3. Deploy to Dev (Manual Trigger)
   ├── Terraform apply to dev environment
   ├── Smoke tests
   └── Validation

4. Deploy to Test (Manual Trigger)
   ├── Terraform apply to test environment
   ├── Integration tests
   ├── E2E tests
   └── Validation

5. Deploy to Production (Manual Trigger)
   ├── Merge to main branch
   ├── Terraform apply to prod environment
   ├── Health checks
   └── Monitoring
```

### Processing Workflow

```
1. Client uploads document
   ↓
2. S3 event → EventBridge → SQS Queue
   ↓
3. SQS Poller Lambda → Step Functions
   ↓
4. Language Detection → Classification → Extraction
   ↓
5. Output saved to S3 + DynamoDB updated
   ↓
6. Client callback (optional)

Error Handling:
- Step Functions retries (3 attempts with exponential backoff)
- SQS retries (3 attempts)
- Failed messages → DLQ → Alert
```

---

## Configuration Management

### Document Type Configuration

Each document type requires three configuration files in S3:

**Location**: `s3://idp-config-{env}/configs/{document-type}/`

**Files**:
1. `config.json` - Processing configuration
2. `schema.json` - Extraction schema (JSON Schema format)
3. `prompt.txt` - BDA extraction prompt

**Example**:
```
s3://idp-config-dev/configs/
├── passport/
│   ├── config.json
│   ├── schema.json
│   └── prompt.txt
├── bank-statement/
│   ├── config.json
│   ├── schema.json
│   └── prompt.txt
└── utility-bill/
    ├── config.json
    ├── schema.json
    └── prompt.txt
```

### Environment Variables

**Lambda Functions** (set via Terraform):
- `INPUT_BUCKET_NAME`: S3 bucket for input documents
- `OUTPUT_BUCKET_NAME`: S3 bucket for output JSON
- `CONFIG_BUCKET_NAME`: S3 bucket for configurations
- `METADATA_TABLE_NAME`: DynamoDB table for metadata
- `BDA_PROJECT_ARN`: Bedrock Data Automation project ARN
- `LOG_LEVEL`: Logging level (INFO, DEBUG, ERROR)

**GitLab CI/CD** (set in GitLab UI):
- `AWS_ACCESS_KEY_ID_{ENV}`: AWS access key per environment
- `AWS_SECRET_ACCESS_KEY_{ENV}`: AWS secret key per environment
- `AWS_ACCOUNT_ID_{ENV}`: AWS account ID per environment
- `AWS_DEFAULT_REGION`: AWS region (eu-west-1)

---

## Testing

### Local Testing

**Unit Tests**:
```bash
# Install dependencies
pip install -r requirements-dev.txt

# Run unit tests
pytest tests/unit/ --cov=src --cov-report=html

# View coverage report
open htmlcov/index.html
```

**Integration Tests** (requires AWS credentials):
```bash
# Run integration tests
pytest tests/integration/ -v

# Run specific test
pytest tests/integration/test_document_processing_flow.py -v
```

**Docker Build Test**:
```bash
# Build Lambda image locally
cd src/lambda_functions/bda_orchestrator
docker build -t idp-bda-orchestrator:test .

# Test Lambda locally
docker run -p 9000:8080 idp-bda-orchestrator:test
curl -XPOST "http://localhost:9000/2015-03-31/functions/function/invocations" -d '{}'
```

### CI/CD Testing

Tests run automatically in GitLab pipeline:
- **Unit Tests**: On every commit
- **Integration Tests**: On develop/main branches
- **E2E Tests**: After deployment to test environment
- **Smoke Tests**: After deployment to any environment

---

## Monitoring & Troubleshooting

### CloudWatch Dashboards

**IDP Overview Dashboard**:
- Document processing rate
- Success/failure rates
- Average processing time
- Error distribution by type

**Lambda Performance Dashboard**:
- Invocation count
- Duration
- Error rate
- Throttles
- Concurrent executions

**BDA Performance Dashboard**:
- Job submission rate
- Job completion time
- Confidence scores
- Model usage

### Common Issues

**Issue: Low Confidence Scores**
- **Check**: Document quality, image resolution
- **Solution**: Implement pre-processing, adjust confidence thresholds
- **Logs**: Check Lambda logs for BDA response details

**Issue: Processing Timeout**
- **Check**: Lambda timeout settings, document size, SQS visibility timeout
- **Solution**: Increase timeout, optimize code, split large documents
- **Logs**: CloudWatch Logs for timeout errors, SQS message age

**Issue: Messages in DLQ**
- **Check**: DLQ CloudWatch metrics, DLQ Analyzer Lambda logs
- **Solution**: Review error details, fix root cause, replay messages from DLQ
- **Logs**: DLQ Analyzer logs, Step Functions execution history

**Issue: High SQS Queue Depth**
- **Check**: SQS metrics, Lambda concurrency, Step Functions execution rate
- **Solution**: Increase Lambda reserved concurrency, check for bottlenecks
- **Logs**: CloudWatch metrics for queue depth and message age

**Issue: Deployment Failure**
- **Check**: Terraform plan output, IAM permissions
- **Solution**: Review error messages, verify prerequisites
- **Logs**: GitLab CI/CD pipeline logs, Terraform logs

**Issue: High Error Rate**
- **Check**: CloudWatch metrics, error logs, DLQ message count
- **Solution**: Review recent changes, check BDA service status
- **Logs**: Lambda logs, Step Functions execution history, DLQ messages

### Log Locations

- **Lambda Logs**: `/aws/lambda/idp-{function-name}-{env}`
- **Step Functions**: `/aws/stepfunctions/idp-document-processing-{env}`
- **API Gateway**: `/aws/apigateway/idp-api-{env}`
- **SQS Metrics**: CloudWatch Metrics → AWS/SQS namespace
- **DLQ Analyzer**: `/aws/lambda/idp-dlq-analyzer-{env}`
- **Terraform**: GitLab CI/CD pipeline artifacts
- **Audit Logs**: `s3://idp-audit-logs-{env}/`

---

## Security Considerations

### Data Protection

1. **Encryption at Rest**: All S3 buckets and DynamoDB tables encrypted
2. **Encryption in Transit**: HTTPS/TLS for all communications
3. **Data Retention**: Automatic deletion based on lifecycle policies
4. **Access Control**: IAM roles with least privilege
5. **Audit Trail**: All API calls logged to CloudTrail

### Compliance

- **GDPR**: Data residency in EU, retention policies enforced
- **Data Minimization**: Only required data stored
- **Right to Erasure**: Automated cleanup mechanisms
- **Audit Logging**: Complete audit trail maintained

### Secrets Management

- **AWS Secrets Manager**: All secrets stored securely
- **No Hardcoded Secrets**: Secrets injected at runtime
- **Rotation**: Manual rotation for API keys
- **Access Control**: Secrets accessible only by authorized services

---

## Cost Optimization

### Cost Breakdown (Estimated Monthly)

| Service | Usage | Estimated Cost |
|---------|-------|----------------|
| Lambda | 1M invocations, 1GB-sec | $20 |
| BDA | 10K documents, 50K pages | $150 |
| S3 | 100GB storage, 1M requests | $25 |
| DynamoDB | On-demand, 1M reads/writes | $30 |
| SQS | 1M requests | $0.40 |
| Step Functions | 10K executions | $25 |
| CloudWatch | Logs, metrics, alarms | $20 |
| **Total** | | **~$270/month** |

### Optimization Strategies

1. **Lambda**: ARM64 architecture (20% savings), right-sized memory
2. **BDA**: Use Haiku model for simple documents (80% cost reduction)
3. **S3**: Lifecycle policies for automatic tiering
4. **DynamoDB**: On-demand pricing for variable workloads
5. **VPC Endpoints**: Reduce NAT Gateway data transfer costs

---

## Support & Contacts

### Team Contacts

- **IDP Team Lead**: [Name] - [email]
- **DevOps Lead**: [Name] - [email]
- **AWS Architect**: [Name] - [email]
- **Security Lead**: [Name] - [email]

### Escalation Path

1. **Level 1**: IDP Development Team
2. **Level 2**: DevOps/Platform Team
3. **Level 3**: AWS Solutions Architect (Evoke)
4. **Level 4**: AWS Support (Enterprise)

### Resources

- **GitLab Repository**: [URL]
- **Confluence Documentation**: [URL]
- **Jira Board**: [URL]
- **Slack Channel**: #idp-team
- **AWS Console**: [Account URL]

---

## Change Log

| Version | Date | Author | Changes |
|---------|------|--------|---------|
| 1.0 | 2026-01-14 | IDP Team | Initial LLD documentation |

---

## Appendix

### Glossary

- **BDA**: Bedrock Data Automation - AWS managed service for document processing
- **IDP**: Intelligent Document Processing
- **VIZ**: Visual Inspection Zone (passport)
- **MRZ**: Machine Readable Zone (passport)
- **POA**: Proof of Address
- **SOF**: Source of Funds
- **HITL**: Human In The Loop
- **E2E**: End-to-End
- **SAST**: Static Application Security Testing

### References

- [AWS Bedrock Documentation](https://docs.aws.amazon.com/bedrock/)
- [Terraform AWS Provider](https://registry.terraform.io/providers/hashicorp/aws/latest/docs)
- [GitLab CI/CD Documentation](https://docs.gitlab.com/ee/ci/)
- [AWS Lambda Best Practices](https://docs.aws.amazon.com/lambda/latest/dg/best-practices.html)
- [Step Functions Best Practices](https://docs.aws.amazon.com/step-functions/latest/dg/best-practices.html)

---

**Document Status**: Draft  
**Last Updated**: January 14, 2026  
**Next Review**: February 14, 2026
