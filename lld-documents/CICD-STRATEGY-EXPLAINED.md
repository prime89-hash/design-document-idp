# CI/CD Pipeline Strategy - Detailed Explanation

## Question: Single Pipeline or Separate Pipelines?

### Answer: **Single Unified Pipeline with Change Detection** ✅

---

## Why Single Unified Pipeline?

### 1. **Atomic Deployments**
All components (Infrastructure, Application, BDA configs) are deployed together as a single unit.

**Benefit**: Ensures compatibility between all layers.

**Example**:
```
Scenario: You add a new Lambda function for document validation

Single Pipeline:
✅ Terraform creates the Lambda function
✅ Docker image is built and pushed to ECR
✅ Lambda code is deployed
✅ BDA config is updated to use the new validation
✅ All deployed together - guaranteed to work

Separate Pipelines:
❌ Infra pipeline creates Lambda (deployed Monday)
❌ App pipeline builds image (deployed Tuesday)
❌ BDA config updated (deployed Wednesday)
❌ Risk: Mismatched versions, broken functionality
```

### 2. **Simplified Dependency Management**
The pipeline automatically handles dependencies between layers.

**Deployment Order**:
```
1. Infrastructure (Terraform)
   └─ Creates: S3, DynamoDB, SQS, Lambda shells, Step Functions, API Gateway
   
2. Application (Lambda Code)
   └─ Updates: Lambda function code with ECR images
   
3. BDA Configurations
   └─ Uploads: config.json, schema.json, prompt.txt to S3
```

**Automatic Dependencies**:
```yaml
deploy:application:
  needs:
    - deploy:infrastructure  # Waits for infra to complete

deploy:bda-configs:
  needs:
    - deploy:application     # Waits for app to complete
```

### 3. **Change Detection = Fast Pipelines**
Only runs stages for components that changed.

**Example Scenarios**:

| What Changed | Stages Executed | Duration |
|--------------|----------------|----------|
| Only Lambda code (src/) | Validate → Build → Test → Security → Package → Deploy App | ~15 min |
| Only Terraform (terraform/) | Validate → Test (plan) → Security → Deploy Infra | ~12 min |
| Only BDA configs (config/) | Validate → Deploy Configs | ~5 min |
| Everything | Full pipeline | ~30 min |
| Nothing (just docs) | Validate only | ~3 min |

**Implementation**:
```yaml
deploy:infrastructure:
  rules:
    - changes:
      - terraform/**/*      # Only run if Terraform files changed

deploy:application:
  rules:
    - changes:
      - src/**/*            # Only run if source code changed

deploy:bda-configs:
  rules:
    - changes:
      - config/**/*         # Only run if configs changed
```

### 4. **Easier Rollback**
Rollback the entire stack together to a known good state.

**Single Pipeline Rollback**:
```bash
# Rollback to previous commit
git revert HEAD
git push

# Pipeline automatically:
# 1. Reverts infrastructure
# 2. Reverts application code
# 3. Reverts BDA configs
# All in sync!
```

**Separate Pipelines Rollback**:
```bash
# Need to coordinate rollback across 3 repos
# 1. Rollback infra repo to commit ABC
# 2. Rollback app repo to commit XYZ
# 3. Rollback config repo to commit 123
# Risk: Forget one, or rollback to wrong versions
```

---

## Pipeline Flow Visualization

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SINGLE UNIFIED PIPELINE                              │
│                                                                              │
│  Git Push to develop/main                                                   │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ STAGE 1: VALIDATE (Parallel)                                         │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐                     │  │
│  │  │ Code       │  │ Terraform  │  │ BDA Config │                     │  │
│  │  │ Lint       │  │ Validate   │  │ Validate   │                     │  │
│  │  └────────────┘  └────────────┘  └────────────┘                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ STAGE 2: BUILD (Parallel)                                            │  │
│  │  - Build 9 Lambda Docker images                                      │  │
│  │  - Build Lambda layer                                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ STAGE 3: TEST                                                         │  │
│  │  - Unit tests (coverage >80%)                                        │  │
│  │  - Integration tests                                                  │  │
│  │  - Terraform plan                                                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ STAGE 4: SECURITY (Parallel)                                         │  │
│  │  ┌────────────┐  ┌────────────┐  ┌────────────┐                     │  │
│  │  │ SAST       │  │ Container  │  │ Secrets    │                     │  │
│  │  │ (Bandit)   │  │ Scan       │  │ Detection  │                     │  │
│  │  └────────────┘  └────────────┘  └────────────┘                     │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ STAGE 5: PACKAGE                                                      │  │
│  │  - Push Docker images to ECR                                         │  │
│  │  - Upload Lambda layer to S3                                         │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ STAGE 6: DEPLOY TO DEV (Manual Trigger)                              │  │
│  │                                                                        │  │
│  │  Step 1: Deploy Infrastructure (Terraform)                           │  │
│  │    ├─ S3 buckets, DynamoDB tables                                    │  │
│  │    ├─ SQS queues, DLQ                                                │  │
│  │    ├─ Lambda functions (shells)                                      │  │
│  │    ├─ Step Functions, EventBridge                                    │  │
│  │    ├─ API Gateway                                                     │  │
│  │    └─ IAM roles, CloudWatch alarms                                   │  │
│  │    Duration: ~5-10 minutes                                            │  │
│  │         │                                                              │  │
│  │         ▼                                                              │  │
│  │  Step 2: Deploy Application (Lambda Code)                            │  │
│  │    ├─ Update Lambda function code (ECR images)                       │  │
│  │    ├─ Update Lambda layers                                           │  │
│  │    └─ Publish new versions                                           │  │
│  │    Duration: ~2-3 minutes                                             │  │
│  │         │                                                              │  │
│  │         ▼                                                              │  │
│  │  Step 3: Deploy BDA Configurations                                   │  │
│  │    ├─ Upload config.json to S3                                       │  │
│  │    ├─ Upload schema.json to S3                                       │  │
│  │    ├─ Upload prompt.txt to S3                                        │  │
│  │    └─ Invalidate Lambda config cache                                 │  │
│  │    Duration: ~1 minute                                                │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  ┌──────────────────────────────────────────────────────────────────────┐  │
│  │ STAGE 7: VALIDATE DEV (Smoke Tests)                                  │  │
│  │  - Test API Gateway endpoints                                        │  │
│  │  - Upload test document                                              │  │
│  │  - Verify processing flow                                            │  │
│  │  Duration: ~5 minutes                                                 │  │
│  └──────────────────────────────────────────────────────────────────────┘  │
│         │                                                                    │
│         ▼                                                                    │
│  (Repeat for TEST and PROD environments)                                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## When Would You Use Separate Pipelines?

### Scenario: Large Enterprise with Specialized Teams

**Organization Structure**:
```
Infrastructure Team (10 people)
  └─ Manages: AWS resources, networking, security
  └─ Changes: Infrequent (weekly or monthly)
  └─ Approval: Requires architecture review board

Application Team (20 people)
  └─ Manages: Lambda functions, business logic
  └─ Changes: Frequent (multiple times per day)
  └─ Approval: Code review only

ML/AI Team (5 people)
  └─ Manages: BDA configurations, prompts, schemas
  └─ Changes: As needed (when adding document types)
  └─ Approval: ML lead approval
```

**Separate Pipelines Structure**:
```
Repository 1: idp-infrastructure
  Pipeline: infra-pipeline.yml
  Stages: validate → plan → deploy-infra
  Trigger: Changes to terraform/
  Frequency: Weekly
  Duration: ~15 minutes

Repository 2: idp-application
  Pipeline: app-pipeline.yml
  Stages: build → test → deploy-app
  Trigger: Changes to src/
  Frequency: Multiple times per day
  Duration: ~10 minutes

Repository 3: idp-bda-configs
  Pipeline: config-pipeline.yml
  Stages: validate → deploy-configs
  Trigger: Changes to config/
  Frequency: As needed
  Duration: ~5 minutes
```

**Challenges with Separate Pipelines**:
1. **Version Compatibility**: Need to track which app version works with which infra version
2. **Coordination**: Teams must coordinate deployments
3. **Rollback Complexity**: Need to rollback multiple repos
4. **Testing**: Integration testing across repos is difficult
5. **Documentation**: Need to maintain compatibility matrix

---

## Our Recommendation for IDP

### Use Single Unified Pipeline Because:

1. **Team Size**: Small to medium team (5-15 people)
2. **Tight Coupling**: Infrastructure, app, and configs are interdependent
3. **Deployment Frequency**: All components change together frequently
4. **Simplicity**: Easier to maintain and operate
5. **Atomic Deployments**: Guaranteed compatibility

### Change Detection Handles Performance:

**Example: Only Lambda Code Changed**
```yaml
# Pipeline automatically skips infrastructure and BDA config stages
# Only runs: Validate → Build → Test → Security → Package → Deploy App
# Duration: ~15 minutes instead of ~30 minutes
```

**Example: Only BDA Config Changed**
```yaml
# Pipeline automatically skips infrastructure and application stages
# Only runs: Validate → Deploy Configs
# Duration: ~5 minutes instead of ~30 minutes
```

---

## Migration Path (If Needed Later)

If your team grows and you need separate pipelines:

### Phase 1: Current State (Single Pipeline)
```
idp-solution/
├── terraform/
├── src/
├── config/
└── .gitlab-ci.yml (unified)
```

### Phase 2: Transition (Monorepo with Separate Pipelines)
```
idp-solution/
├── terraform/
│   └── .gitlab-ci.yml (infra pipeline)
├── src/
│   └── .gitlab-ci.yml (app pipeline)
├── config/
│   └── .gitlab-ci.yml (config pipeline)
└── .gitlab-ci.yml (orchestrator)
```

### Phase 3: Separate Repos (If Needed)
```
idp-infrastructure/
  └── .gitlab-ci.yml

idp-application/
  └── .gitlab-ci.yml

idp-bda-configs/
  └── .gitlab-ci.yml
```

**Note**: Most teams never need Phase 3. Phase 1 (current design) works well for most use cases.

---

## Summary

| Aspect | Single Pipeline | Separate Pipelines |
|--------|----------------|-------------------|
| **Complexity** | Low | High |
| **Deployment Speed** | Medium (with change detection) | Fast (individual) |
| **Coordination** | Not needed | Required |
| **Rollback** | Easy | Complex |
| **Version Management** | Automatic | Manual |
| **Team Size** | Small to Medium | Large |
| **Best For** | Tightly coupled systems | Independent components |
| **Our Choice** | ✅ YES | ❌ NO |

---

## Conclusion

**We use a single unified pipeline with change detection** because:
- ✅ Simpler to maintain
- ✅ Atomic deployments
- ✅ Automatic dependency management
- ✅ Easier rollback
- ✅ Suitable for our team size
- ✅ Change detection keeps it fast

**The pipeline is smart enough to only deploy what changed**, giving us the best of both worlds: simplicity of a single pipeline with the speed of separate pipelines.

---

**Document Status**: Complete  
**Last Updated**: January 14, 2026  
**Reviewed By**: IDP Team
