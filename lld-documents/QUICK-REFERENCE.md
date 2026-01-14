# IDP Solution - Quick Reference Guide

## 1. Architecture at a Glance

### Recommended (Start Simple):
```
Client → API Gateway → S3 → EventBridge → Step Functions → BDA → Output
```

### Optional (Add if needed):
```
Client → API Gateway → S3 → EventBridge → SQS → Step Functions → BDA → Output
                                            ↓
                                           DLQ → Analyzer → Alerts
```

**Recommendation**: Start without SQS, add only if traffic patterns require it.

## 2. Components Count

### Recommended Architecture (No SQS):

| Category | Count | Components |
|----------|-------|------------|
| **Lambda Functions** | 5 | Presigned URL Gen, BDA Orchestrator, Output Processor, Status Updater, Cleanup |
| **Storage** | 4 | Input S3, Output S3, Config S3, Audit S3 |
| **Database** | 2 | Metadata DynamoDB, Config Cache DynamoDB |
| **Orchestration** | 1 | Step Functions State Machine |
| **API** | 1 | API Gateway (2 endpoints) |
| **Events** | 1 | EventBridge Rule |

**Total AWS Resources**: ~25 resources per environment

### Optional (With SQS):

Add if needed:
- +1 SQS Queue
- +1 Dead Letter Queue
- +1 DLQ Analyzer Lambda (optional)

**Total with SQS**: ~28 resources per environment

## 3. Processing Flow (4 Phases)

### Recommended (Without SQS):
```
1. UPLOAD
   Client → API Gateway → Lambda → DynamoDB → S3

2. TRIGGER
   S3 → EventBridge → Step Functions (direct)

3. PROCESS
   Step Functions → BDA Orchestrator (language + classification + extraction)
                 → Output Processor (validate & save)
                 → Status Updater (update status & callback)

4. ERROR
   Failure → Step Functions Retry (3x: 2s, 4s, 8s)
          → If all fail: Mark as FAILED → CloudWatch Alarm → Alert
```

### Optional (With SQS - if traffic spikes occur):
```
1. UPLOAD
   Client → API Gateway → Lambda → DynamoDB → S3

2. TRIGGER
   S3 → EventBridge → SQS Queue

3. PROCESS
   SQS → Step Functions (native integration)
      → BDA Orchestrator → Output Processor → Status Updater

4. ERROR
   Failure → Step Functions Retry (3x)
          → SQS Retry (3x after visibility timeout)
          → DLQ (after 3 SQS retries) → Analyzer → Alert
```

## 4. CI/CD Pipeline

### Strategy: **Single Unified Pipeline with Change Detection**

```
Validate → Build → Test → Security → Package → Deploy (Infra → App → Config) → Validate
```

### Deployment Order:
```
1. Infrastructure (Terraform)    ~5-10 min
2. Application (Lambda Code)     ~2-3 min
3. BDA Configurations            ~1 min
```

### Change Detection:
| Changed | Stages Run | Duration |
|---------|-----------|----------|
| Lambda code only | Build → Test → Deploy App | ~15 min |
| Terraform only | Validate → Deploy Infra | ~12 min |
| BDA configs only | Validate → Deploy Config | ~5 min |
| Everything | Full pipeline | ~30 min |

## 5. Key Configurations

### Lambda Functions
```
Memory: 256 MB - 2048 MB
Timeout: 30s - 300s
Architecture: ARM64 (Graviton2)
Runtime: Python 3.12 (Docker)
VPC: Private subnets (for BDA access)
```

### SQS Queue
```
Visibility Timeout: 900s (15 min)
Message Retention: 14 days
Max Receive Count: 3
Batch Size: 10 messages
Long Polling: 20 seconds
```

### Step Functions
```
Type: STANDARD
Max Duration: 15 minutes
Retry: 3 attempts (2s, 4s, 8s backoff)
Error Handling: Return to SQS
```

### BDA
```
Models: Claude Sonnet (default), Haiku (fast), Opus (accurate)
Confidence Threshold: 0.85 (classification), 0.80 (extraction)
Blueprints: Per document type
```

## 6. Monitoring & Alerts

### Critical Alarms
- ✅ DLQ Message Count > 0 (CRITICAL)
- ✅ Lambda Error Rate > 5%
- ✅ Step Functions Failed > 10
- ✅ API Gateway 5xx > 50

### Key Metrics
- Documents Processed (Count)
- Processing Duration (ms)
- Classification Confidence (%)
- Extraction Confidence (%)
- SQS Queue Depth (Count)
- DLQ Message Count (Count)

### Log Locations
```
/aws/lambda/idp-{function-name}-{env}
/aws/stepfunctions/idp-document-processing-{env}
/aws/apigateway/idp-api-{env}
s3://idp-audit-logs-{env}/
```

## 7. Document Types Supported

### Verification Documents (14 types)
- **ID**: Passport, Driver's License, National ID, Residence Permit, Citizenship Card
- **POA**: Utility bills, Bank statements, Lease agreements, Tax bills, Payslips
- **SOF**: Payslips, Bank statements, Tax declarations
- **PH/ID**: Photo holding ID

### Legal Claims (15+ types)
- **UK**: Claim forms, Judgements, Orders, Notices
- **Malta**: Cedola, Enforcement documents
- **Sweden**: Enforcement documents
- **Spain**: Court claims, Judgements
- **Germany**: Claim applications, Cost orders, EPO documents

## 8. Environments

| Environment | Purpose | Trigger | Approval |
|-------------|---------|---------|----------|
| **Dev** | Development | Push to `develop` | Manual |
| **Test** | Integration testing | After Dev success | Manual |
| **Prod** | Production | Push to `main` | Manual |

## 9. Cost Estimate (Monthly)

| Service | Usage | Cost |
|---------|-------|------|
| Lambda | 1M invocations | $20 |
| BDA | 10K docs, 50K pages | $150 |
| S3 | 100GB, 1M requests | $25 |
| DynamoDB | 1M reads/writes | $30 |
| SQS | 1M requests | $0.40 |
| Step Functions | 10K executions | $25 |
| CloudWatch | Logs, metrics | $20 |
| **Total** | | **~$270** |

## 10. Performance Targets

```
Throughput: 100 documents/minute
API Response: < 500ms
Processing Time: < 2 minutes (average)
Classification Accuracy: > 95%
Extraction Accuracy: > 90%
Availability: 99.9%
```

## 11. Error Handling

### Retry Strategy
```
Step Functions: 3 retries (exponential backoff)
    ↓ (if all fail)
SQS: 3 retries (visibility timeout)
    ↓ (if all fail)
DLQ: Manual review + alert
```

### Total Retries
```
Maximum: 9 attempts (3 Step Functions × 3 SQS)
Duration: Up to 45 minutes (worst case)
```

## 12. Security

### Data Protection
- ✅ Encryption at rest (S3, DynamoDB)
- ✅ Encryption in transit (HTTPS/TLS)
- ✅ VPC for Lambda functions
- ✅ IAM least privilege
- ✅ Secrets Manager for credentials

### Compliance
- ✅ GDPR compliant (EU data residency)
- ✅ Data retention policies
- ✅ Audit trail (CloudTrail)
- ✅ Access logging

## 13. Deployment Checklist

### Pre-Deployment
- [ ] VPC and subnets provided by Evoke
- [ ] AWS credentials configured
- [ ] Terraform backend configured
- [ ] GitLab CI/CD variables set
- [ ] BDA project created
- [ ] Synthetic test data prepared

### Post-Deployment
- [ ] Smoke tests passed
- [ ] API Gateway endpoints accessible
- [ ] Test document processed successfully
- [ ] CloudWatch alarms configured
- [ ] New Relic integration working
- [ ] DLQ alerts tested

## 14. Common Commands

### Terraform
```bash
# Initialize
terraform init

# Plan
terraform plan -out=tfplan

# Apply
terraform apply tfplan

# Destroy
terraform destroy
```

### Docker
```bash
# Build Lambda image
docker build -t idp-lambda:latest .

# Push to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin ${ECR_REGISTRY}
docker push ${ECR_REGISTRY}/idp-lambda:latest
```

### AWS CLI
```bash
# Update Lambda code
aws lambda update-function-code \
  --function-name idp-lambda-dev \
  --image-uri ${ECR_REGISTRY}/idp-lambda:latest

# Check SQS queue depth
aws sqs get-queue-attributes \
  --queue-url ${QUEUE_URL} \
  --attribute-names ApproximateNumberOfMessages

# Check DLQ messages
aws sqs get-queue-attributes \
  --queue-url ${DLQ_URL} \
  --attribute-names ApproximateNumberOfMessages
```

### Testing
```bash
# Unit tests
pytest tests/unit/ --cov=src

# Integration tests
pytest tests/integration/ -v

# E2E tests
pytest tests/e2e/ --environment=dev
```

## 15. Troubleshooting Quick Guide

| Issue | Check | Solution |
|-------|-------|----------|
| **High error rate** | CloudWatch Logs, DLQ | Review logs, check BDA status |
| **Slow processing** | SQS queue depth, Lambda duration | Increase concurrency, optimize code |
| **DLQ messages** | DLQ Analyzer logs | Review error details, fix root cause |
| **Deployment failure** | Terraform logs, IAM permissions | Review error, verify prerequisites |
| **Low confidence** | BDA blueprints, document quality | Refine prompts, check document quality |

## 16. Key Files

```
.gitlab-ci.yml                    # Main CI/CD pipeline
terraform/environments/dev/       # Dev environment config
src/lambda_functions/             # Lambda function code
config/document-types/            # BDA configurations
lld-documents/                    # This documentation
```

## 17. Important URLs

```
API Gateway: https://idp-{env}.evoke.com
AWS Console: https://console.aws.amazon.com
GitLab Repo: [Your GitLab URL]
New Relic: [Your New Relic URL]
```

## 18. Team Contacts

```
IDP Team Lead: [Name] - [Email]
DevOps Lead: [Name] - [Email]
AWS Architect: [Name] - [Email]
Slack Channel: #idp-team
```

---

**Quick Reference Version**: 1.0  
**Last Updated**: January 14, 2026  
**For detailed information, see the full LLD documents**
