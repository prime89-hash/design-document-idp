# Infrastructure Low Level Design (LLD)
## Intelligent Document Processing (IDP) Solution

### Document Control
- **Version**: 1.0
- **Last Updated**: January 14, 2026
- **Status**: Draft

---

## Complete Infrastructure Architecture Diagram

```
╔═════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╗
║                                                    AWS CLOUD - EU REGION (eu-west-1)                                                                 ║
║                                                    IDP SOLUTION - COMPLETE INFRASTRUCTURE                                                             ║
╚═════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════════╝

┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                          EXTERNAL ZONE                                                                                │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                                                       │
│  ┌──────────────────────────────────┐                                    ┌──────────────────────────────────┐                                       │
│  │   ON-PREMISES                    │                                    │   AWS-HOSTED                     │                                       │
│  │   CLIENT APPLICATIONS            │                                    │   CLIENT APPLICATIONS            │                                       │
│  │                                  │                                    │                                  │                                       │
│  │  • UiPath Workflows              │                                    │  • EC2 Applications              │                                       │
│  │  • Internal Systems              │                                    │  • ECS Services                  │                                       │
│  │  • Legacy Applications           │                                    │  • Lambda Functions              │                                       │
│  └────────────┬─────────────────────┘                                    └────────────┬─────────────────────┘                                       │
│               │                                                                       │                                                               │
│               │ Direct Connect / VPN                                                  │ AWS PrivateLink                                               │
│               │                                                                       │                                                               │
└───────────────┼───────────────────────────────────────────────────────────────────────┼───────────────────────────────────────────────────────────────┘
                │                                                                       │
                │                                    HTTPS                              │
                └───────────────────────────────────────┬───────────────────────────────┘
                                                        │
                                                        ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                          API LAYER                                                                                    │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                                                       │
│  ┌───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐  │
│  │                                        AMAZON API GATEWAY (Regional)                                                                           │  │
│  │                                                                                                                                                 │  │
│  │  Endpoints:                                                                                                                                     │  │
│  │  • POST /upload        → Generate pre-signed S3 URL                                                                                            │  │
│  │  • GET  /status/{id}   → Get document processing status                                                                                        │  │
│  │                                                                                                                                                 │  │
│  │  Configuration:                                                                                                                                 │  │
│  │  • Type: REST API                                                                                                                               │  │
│  │  • Authorization: AWS_IAM                                                                                                                       │  │
│  │  • Throttling: 1000 req/s, Burst: 2000                                                                                                         │  │
│  │  • Stage: dev/test/prod                                                                                                                         │  │
│  └─────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘  │
│                                                                          │                                                                             │
└──────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┘
                                                                           │
                                                                           │ Invoke Lambda
                                                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                    COMPUTE LAYER (Outside VPC)                                                                        │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  Lambda: Presigned URL Generator                                                                                                             │   │
│  │  • Runtime: Python 3.12 (Docker)                                                                                                              │   │
│  │  • Memory: 512 MB                                                                                                                              │   │
│  │  • Timeout: 30s                                                                                                                                │   │
│  │  • VPC: No (public API Gateway integration)                                                                                                   │   │
│  │  • ECR Image: idp-presigned-url-generator:latest                                                                                              │   │
│  └─────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘   │
│                                                                          │                                                                             │
│                                                                          │ Write metadata                                                              │
│                                                                          ▼                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  DynamoDB: Metadata Table                                                                                                                     │   │
│  │  • Table: idp-document-metadata-{env}                                                                                                         │   │
│  │  • PK: document_id                                                                                                                             │   │
│  │  • Billing: On-Demand                                                                                                                          │   │
│  │  • Encryption: AWS Managed                                                                                                                     │   │
│  │  • PITR: Enabled                                                                                                                               │   │
│  └─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘   │
│                                                                                                                                                       │
└───────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┘
                                                                           │
                                                                           │ Return presigned URL
                                                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                        STORAGE LAYER                                                                                  │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  S3: Input Documents Bucket                                                                                                                   │   │
│  │  • Bucket: idp-input-documents-{env}-{account-id}                                                                                             │   │
│  │  • Encryption: AES-256 (SSE-S3)                                                                                                                │   │
│  │  • Versioning: Disabled                                                                                                                        │   │
│  │  • Lifecycle: Delete after 7 days                                                                                                              │   │
│  │  • Event Notifications: EventBridge enabled                                                                                                    │   │
│  │  • Public Access: Blocked                                                                                                                      │   │
│  └─────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘   │
│                                                                          │                                                                             │
│                                                                          │ S3 Event: ObjectCreated                                                     │
│                                                                          ▼                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  EventBridge: S3 Upload Rule                                                                                                                  │   │
│  │  • Rule: idp-s3-upload-trigger-{env}                                                                                                          │   │
│  │  • Event Pattern: S3 Object Created                                                                                                            │   │
│  │  • Target: SQS Queue                                                                                                                           │   │
│  └─────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘   │
│                                                                          │                                                                             │
└──────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┘
                                                                           │
                                                                           │ Send message
                                                                           ▼
┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐
│                                                       MESSAGING LAYER                                                                                 │
├─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┤
│                                                                                                                                                       │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  SQS: Document Processing Queue                                                                                                               │   │
│  │  • Queue: idp-document-processing-queue-{env}                                                                                                 │   │
│  │  • Type: Standard                                                                                                                              │   │
│  │  • Visibility Timeout: 900s (15 min)                                                                                                           │   │
│  │  • Message Retention: 14 days                                                                                                                  │   │
│  │  • Max Receive Count: 3                                                                                                                        │   │
│  │  • Long Polling: 20s                                                                                                                           │   │
│  │  • DLQ: idp-document-processing-dlq-{env}                                                                                                     │   │
│  └─────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘   │
│                                                                          │                                                                             │
│                                                                          │ Poll messages (batch: 10)                                                   │
│                                                                          ▼                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  Lambda: SQS Poller                                                                                                                            │   │
│  │  • Runtime: Python 3.12 (Docker)                                                                                                               │   │
│  │  • Memory: 256 MB                                                                                                                               │   │
│  │  • Timeout: 60s                                                                                                                                 │   │
│  │  • VPC: No                                                                                                                                      │   │
│  │  • Event Source: SQS Queue                                                                                                                      │   │
│  │  • Batch Size: 10                                                                                                                               │   │
│  └─────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘   │
│                                                                          │                                                                             │
│                                                                          │ Start execution                                                             │
│                                                                          ▼                                                                             │
│  ┌─────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────────┐   │
│  │  Dead Letter Queue (DLQ)                                                                                                                       │   │
│  │  • Queue: idp-document-processing-dlq-{env}                                                                                                   │   │
│  │  • Type: Standard                                                                                                                              │   │
│  │  • Message Retention: 14 days                                                                                                                  │   │
│  │  • CloudWatch Alarm: Triggers on message count > 0                                                                                            │   │
│  └─────────────────────────────────────────────────────────────────────┬───────────────────────────────────────────────────────────────────────────┘   │
│                                                                          │                                                                             │
└──────────────────────────────────────────────────────────────────────────┼─────────────────────────────────────────────────────────────────────────────┘
                                                                           │
                                                                           │ Trigger on DLQ message
                                                                           ▼
                                                                    Lambda: DLQ Analyzer
                                                                           │
                                                                           │ Publish alert
                                                                           ▼
                                                                    SNS → New Relic/Email



```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           CLIENT APPLICATIONS                                    │
│                    (On-Prem / AWS via Direct Connect)                           │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 │ HTTPS
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                          API GATEWAY (Regional)                                  │
│  POST /upload → Generate Pre-signed URL                                         │
│  GET /status/{id} → Get Processing Status                                       │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                    ┌────────────┴────────────┐
                    │                         │
                    ▼                         ▼
        ┌──────────────────┐      ┌──────────────────┐
        │  Lambda:         │      │   DynamoDB:      │
        │  Presigned URL   │─────▶│   Metadata       │
        │  Generator       │      │   Table          │
        └──────────────────┘      └──────────────────┘
                    │
                    │ Returns Pre-signed URL
                    ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         S3: Input Documents Bucket                               │
│  Client uploads document using pre-signed URL                                   │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 │ S3 Event Notification
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                            EventBridge Rule                                      │
│  Event Pattern: S3 Object Created                                               │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 │ Trigger
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    SQS: Document Processing Queue                                │
│  - Standard Queue                                                                │
│  - Visibility Timeout: 900s                                                      │
│  - Message Retention: 14 days                                                    │
│  - DLQ: Enabled (after 3 retries)                                               │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 │ Poll Messages
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                      STEP FUNCTIONS: Processing Workflow                         │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  1. Language Detection Lambda                                             │  │
│  │     ↓                                                                      │  │
│  │  2. Document Classification Lambda                                        │  │
│  │     ↓                                                                      │  │
│  │  3. Confidence Check (Choice State)                                       │  │
│  │     ├─ HIGH → Continue                                                    │  │
│  │     └─ LOW → Mark as LOW_CONFIDENCE → End                                │  │
│  │     ↓                                                                      │  │
│  │  4. Schema Extraction Lambda (calls BDA)                                  │  │
│  │     ↓                                                                      │  │
│  │  5. Status Update Lambda                                                  │  │
│  │     ↓                                                                      │  │
│  │  6. Callback Invocation (if configured)                                   │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  Error Handling: Retry with exponential backoff → DLQ after 3 attempts          │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                    ┌────────────┼────────────┐
                    │            │            │
                    ▼            ▼            ▼
        ┌──────────────┐  ┌──────────┐  ┌──────────────────┐
        │  Bedrock     │  │ DynamoDB │  │  S3: Output      │
        │  Data        │  │ Update   │  │  Documents       │
        │  Automation  │  │ Status   │  │  Bucket          │
        │  (BDA)       │  │          │  │                  │
        └──────────────┘  └──────────┘  └──────────────────┘
                                 │
                                 │ On Failure
                                 ▼
                    ┌──────────────────────────┐
                    │  SQS: Dead Letter Queue  │
                    │  - Failed messages       │
                    │  - Manual review         │
                    │  - CloudWatch Alarm      │
                    └──────────────────────────┘
                                 │
                                 │ Alert
                                 ▼
                    ┌──────────────────────────┐
                    │  CloudWatch Alarms       │
                    │  → SNS → New Relic       │
                    └──────────────────────────┘

VPC Configuration (Provided by Evoke):
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  VPC                                             │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │  Private Subnets (Lambda Functions)                                     │    │
│  │  - Language Detector                                                    │    │
│  │  - Document Classifier                                                  │    │
│  │  - Schema Extractor                                                     │    │
│  │  - BDA Orchestrator                                                     │    │
│  └────────────────────────────┬───────────────────────────────────────────┘    │
│                                │                                                 │
│                                │ Route to NAT Gateway                            │
│                                ▼                                                 │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │  Public Subnets                                                         │    │
│  │  - NAT Gateway (for Lambda outbound internet access)                   │    │
│  │  - Internet Gateway                                                     │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
└─────────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Infrastructure Overview

### 1.1 Architecture Principles
- Serverless-first approach for scalability and cost optimization
- Multi-AZ deployment for high availability
- Infrastructure as Code using Terraform
- Secure by default with least privilege access

### 1.2 AWS Services Used
- **Compute**: AWS Lambda (containerized)
- **Storage**: Amazon S3, Amazon DynamoDB
- **Orchestration**: AWS Step Functions
- **Messaging**: Amazon SQS (Standard Queue + DLQ)
- **AI/ML**: Amazon Bedrock Data Automation (BDA)
- **Integration**: Amazon EventBridge, API Gateway
- **Networking**: VPC (existing), NAT Gateway (existing)
- **Security**: AWS Secrets Manager, IAM
- **Container Registry**: Amazon ECR
- **Monitoring**: CloudWatch, New Relic

---

## 2. Network Architecture

### 2.1 VPC Configuration (Existing - Provided by Evoke)
```
VPC CIDR: [To be provided by Evoke]
Region: [EU region - TBD]
Availability Zones: 3 AZs for HA
```

### 2.2 Subnet Design
**Private Subnets** (Lambda functions):
- Private Subnet 1: AZ-1 - CIDR: [Evoke provided]
- Private Subnet 2: AZ-2 - CIDR: [Evoke provided]
- Private Subnet 3: AZ-3 - CIDR: [Evoke provided]

**Public Subnets** (NAT Gateway):
- Public Subnet 1: AZ-1 - CIDR: [Evoke provided]
- Public Subnet 2: AZ-2 - CIDR: [Evoke provided]

### 2.3 Connectivity
- **NAT Gateway**: Deployed in public subnets for Lambda outbound internet access
- **VPC Endpoints** (if available):
  - S3 Gateway Endpoint
  - DynamoDB Gateway Endpoint
  - Bedrock Interface Endpoint (for private connectivity)
  - ECR Interface Endpoints (for Lambda container pulls)
  - CloudWatch Logs Interface Endpoint

### 2.4 Security Groups

**Lambda Security Group** (`sg-lambda-idp`):
```hcl
Inbound Rules:
  - None (Lambda doesn't accept inbound traffic)

Outbound Rules:
  - HTTPS (443) to 0.0.0.0/0 (for AWS service APIs)
  - Description: Allow Lambda to call AWS services
```

**VPC Endpoint Security Group** (`sg-vpc-endpoints`):
```hcl
Inbound Rules:
  - HTTPS (443) from Lambda Security Group
  - Description: Allow Lambda to access VPC endpoints

Outbound Rules:
  - All traffic to 0.0.0.0/0
```

---

## 3. Storage Layer

### 3.1 Amazon S3 Buckets

#### 3.1.1 Input Documents Bucket
```hcl
Bucket Name: idp-input-documents-{environment}-{account-id}
Purpose: Store uploaded documents for processing
Lifecycle Policy:
  - Delete objects after 7 days (configurable)
Versioning: Disabled
Encryption: AES-256 (SSE-S3)
Public Access: Blocked
CORS: Enabled for pre-signed URL uploads
Event Notifications: EventBridge enabled
```

#### 3.1.2 Output Documents Bucket
```hcl
Bucket Name: idp-output-documents-{environment}-{account-id}
Purpose: Store processed JSON outputs
Lifecycle Policy:
  - Transition to IA after 30 days
  - Delete after 90 days (configurable)
Versioning: Enabled
Encryption: AES-256 (SSE-S3)
Public Access: Blocked
```

#### 3.1.3 Configuration Bucket
```hcl
Bucket Name: idp-config-{environment}-{account-id}
Purpose: Store document type configurations, prompts, schemas
Versioning: Enabled
Encryption: AES-256 (SSE-S3)
Public Access: Blocked
Structure:
  /configs/
    /{document-type}/
      config.json
      schema.json
      prompt.txt
```

#### 3.1.4 Audit/Logs Bucket
```hcl
Bucket Name: idp-audit-logs-{environment}-{account-id}
Purpose: Store audit trails and processing logs
Lifecycle Policy:
  - Transition to Glacier after 90 days
  - Delete after 365 days
Versioning: Enabled
Encryption: AES-256 (SSE-S3)
Object Lock: Enabled (compliance mode)
```

### 3.2 Amazon DynamoDB Tables

#### 3.2.1 Document Metadata Table
```hcl
Table Name: idp-document-metadata-{environment}
Purpose: Track document processing status and metadata
Partition Key: document_id (String)
Sort Key: None
Attributes:
  - document_id: String (PK)
  - upload_timestamp: Number (Unix timestamp)
  - presigned_url: String
  - callback_url: String
  - status: String (UPLOADED, PROCESSING, COMPLETED, FAILED)
  - document_type: String
  - classification_confidence: Number
  - extraction_confidence: Number
  - language: String
  - s3_input_key: String
  - s3_output_key: String
  - error_message: String
  - processing_duration_ms: Number
  - config_version: String
  - created_at: String (ISO 8601)
  - updated_at: String (ISO 8601)

GSI-1: status-index
  - Partition Key: status
  - Sort Key: upload_timestamp
  - Purpose: Query documents by status

GSI-2: document-type-index
  - Partition Key: document_type
  - Sort Key: upload_timestamp
  - Purpose: Query documents by type

Capacity:
  - Billing Mode: PAY_PER_REQUEST (On-Demand)
  - Auto-scaling: Not required with on-demand

TTL: enabled on 'ttl' attribute (optional cleanup after 90 days)
Point-in-Time Recovery: Enabled
Encryption: AWS managed key
Stream: Enabled (NEW_AND_OLD_IMAGES) for audit trail
```

#### 3.2.2 Configuration Cache Table
```hcl
Table Name: idp-config-cache-{environment}
Purpose: Cache document type configurations for fast lookup
Partition Key: document_type (String)
Sort Key: config_version (String)
Attributes:
  - document_type: String (PK)
  - config_version: String (SK)
  - schema: Map
  - prompt: String
  - confidence_threshold: Number
  - llm_model: String
  - extraction_fields: List
  - is_active: Boolean
  - updated_at: String

Capacity: PAY_PER_REQUEST
TTL: Not enabled (configs are long-lived)
Encryption: AWS managed key
```

---

## 4. Compute Layer

### 4.1 AWS Lambda Functions

All Lambda functions are containerized and deployed from ECR.

#### 4.1.1 Presigned URL Generator Lambda
```hcl
Function Name: idp-presigned-url-generator-{environment}
Runtime: Python 3.12 (Docker)
Memory: 512 MB
Timeout: 30 seconds
Architecture: arm64 (Graviton2 for cost savings)
VPC: Not required (public API Gateway integration)
Environment Variables:
  - INPUT_BUCKET_NAME
  - METADATA_TABLE_NAME
  - PRESIGNED_URL_EXPIRY (default: 3600)
  - LOG_LEVEL (INFO)
IAM Role Permissions:
  - s3:PutObject on input bucket
  - dynamodb:PutItem on metadata table
  - logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents
Reserved Concurrency: 100
```

#### 4.1.2 SQS Poller Lambda
```hcl
Function Name: idp-sqs-poller-{environment}
Runtime: Python 3.12 (Docker)
Memory: 256 MB
Timeout: 60 seconds
Architecture: arm64
VPC: Not required
Event Source: SQS Queue (idp-document-processing-queue)
Batch Size: 10 messages
Batch Window: 5 seconds
Environment Variables:
  - STATE_MACHINE_ARN
  - LOG_LEVEL
IAM Role Permissions:
  - sqs:ReceiveMessage, sqs:DeleteMessage, sqs:ChangeMessageVisibility
  - states:StartExecution
  - logs:*
Reserved Concurrency: 10
```

#### 4.1.3 BDA Orchestrator Lambda
```hcl
Function Name: idp-bda-orchestrator-{environment}
Runtime: Python 3.12 (Docker)
Memory: 1024 MB
Timeout: 300 seconds (5 minutes)
Architecture: arm64
VPC: Enabled (private subnets)
Subnets: [Private subnet IDs from Evoke]
Security Groups: [Lambda security group]
Environment Variables:
  - BDA_PROJECT_ARN
  - CONFIG_BUCKET_NAME
  - CONFIG_CACHE_TABLE_NAME
  - BEDROCK_REGION
  - LOG_LEVEL
IAM Role Permissions:
  - bedrock:InvokeModel
  - bedrock:StartDocumentAnalysis (BDA)
  - bedrock:GetDocumentAnalysis (BDA)
  - s3:GetObject on input/config buckets
  - dynamodb:GetItem, dynamodb:Query on config cache
  - logs:*
Reserved Concurrency: 50
```

#### 4.1.3 BDA Orchestrator Lambda
```hcl
Function Name: idp-bda-orchestrator-{environment}
Runtime: Python 3.12 (Docker)
Memory: 1024 MB
Timeout: 300 seconds (5 minutes)
Architecture: arm64
VPC: Enabled (private subnets)
Subnets: [Private subnet IDs from Evoke]
Security Groups: [Lambda security group]
Environment Variables:
  - BDA_PROJECT_ARN
  - CONFIG_BUCKET_NAME
  - CONFIG_CACHE_TABLE_NAME
  - BEDROCK_REGION
  - LOG_LEVEL
IAM Role Permissions:
  - bedrock:InvokeModel
  - bedrock:StartDocumentAnalysis (BDA)
  - bedrock:GetDocumentAnalysis (BDA)
  - s3:GetObject on input/config buckets
  - dynamodb:GetItem, dynamodb:Query on config cache
  - logs:*
Reserved Concurrency: 50
```

#### 4.1.4 Language Detection Lambda
```hcl
Function Name: idp-language-detector-{environment}
Runtime: Python 3.12 (Docker)
Memory: 512 MB
Timeout: 60 seconds
Architecture: arm64
VPC: Enabled
Environment Variables:
  - BEDROCK_REGION
  - COMPREHEND_ENABLED (true/false)
IAM Role Permissions:
  - comprehend:DetectDominantLanguage
  - bedrock:InvokeModel (fallback)
  - s3:GetObject
  - logs:*
Reserved Concurrency: 50
```

#### 4.1.4 Language Detection Lambda
```hcl
Function Name: idp-language-detector-{environment}
Runtime: Python 3.12 (Docker)
Memory: 512 MB
Timeout: 60 seconds
Architecture: arm64
VPC: Enabled
Environment Variables:
  - BEDROCK_REGION
  - COMPREHEND_ENABLED (true/false)
IAM Role Permissions:
  - comprehend:DetectDominantLanguage
  - bedrock:InvokeModel (fallback)
  - s3:GetObject
  - logs:*
Reserved Concurrency: 50
```

#### 4.1.5 Document Classifier Lambda
```hcl
Function Name: idp-document-classifier-{environment}
Runtime: Python 3.12 (Docker)
Memory: 1024 MB
Timeout: 120 seconds
Architecture: arm64
VPC: Enabled
Environment Variables:
  - BDA_PROJECT_ARN
  - CONFIG_BUCKET_NAME
  - CONFIDENCE_THRESHOLD (default: 0.85)
IAM Role Permissions:
  - bedrock:InvokeModel
  - s3:GetObject
  - dynamodb:GetItem
  - logs:*
Reserved Concurrency: 50
```

#### 4.1.5 Document Classifier Lambda
```hcl
Function Name: idp-document-classifier-{environment}
Runtime: Python 3.12 (Docker)
Memory: 1024 MB
Timeout: 120 seconds
Architecture: arm64
VPC: Enabled
Environment Variables:
  - BDA_PROJECT_ARN
  - CONFIG_BUCKET_NAME
  - CONFIDENCE_THRESHOLD (default: 0.85)
IAM Role Permissions:
  - bedrock:InvokeModel
  - s3:GetObject
  - dynamodb:GetItem
  - logs:*
Reserved Concurrency: 50
```

#### 4.1.6 Schema Extractor Lambda
```hcl
Function Name: idp-schema-extractor-{environment}
Runtime: Python 3.12 (Docker)
Memory: 2048 MB
Timeout: 300 seconds
Architecture: arm64
VPC: Enabled
Environment Variables:
  - BDA_PROJECT_ARN
  - CONFIG_BUCKET_NAME
  - OUTPUT_BUCKET_NAME
IAM Role Permissions:
  - bedrock:InvokeModel
  - s3:GetObject, s3:PutObject
  - dynamodb:GetItem
  - logs:*
Reserved Concurrency: 50
```

#### 4.1.6 Schema Extractor Lambda
```hcl
Function Name: idp-schema-extractor-{environment}
Runtime: Python 3.12 (Docker)
Memory: 2048 MB
Timeout: 300 seconds
Architecture: arm64
VPC: Enabled
Environment Variables:
  - BDA_PROJECT_ARN
  - CONFIG_BUCKET_NAME
  - OUTPUT_BUCKET_NAME
IAM Role Permissions:
  - bedrock:InvokeModel
  - s3:GetObject, s3:PutObject
  - dynamodb:GetItem
  - logs:*
Reserved Concurrency: 50
```

#### 4.1.7 Status Updater Lambda
```hcl
Function Name: idp-status-updater-{environment}
Runtime: Python 3.12 (Docker)
Memory: 256 MB
Timeout: 30 seconds
Architecture: arm64
VPC: Not required
Environment Variables:
  - METADATA_TABLE_NAME
  - CALLBACK_ENABLED (true/false)
IAM Role Permissions:
  - dynamodb:UpdateItem
  - logs:*
  - (Optional) execute-api:Invoke for callbacks
Reserved Concurrency: 100
```

#### 4.1.7 Status Updater Lambda
```hcl
Function Name: idp-status-updater-{environment}
Runtime: Python 3.12 (Docker)
Memory: 256 MB
Timeout: 30 seconds
Architecture: arm64
VPC: Not required
Environment Variables:
  - METADATA_TABLE_NAME
  - CALLBACK_ENABLED (true/false)
IAM Role Permissions:
  - dynamodb:UpdateItem
  - logs:*
  - (Optional) execute-api:Invoke for callbacks
Reserved Concurrency: 100
```

#### 4.1.8 Document Cleanup Lambda
```hcl
Function Name: idp-document-cleanup-{environment}
Runtime: Python 3.12 (Docker)
Memory: 256 MB
Timeout: 60 seconds
Architecture: arm64
VPC: Not required
Trigger: EventBridge scheduled rule (daily)
Environment Variables:
  - INPUT_BUCKET_NAME
  - RETENTION_DAYS (default: 7)
IAM Role Permissions:
  - s3:ListBucket, s3:DeleteObject
  - logs:*
```

#### 4.1.9 DLQ Analyzer Lambda
```hcl
Function Name: idp-dlq-analyzer-{environment}
Runtime: Python 3.12 (Docker)
Memory: 256 MB
Timeout: 60 seconds
Architecture: arm64
VPC: Not required
Trigger: CloudWatch Event (DLQ message count > 0)
Environment Variables:
  - DLQ_QUEUE_URL
  - SNS_TOPIC_ARN
  - LOG_LEVEL
IAM Role Permissions:
  - sqs:ReceiveMessage, sqs:GetQueueAttributes
  - sns:Publish
  - logs:*
Purpose: Analyze DLQ messages and send detailed alerts with error context
```

### 4.2 Lambda Layer (Shared Dependencies)
```hcl
Layer Name: idp-common-dependencies-{environment}
Compatible Runtimes: python3.12
Architecture: arm64
Contents:
  - boto3 (latest)
  - botocore
  - requests
  - python-json-logger
  - Custom utility modules
Size: < 50 MB
```

---

## 5. Orchestration Layer

### 5.1 Amazon SQS Queues

#### 5.1.1 Document Processing Queue

```hcl
Queue Name: idp-document-processing-queue-{environment}
Queue Type: Standard
Purpose: Buffer S3 upload events and trigger Step Functions processing
```

**Configuration**:
```hcl
resource "aws_sqs_queue" "document_processing" {
  name                       = "idp-document-processing-queue-${var.environment}"
  visibility_timeout_seconds = 900  # 15 minutes (Step Function max duration)
  message_retention_seconds  = 1209600  # 14 days
  max_message_size          = 262144  # 256 KB
  delay_seconds             = 0
  receive_wait_time_seconds = 20  # Long polling
  
  # Dead Letter Queue configuration
  redrive_policy = jsonencode({
    deadLetterTargetArn = aws_sqs_queue.document_processing_dlq.arn
    maxReceiveCount     = 3  # Retry 3 times before moving to DLQ
  })
  
  # Encryption
  sqs_managed_sse_enabled = true
  
  tags = var.tags
}
```

**Queue Attributes**:
- **Visibility Timeout**: 900 seconds (matches Step Function timeout)
- **Message Retention**: 14 days
- **Max Receive Count**: 3 (after 3 failed attempts, move to DLQ)
- **Long Polling**: 20 seconds (reduces empty receives)
- **Encryption**: SSE-SQS (AWS managed)

**Message Format**:
```json
{
  "Records": [
    {
      "eventVersion": "2.1",
      "eventSource": "aws:s3",
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "idp-input-documents-dev-123456"
        },
        "object": {
          "key": "documents/550e8400-e29b-41d4-a716-446655440000.pdf",
          "size": 1024000
        }
      }
    }
  ]
}
```

#### 5.1.2 Dead Letter Queue (DLQ)

```hcl
Queue Name: idp-document-processing-dlq-{environment}
Queue Type: Standard
Purpose: Store failed messages for manual review and debugging
```

**Configuration**:
```hcl
resource "aws_sqs_queue" "document_processing_dlq" {
  name                       = "idp-document-processing-dlq-${var.environment}"
  message_retention_seconds  = 1209600  # 14 days
  
  # Encryption
  sqs_managed_sse_enabled = true
  
  tags = merge(var.tags, {
    Purpose = "DeadLetterQueue"
  })
}
```

**DLQ Monitoring**:
```hcl
resource "aws_cloudwatch_metric_alarm" "dlq_messages" {
  alarm_name          = "idp-dlq-messages-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Average"
  threshold           = 0
  alarm_description   = "Alert when messages appear in DLQ"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    QueueName = aws_sqs_queue.document_processing_dlq.name
  }
}
```

**DLQ Message Analysis Lambda**:
```hcl
Function Name: idp-dlq-analyzer-{environment}
Purpose: Analyze DLQ messages and send detailed alerts
Trigger: CloudWatch Event (DLQ message count > 0)
```

#### 5.1.3 SQS to Step Functions Integration

**EventBridge Rule**:
```hcl
resource "aws_cloudwatch_event_rule" "s3_to_sqs" {
  name        = "idp-s3-upload-to-sqs-${var.environment}"
  description = "Route S3 upload events to SQS"
  
  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
    detail = {
      bucket = {
        name = [aws_s3_bucket.input_documents.id]
      }
    }
  })
}

resource "aws_cloudwatch_event_target" "sqs" {
  rule      = aws_cloudwatch_event_rule.s3_to_sqs.name
  target_id = "SendToSQS"
  arn       = aws_sqs_queue.document_processing.arn
}
```

**Lambda SQS Poller** (triggers Step Functions):
```hcl
Function Name: idp-sqs-poller-{environment}
Runtime: Python 3.12 (Docker)
Memory: 256 MB
Timeout: 60 seconds
Reserved Concurrency: 10
Batch Size: 10 messages
Batch Window: 5 seconds

Event Source Mapping:
  - SQS Queue: idp-document-processing-queue
  - Batch Size: 10
  - Maximum Batching Window: 5 seconds
  - Function Response Types: ReportBatchItemFailures
```

**SQS Poller Lambda Code**:
```python
import json
import boto3
import os

stepfunctions = boto3.client('stepfunctions')
STATE_MACHINE_ARN = os.environ['STATE_MACHINE_ARN']

def handler(event, context):
    """
    Poll SQS messages and trigger Step Functions
    """
    batch_item_failures = []
    
    for record in event['Records']:
        try:
            # Parse S3 event from SQS message
            s3_event = json.loads(record['body'])
            
            for s3_record in s3_event['Records']:
                bucket = s3_record['s3']['bucket']['name']
                key = s3_record['s3']['object']['key']
                
                # Extract document_id from key
                document_id = key.split('/')[-1].split('.')[0]
                
                # Start Step Functions execution
                response = stepfunctions.start_execution(
                    stateMachineArn=STATE_MACHINE_ARN,
                    name=f"doc-{document_id}-{int(time.time())}",
                    input=json.dumps({
                        'document_id': document_id,
                        's3_bucket': bucket,
                        's3_key': key
                    })
                )
                
                logger.info(f"Started execution: {response['executionArn']}")
        
        except Exception as e:
            logger.error(f"Failed to process message: {str(e)}")
            # Report failure to SQS (message will be retried)
            batch_item_failures.append({
                'itemIdentifier': record['messageId']
            })
    
    return {
        'batchItemFailures': batch_item_failures
    }
```

**IAM Role for SQS Poller**:
```hcl
resource "aws_iam_role_policy" "sqs_poller" {
  name = "sqs-poller-policy"
  role = aws_iam_role.sqs_poller.id
  
  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "sqs:ReceiveMessage",
          "sqs:DeleteMessage",
          "sqs:GetQueueAttributes",
          "sqs:ChangeMessageVisibility"
        ]
        Resource = aws_sqs_queue.document_processing.arn
      },
      {
        Effect = "Allow"
        Action = [
          "states:StartExecution"
        ]
        Resource = aws_sfn_state_machine.document_processing.arn
      },
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "*"
      }
    ]
  })
}
```

### 5.2 AWS Step Functions State Machine

```hcl
State Machine Name: idp-document-processing-{environment}
Type: STANDARD
Execution Role: StepFunctionsExecutionRole
Logging: CloudWatch Logs (ALL events)
Tracing: AWS X-Ray enabled
```

**State Machine Definition** (see Application LLD for detailed JSON)

**Key States**:
1. Language Detection
2. Document Classification
3. Schema Extraction
4. Status Update
5. Error Handling

**Execution Settings**:
- Max Execution Time: 15 minutes
- Retry Strategy: Exponential backoff (3 attempts)
- Error Handling: Failures reported back to SQS (message goes to DLQ after 3 retries)

**Error Handling Flow**:
```
Step Function Failure
    ↓
Lambda returns error to SQS
    ↓
SQS increments receive count
    ↓
If receive count < 3: Retry (with visibility timeout)
    ↓
If receive count >= 3: Move to DLQ
    ↓
CloudWatch Alarm triggered
    ↓
SNS notification → New Relic alert
```

---

## 6. Integration Layer

### 6.1 Amazon API Gateway

```hcl
API Name: idp-api-{environment}
Type: REST API
Endpoint Type: REGIONAL
Authorization: IAM (AWS_IAM)
Throttling:
  - Rate Limit: 1000 requests/second
  - Burst Limit: 2000 requests
Stage: {environment} (dev, test, prod)
```

**Endpoints**:

**POST /upload**
- Integration: Lambda (Presigned URL Generator)
- Request Validation: Enabled
- Request Body Schema:
```json
{
  "document_type": "string (optional)",
  "callback_url": "string (optional)",
  "metadata": "object (optional)"
}
```
- Response: 200 with presigned URL and document_id

**GET /status/{document_id}**
- Integration: DynamoDB direct integration
- Authorization: IAM
- Response: Document processing status

### 6.2 Amazon EventBridge

**Event Bus**: default

**Rules**:

**S3 Document Upload Rule**
```hcl
Rule Name: idp-s3-upload-trigger-{environment}
Event Pattern:
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["idp-input-documents-{environment}-{account-id}"]
    }
  }
}
Target: SQS Queue (idp-document-processing-queue)
```

**Processing Completion Rule**
```hcl
Rule Name: idp-processing-complete-{environment}
Event Pattern:
{
  "source": ["aws.states"],
  "detail-type": ["Step Functions Execution Status Change"],
  "detail": {
    "status": ["SUCCEEDED"],
    "stateMachineArn": ["arn:aws:states:..."]
  }
}
Target: Status Updater Lambda
```

**DLQ Alert Rule**
```hcl
Rule Name: idp-dlq-alert-{environment}
Event Pattern:
{
  "source": ["aws.sqs"],
  "detail-type": ["SQS Queue Message"],
  "detail": {
    "queueName": ["idp-document-processing-dlq-{environment}"]
  }
}
Target: SNS Topic (idp-alerts)
```

---

## 7. Security & IAM

### 7.1 IAM Roles

#### Lambda Execution Role
```hcl
Role Name: idp-lambda-execution-role-{environment}
Managed Policies:
  - AWSLambdaVPCAccessExecutionRole
Custom Inline Policy: LambdaServiceAccess
  - S3: GetObject, PutObject on IDP buckets
  - DynamoDB: GetItem, PutItem, UpdateItem, Query
  - Bedrock: InvokeModel, StartDocumentAnalysis
  - Comprehend: DetectDominantLanguage
  - Secrets Manager: GetSecretValue
  - CloudWatch Logs: CreateLogGroup, CreateLogStream, PutLogEvents
  - X-Ray: PutTraceSegments, PutTelemetryRecords
```

#### Step Functions Execution Role
```hcl
Role Name: idp-stepfunctions-execution-role-{environment}
Permissions:
  - Lambda: InvokeFunction on all IDP Lambdas
  - CloudWatch Logs: CreateLogGroup, CreateLogStream, PutLogEvents
  - X-Ray: PutTraceSegments
  - EventBridge: PutEvents
```

#### API Gateway Execution Role
```hcl
Role Name: idp-apigateway-execution-role-{environment}
Permissions:
  - Lambda: InvokeFunction on Presigned URL Generator
  - DynamoDB: GetItem on metadata table
  - CloudWatch Logs: CreateLogGroup, CreateLogStream, PutLogEvents
```

### 7.2 Secrets Manager

```hcl
Secret Name: idp/{environment}/config
Secret Type: Key-value pairs
Contents:
  - new_relic_license_key
  - callback_api_key (if needed)
  - external_api_credentials (if any)
Rotation: Manual (no auto-rotation)
Encryption: AWS managed key
```

### 7.3 KMS Keys (Optional - for enhanced encryption)

```hcl
Key Alias: alias/idp-{environment}
Key Type: Symmetric
Usage: Encrypt S3 objects, DynamoDB tables
Key Policy: Allow Lambda and Step Functions to decrypt
Rotation: Enabled (annual)
```

---

## 8. Monitoring & Observability

### 8.1 CloudWatch Log Groups

```hcl
Log Groups:
  - /aws/lambda/idp-presigned-url-generator-{environment}
  - /aws/lambda/idp-bda-orchestrator-{environment}
  - /aws/lambda/idp-language-detector-{environment}
  - /aws/lambda/idp-document-classifier-{environment}
  - /aws/lambda/idp-schema-extractor-{environment}
  - /aws/lambda/idp-status-updater-{environment}
  - /aws/lambda/idp-document-cleanup-{environment}
  - /aws/stepfunctions/idp-document-processing-{environment}
  - /aws/apigateway/idp-api-{environment}

Retention: 30 days (configurable)
Encryption: Enabled
```

### 8.2 CloudWatch Metrics

**Custom Metrics**:
- DocumentsProcessed (Count)
- ProcessingDuration (Milliseconds)
- ClassificationAccuracy (Percent)
- ExtractionConfidence (Percent)
- FailureRate (Percent)
- LanguageDetectionAccuracy (Percent)
- SQSQueueDepth (Count)
- DLQMessageCount (Count)
- SQSMessageAge (Seconds)

**Dimensions**:
- Environment
- DocumentType
- Language
- Status
- QueueName

### 8.3 CloudWatch Alarms

```hcl
Alarms:
  - Lambda Error Rate > 5% (5 min period)
  - Step Function Failed Executions > 10 (5 min period)
  - API Gateway 5xx Errors > 50 (5 min period)
  - DynamoDB Throttled Requests > 0
  - Lambda Duration > 80% of timeout
  - S3 Bucket Size > threshold
  - SQS DLQ Message Count > 0 (CRITICAL)
  - SQS Queue Depth > 1000 (WARNING)
  - SQS Message Age > 900 seconds (WARNING)
  - SQS Poller Lambda Errors > 5 (5 min period)

Actions:
  - SNS Topic: idp-alerts-{environment}
  - New Relic integration
```

**SQS-Specific Alarms**:
```hcl
# DLQ Messages Alarm (Critical)
resource "aws_cloudwatch_metric_alarm" "dlq_messages" {
  alarm_name          = "idp-dlq-messages-critical-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Average"
  threshold           = 0
  alarm_description   = "CRITICAL: Messages in DLQ require immediate attention"
  alarm_actions       = [aws_sns_topic.critical_alerts.arn]
  treat_missing_data  = "notBreaching"
  
  dimensions = {
    QueueName = aws_sqs_queue.document_processing_dlq.name
  }
}

# Queue Depth Alarm (Warning)
resource "aws_cloudwatch_metric_alarm" "queue_depth" {
  alarm_name          = "idp-queue-depth-warning-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Average"
  threshold           = 1000
  alarm_description   = "WARNING: SQS queue depth is high, may indicate processing bottleneck"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    QueueName = aws_sqs_queue.document_processing.name
  }
}

# Message Age Alarm (Warning)
resource "aws_cloudwatch_metric_alarm" "message_age" {
  alarm_name          = "idp-message-age-warning-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ApproximateAgeOfOldestMessage"
  namespace           = "AWS/SQS"
  period              = 300
  statistic           = "Maximum"
  threshold           = 900  # 15 minutes
  alarm_description   = "WARNING: Messages are aging in queue, processing may be slow"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    QueueName = aws_sqs_queue.document_processing.name
  }
}
```

### 8.4 AWS X-Ray Tracing

```hcl
Enabled for:
  - All Lambda functions
  - Step Functions
  - API Gateway
Sampling Rate: 10% (adjustable)
```

### 8.5 New Relic Integration

```hcl
Integration Type: CloudWatch Logs subscription
Log Streaming: Real-time via Kinesis Firehose
Metrics: Custom metrics pushed via Lambda extension
Dashboards:
  - Document Processing Overview
  - Error Rate by Document Type
  - Performance Metrics
  - Cost Analysis
```

---

## 9. Disaster Recovery & Backup

### 9.1 Backup Strategy

**DynamoDB**:
- Point-in-Time Recovery: Enabled
- On-Demand Backups: Weekly (retained for 35 days)
- Cross-Region Replication: Optional (for DR)

**S3**:
- Versioning: Enabled on critical buckets
- Cross-Region Replication: Optional
- Backup: Not required (lifecycle policies handle retention)

**Lambda**:
- Code stored in ECR with image versioning
- Terraform state manages infrastructure

### 9.2 Recovery Objectives

```
RTO (Recovery Time Objective): 4 hours
RPO (Recovery Point Objective): 1 hour
```

---

## 10. Cost Optimization

### 10.1 Strategies

1. **Lambda**:
   - ARM64 architecture (20% cost savings)
   - Right-sized memory allocation
   - Reserved concurrency to prevent over-provisioning

2. **DynamoDB**:
   - On-Demand pricing for variable workloads
   - TTL for automatic data cleanup

3. **S3**:
   - Lifecycle policies for automatic tiering
   - Intelligent-Tiering for unpredictable access patterns

4. **Step Functions**:
   - Standard workflows (cheaper for long-running processes)

5. **VPC**:
   - VPC Endpoints to avoid NAT Gateway data transfer costs

### 10.2 Cost Monitoring

- AWS Cost Explorer tags: Environment, Project, CostCenter
- Budget Alerts: Monthly threshold alerts
- Cost Anomaly Detection: Enabled

---

## 11. Terraform Module Structure

```
terraform/
├── environments/
│   ├── dev/
│   │   ├── main.tf
│   │   ├── terraform.tfvars
│   │   └── backend.tf
│   ├── test/
│   └── prod/
├── modules/
│   ├── networking/
│   │   ├── main.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── storage/
│   │   ├── s3.tf
│   │   ├── dynamodb.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── messaging/
│   │   ├── sqs.tf
│   │   ├── dlq.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── compute/
│   │   ├── lambda.tf
│   │   ├── ecr.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── orchestration/
│   │   ├── stepfunctions.tf
│   │   ├── eventbridge.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── api/
│   │   ├── apigateway.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   ├── security/
│   │   ├── iam.tf
│   │   ├── secrets.tf
│   │   ├── kms.tf
│   │   ├── variables.tf
│   │   └── outputs.tf
│   └── monitoring/
│       ├── cloudwatch.tf
│       ├── alarms.tf
│       ├── variables.tf
│       └── outputs.tf
├── provider.tf
├── variables.tf
├── outputs.tf
└── versions.tf
```

---

## 12. Deployment Prerequisites

### 12.1 Required from Evoke

- VPC ID and subnet IDs (private and public)
- NAT Gateway IDs
- Security group IDs (if pre-existing)
- AWS Account ID and Region
- IAM permissions for Terraform execution
- S3 bucket for Terraform state
- GitLab runner access to AWS
- New Relic account and license key

### 12.2 Terraform Backend Configuration

```hcl
terraform {
  backend "s3" {
    bucket         = "evoke-terraform-state-{region}"
    key            = "idp/{environment}/terraform.tfstate"
    region         = "eu-west-1"
    encrypt        = true
    dynamodb_table = "terraform-state-lock"
  }
}
```

---

## 13. Scaling Considerations

### 13.1 Auto-Scaling Configuration

**Lambda**:
- Concurrent executions: 1000 (account limit)
- Reserved concurrency per function: 50-100
- Provisioned concurrency: Not required (cold start acceptable)

**DynamoDB**:
- On-Demand mode handles auto-scaling automatically
- Monitor for hot partitions

**Step Functions**:
- Max concurrent executions: 1000 (soft limit, can be increased)

### 13.2 Performance Targets

```
Document Processing Throughput: 100 documents/minute
API Response Time: < 500ms (presigned URL generation)
End-to-End Processing Time: < 2 minutes (average)
Classification Accuracy: > 95%
Extraction Accuracy: > 90%
```

---

## 14. Compliance & Data Residency

### 14.1 EU Compliance

- All resources deployed in EU regions
- Data residency: EU only
- GDPR compliance: Data retention policies enforced
- Bedrock models: EU-compliant models only

### 14.2 Audit Trail

- CloudTrail: Enabled for all API calls
- S3 Access Logs: Enabled on all buckets
- DynamoDB Streams: Enabled for audit trail
- VPC Flow Logs: Enabled (if required by Evoke)

---

## 15. Infrastructure Validation

### 15.1 Terraform Validation Steps

```bash
# Format check
terraform fmt -check -recursive

# Validation
terraform validate

# Security scan
tfsec .

# Cost estimation
infracost breakdown --path .

# Plan review
terraform plan -out=tfplan
```

### 15.2 Post-Deployment Validation

- [ ] All Lambda functions deployed successfully
- [ ] Step Functions state machine executable
- [ ] API Gateway endpoints accessible
- [ ] S3 buckets created with correct policies
- [ ] DynamoDB tables created with correct schema
- [ ] IAM roles have correct permissions
- [ ] CloudWatch alarms configured
- [ ] New Relic integration working
- [ ] X-Ray tracing enabled
- [ ] Secrets stored in Secrets Manager

---

## 16. Appendix

### 16.1 Resource Naming Convention

```
Pattern: {service}-{resource-type}-{environment}
Example: idp-lambda-bda-orchestrator-dev

Tags (mandatory):
  - Project: IDP
  - Environment: dev/test/prod
  - ManagedBy: Terraform
  - CostCenter: [Evoke provided]
  - Owner: [Team name]
```

### 16.2 Environment-Specific Configurations

| Configuration | Dev | Test | Prod |
|--------------|-----|------|------|
| Lambda Memory | 512-1024 MB | 1024-2048 MB | 1024-2048 MB |
| Lambda Timeout | 60-300s | 60-300s | 60-300s |
| DynamoDB Mode | On-Demand | On-Demand | On-Demand |
| S3 Retention | 7 days | 30 days | 90 days |
| Log Retention | 7 days | 30 days | 90 days |
| Backup Retention | 7 days | 30 days | 35 days |
| Reserved Concurrency | 10 | 50 | 100 |
| SQS Visibility Timeout | 900s | 900s | 900s |
| SQS Message Retention | 14 days | 14 days | 14 days |
| SQS Max Receive Count | 3 | 3 | 3 |
| SQS Batch Size | 10 | 10 | 10 |

---

**End of Infrastructure LLD**
