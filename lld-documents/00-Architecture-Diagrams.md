# IDP Solution - Architecture Diagrams
## Intelligent Document Processing (IDP)

### Document Control
- **Version**: 1.0
- **Last Updated**: January 14, 2026
- **Status**: Draft

---

## 1. High-Level End-to-End Architecture

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
                                 │ Send Message
                                 ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                    SQS: Document Processing Queue                                │
│  - Standard Queue                                                                │
│  - Visibility Timeout: 900s                                                      │
│  - Message Retention: 14 days                                                    │
│  - DLQ: Enabled (after 3 retries)                                               │
└────────────────────────────────┬────────────────────────────────────────────────┘
                                 │
                                 │ Poll Messages (Batch: 10)
                                 ▼
        ┌──────────────────────────────────────┐
        │  Lambda: SQS Poller                  │
        │  - Reads SQS messages                │
        │  - Triggers Step Functions           │
        └────────────────┬─────────────────────┘
                         │
                         │ Start Execution
                         ▼

┌─────────────────────────────────────────────────────────────────────────────────┐
│                      STEP FUNCTIONS: Processing Workflow                         │
│                                                                                  │
│  ┌──────────────────────────────────────────────────────────────────────────┐  │
│  │  1. Language Detection Lambda                                             │  │
│  │     ├─ AWS Comprehend (primary)                                           │  │
│  │     └─ Bedrock LLM (fallback)                                             │  │
│  │     ↓                                                                      │  │
│  │  2. Document Classification Lambda                                        │  │
│  │     └─ BDA Classification Blueprint                                       │  │
│  │     ↓                                                                      │  │
│  │  3. Confidence Check (Choice State)                                       │  │
│  │     ├─ HIGH (≥0.85) → Continue                                            │  │
│  │     └─ LOW (<0.85) → Mark as LOW_CONFIDENCE → End                        │  │
│  │     ↓                                                                      │  │
│  │  4. Schema Extraction Lambda                                              │  │
│  │     └─ BDA Extraction Blueprint                                           │  │
│  │     ↓                                                                      │  │
│  │  5. Status Update Lambda                                                  │  │
│  │     ├─ Update DynamoDB                                                    │  │
│  │     └─ Trigger Callback (if configured)                                   │  │
│  │     ↓                                                                      │  │
│  │  6. Success → End                                                         │  │
│  └──────────────────────────────────────────────────────────────────────────┘  │
│                                                                                  │
│  Error Handling:                                                                │
│  - Retry with exponential backoff (2s, 4s, 8s)                                 │
│  - Max 3 attempts per Lambda                                                    │
│  - On final failure → Return error to SQS                                       │
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
                                 │ On Failure (after 3 SQS retries)
                                 ▼
                    ┌──────────────────────────┐
                    │  SQS: Dead Letter Queue  │
                    │  - Failed messages       │
                    │  - Manual review         │
                    │  - Retention: 14 days    │
                    └────────────┬─────────────┘
                                 │
                                 │ Trigger on message arrival
                                 ▼
                    ┌──────────────────────────┐
                    │  Lambda: DLQ Analyzer    │
                    │  - Parse error details   │
                    │  - Send detailed alert   │
                    └────────────┬─────────────┘
                                 │
                                 │ Publish Alert
                                 ▼
                    ┌──────────────────────────┐
                    │  SNS Topic: Alerts       │
                    │  → Email                 │
                    │  → New Relic             │
                    │  → Slack                 │
                    └──────────────────────────┘
```

---

## 2. VPC and Networking Architecture

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                                  VPC (Evoke Provided)                            │
│                            CIDR: 10.0.0.0/16 (example)                          │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │                         Availability Zone 1                             │    │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │    │
│  │  │  Private Subnet 1 (10.0.1.0/24)                                   │  │    │
│  │  │  ┌────────────────────────────────────────────────────────────┐  │  │    │
│  │  │  │  Lambda Functions (VPC-enabled):                            │  │  │    │
│  │  │  │  - Language Detector                                        │  │  │    │
│  │  │  │  - Document Classifier                                      │  │  │    │
│  │  │  │  - Schema Extractor                                         │  │  │    │
│  │  │  │  - BDA Orchestrator                                         │  │  │    │
│  │  │  │                                                              │  │  │    │
│  │  │  │  Security Group: sg-lambda-idp                              │  │  │    │
│  │  │  │  Outbound: HTTPS (443) to 0.0.0.0/0                         │  │  │    │
│  │  │  └────────────────────────────────────────────────────────────┘  │  │    │
│  │  └──────────────────────────────────────────────────────────────────┘  │    │
│  │                                                                          │    │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │    │
│  │  │  Public Subnet 1 (10.0.101.0/24)                                 │  │    │
│  │  │  ┌────────────────────────────────────────────────────────────┐  │  │    │
│  │  │  │  NAT Gateway                                                │  │  │    │
│  │  │  │  - Elastic IP attached                                      │  │  │    │
│  │  │  │  - Routes traffic from private subnets to internet          │  │  │    │
│  │  │  └────────────────────────────────────────────────────────────┘  │  │    │
│  │  └──────────────────────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │                         Availability Zone 2                             │    │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │    │
│  │  │  Private Subnet 2 (10.0.2.0/24)                                   │  │    │
│  │  │  - Lambda Functions (same as AZ1)                                 │  │    │
│  │  └──────────────────────────────────────────────────────────────────┘  │    │
│  │  ┌──────────────────────────────────────────────────────────────────┐  │    │
│  │  │  Public Subnet 2 (10.0.102.0/24)                                  │  │    │
│  │  │  - NAT Gateway (optional for HA)                                  │  │    │
│  │  └──────────────────────────────────────────────────────────────────┘  │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  ┌────────────────────────────────────────────────────────────────────────┐    │
│  │  VPC Endpoints (Optional - Cost Optimization)                          │    │
│  │  - S3 Gateway Endpoint                                                  │    │
│  │  - DynamoDB Gateway Endpoint                                            │    │
│  │  - Bedrock Interface Endpoint                                           │    │
│  │  - ECR Interface Endpoints (api, dkr)                                   │    │
│  │  - CloudWatch Logs Interface Endpoint                                   │    │
│  │  - SQS Interface Endpoint                                               │    │
│  │  - Secrets Manager Interface Endpoint                                   │    │
│  └────────────────────────────────────────────────────────────────────────┘    │
│                                                                                  │
│  Internet Gateway                                                                │
│  └─────────────────────────────────────────────────────────────────────────────┘
                                 │
                                 │ Internet Access
                                 ▼
                          Public Internet
```

---

## 3. Data Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                              DATA FLOW                                           │
└─────────────────────────────────────────────────────────────────────────────────┘

1. UPLOAD PHASE
   Client → API Gateway → Lambda (Presigned URL) → DynamoDB (metadata)
   Client → S3 (using presigned URL) → Document stored

2. TRIGGER PHASE
   S3 Event → EventBridge → SQS Queue → Lambda (SQS Poller) → Step Functions

3. PROCESSING PHASE
   Step Functions → Lambda (Language Detection) → Comprehend/Bedrock
                 ↓
   Step Functions → Lambda (Classification) → BDA (Classification Blueprint)
                 ↓
   Step Functions → Choice (Confidence Check)
                 ↓
   Step Functions → Lambda (Extraction) → BDA (Extraction Blueprint)
                 ↓
   Step Functions → Lambda (Status Update) → DynamoDB + S3 (output)

4. OUTPUT PHASE
   S3 (output JSON) ← Lambda (Status Update)
   DynamoDB (status: COMPLETED) ← Lambda (Status Update)
   Client Callback API ← Lambda (Status Update) [optional]

5. ERROR HANDLING PHASE
   Step Functions (failure) → SQS (retry) → Step Functions (retry)
   SQS (3rd failure) → DLQ → Lambda (DLQ Analyzer) → SNS → Alerts

6. CLEANUP PHASE
   EventBridge (daily schedule) → Lambda (Cleanup) → S3 (delete old documents)
```

---

## 4. Component Interaction Diagram

```
┌──────────────┐
│   Client     │
└──────┬───────┘
       │
       │ 1. POST /upload
       ▼
┌──────────────┐     2. Generate      ┌──────────────┐
│ API Gateway  │────presigned URL────▶│  Lambda:     │
└──────┬───────┘                      │  Presigned   │
       │                              │  URL Gen     │
       │ 3. Return URL                └──────┬───────┘
       │                                     │
       │                              4. Store metadata
       │                                     ▼
       │                              ┌──────────────┐
       │                              │  DynamoDB    │
       │                              └──────────────┘
       │
       │ 5. Upload document
       ▼
┌──────────────┐     6. S3 Event      ┌──────────────┐
│  S3 Input    │────────────────────▶│ EventBridge  │
└──────────────┘                      └──────┬───────┘
                                             │
                                      7. Send message
                                             ▼
                                      ┌──────────────┐
                                      │  SQS Queue   │
                                      └──────┬───────┘
                                             │
                                      8. Poll messages
                                             ▼
                                      ┌──────────────┐
                                      │  Lambda:     │
                                      │  SQS Poller  │
                                      └──────┬───────┘
                                             │
                                      9. Start execution
                                             ▼
┌─────────────────────────────────────────────────────────────────────────────────┐
│                           Step Functions Workflow                                │
│                                                                                  │
│  ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐ │
│  │   Lambda:    │───▶│   Lambda:    │───▶│   Lambda:    │───▶│   Lambda:    │ │
│  │   Language   │    │   Document   │    │   Schema     │    │   Status     │ │
│  │   Detector   │    │   Classifier │    │   Extractor  │    │   Updater    │ │
│  └──────┬───────┘    └──────┬───────┘    └──────┬───────┘    └──────┬───────┘ │
│         │                   │                   │                   │          │
└─────────┼───────────────────┼───────────────────┼───────────────────┼──────────┘
          │                   │                   │                   │
          ▼                   ▼                   ▼                   ▼
   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
   │ Comprehend/  │    │   Bedrock    │    │   Bedrock    │    │  DynamoDB +  │
   │   Bedrock    │    │     BDA      │    │     BDA      │    │  S3 Output   │
   └──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
```

---

## 5. Error Handling and Retry Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                         ERROR HANDLING FLOW                                      │
└─────────────────────────────────────────────────────────────────────────────────┘

Step Functions Execution
        │
        │ Lambda invocation
        ▼
┌──────────────────┐
│  Lambda Function │
└────────┬─────────┘
         │
         ├─ SUCCESS ──────────────────────────────────────────────┐
         │                                                         │
         └─ FAILURE                                               │
            │                                                      │
            ▼                                                      │
    ┌──────────────────┐                                          │
    │  Step Functions  │                                          │
    │  Retry Logic     │                                          │
    └────────┬─────────┘                                          │
             │                                                     │
             ├─ Attempt 1: Wait 2s → Retry                        │
             ├─ Attempt 2: Wait 4s → Retry                        │
             ├─ Attempt 3: Wait 8s → Retry                        │
             │                                                     │
             └─ All retries failed                                │
                │                                                  │
                ▼                                                  │
        ┌──────────────────┐                                      │
        │  Return error    │                                      │
        │  to SQS          │                                      │
        └────────┬─────────┘                                      │
                 │                                                 │
                 ▼                                                 │
        ┌──────────────────┐                                      │
        │  SQS increments  │                                      │
        │  receive count   │                                      │
        └────────┬─────────┘                                      │
                 │                                                 │
                 ├─ Receive count < 3: Message visible again      │
                 │  (after visibility timeout)                    │
                 │  └─ SQS Poller retries → Step Functions        │
                 │                                                 │
                 └─ Receive count >= 3                            │
                    │                                              │
                    ▼                                              │
            ┌──────────────────┐                                  │
            │  Move to DLQ     │                                  │
            └────────┬─────────┘                                  │
                     │                                             │
                     ▼                                             │
            ┌──────────────────┐                                  │
            │  CloudWatch      │                                  │
            │  Alarm triggered │                                  │
            └────────┬─────────┘                                  │
                     │                                             │
                     ▼                                             │
            ┌──────────────────┐                                  │
            │  Lambda: DLQ     │                                  │
            │  Analyzer        │                                  │
            └────────┬─────────┘                                  │
                     │                                             │
                     ▼                                             │
            ┌──────────────────┐                                  │
            │  SNS: Send       │                                  │
            │  detailed alert  │                                  │
            └────────┬─────────┘                                  │
                     │                                             │
                     ▼                                             │
            ┌──────────────────┐                                  │
            │  New Relic       │                                  │
            │  Email, Slack    │                                  │
            └──────────────────┘                                  │
                                                                   │
                                                                   │
                                                                   ▼
                                                          ┌──────────────────┐
                                                          │  Continue to     │
                                                          │  next state      │
                                                          └──────────────────┘
```

---

**End of Architecture Diagrams**
