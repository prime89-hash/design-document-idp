# IDP Solution - Final Architecture Summary

## Document Control
- **Version**: 2.0 (Updated)
- **Last Updated**: January 14, 2026
- **Status**: Final Recommendation

---

## Executive Summary

After thorough analysis, the **recommended IDP architecture is simplified** to focus on essential components with optional SQS integration only if traffic patterns require it.

---

## Recommended Architecture

### **Phase 1: Start Simple** ✅ RECOMMENDED

```
┌─────────────────────────────────────────────────────────────────┐
│  Client Applications                                             │
└────────────────┬────────────────────────────────────────────────┘
                 │ HTTPS
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  API Gateway                                                     │
│  POST /upload → Generate Pre-signed URL                         │
│  GET /status/{id} → Get Status                                  │
└────────────────┬────────────────────────────────────────────────┘
                 │
    ┌────────────┴────────────┐
    │                         │
    ▼                         ▼
┌──────────┐          ┌──────────────┐
│ Lambda:  │─────────▶│  DynamoDB:   │
│ Presigned│          │  Metadata    │
│ URL Gen  │          └──────────────┘
└──────────┘
    │
    │ Returns Pre-signed URL
    ▼
┌─────────────────────────────────────────────────────────────────┐
│  S3: Input Documents                                             │
└────────────────┬────────────────────────────────────────────────┘
                 │ S3 Event
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  EventBridge Rule                                                │
│  Event Pattern: S3 Object Created                               │
└────────────────┬────────────────────────────────────────────────┘
                 │ Trigger (direct)
                 ▼
┌─────────────────────────────────────────────────────────────────┐
│  Step Functions: Document Processing                             │
│                                                                  │
│  1. BDA Orchestrator Lambda                                     │
│     └─ BDA (language + classification + extraction)            │
│  2. Confidence Check                                            │
│  3. Output Processor Lambda                                     │
│  4. Status Updater Lambda                                       │
│                                                                  │
│  Retry: 3 attempts with exponential backoff (2s, 4s, 8s)       │
└────────────────┬────────────────────────────────────────────────┘
                 │
    ┌────────────┼────────────┐
    │            │            │
    ▼            ▼            ▼
┌─────────┐  ┌─────────┐  ┌─────────┐
│ Bedrock │  │DynamoDB │  │S3 Output│
│   BDA   │  │ Update  │  │  Bucket │
└─────────┘  └─────────┘  └─────────┘
```

---

## Component Summary

### Lambda Functions: 5

1. **Presigned URL Generator**
   - Purpose: Generate S3 upload URLs
   - VPC: No
   - Memory: 512 MB
   - Timeout: 30s

2. **BDA Orchestrator**
   - Purpose: Complete document processing via BDA
   - VPC: Yes (private subnet)
   - Memory: 1024 MB
   - Timeout: 300s
   - **Does**: Language detection, classification, extraction (all-in-one)

3. **Output Processor**
   - Purpose: Validate BDA results and save to S3
   - VPC: Yes (private subnet)
   - Memory: 512 MB
   - Timeout: 60s

4. **Status Updater**
   - Purpose: Update DynamoDB and trigger callbacks
   - VPC: No
   - Memory: 256 MB
   - Timeout: 30s

5. **Document Cleanup**
   - Purpose: Delete old documents (scheduled daily)
   - VPC: No
   - Memory: 256 MB
   - Timeout: 60s

### AWS Services: ~25 Resources

- **Storage**: 4 S3 Buckets, 2 DynamoDB Tables
- **Compute**: 5 Lambda Functions
- **Orchestration**: 1 Step Functions, 1 EventBridge Rule
- **API**: 1 API Gateway
- **AI/ML**: 1 BDA Project
- **Monitoring**: CloudWatch, X-Ray, SNS
- **Container**: 5 ECR Repositories
- **VPC**: Existing (provided by Evoke)

---

## Key Simplifications

### Removed Components:
- ❌ SQS Queue (optional, add only if needed)
- ❌ SQS Poller Lambda
- ❌ Dead Letter Queue (optional)
- ❌ DLQ Analyzer Lambda (optional)
- ❌ Language Detector Lambda (BDA does this)
- ❌ Document Classifier Lambda (BDA does this)
- ❌ Schema Extractor Lambda (BDA does this)

### Result:
- **Before**: 9 Lambda functions, ~42 resources
- **After**: 5 Lambda functions, ~25 resources
- **Reduction**: 44% fewer Lambda functions, 40% fewer resources

---

## Cost Comparison

### Per 1 Million Documents:

| Component | Without SQS | With SQS |
|-----------|-------------|----------|
| Lambda | $20 | $20 |
| BDA | $150 | $150 |
| S3 | $25 | $25 |
| DynamoDB | $30 | $30 |
| SQS | $0 | $0.40 |
| Step Functions | $25 | $25 |
| CloudWatch | $20 | $20 |
| **Total** | **$270** | **$270.40** |

**Savings without SQS**: Minimal cost difference, but simpler architecture

---

## When to Add SQS

### Add SQS Only If:

1. **Traffic Spikes**
   - Upload rate exceeds 500 documents/minute
   - Step Functions throttling occurs
   - Need to buffer messages

2. **Rate Limiting**
   - Need to control processing rate
   - Want to limit concurrent BDA calls
   - Cost control required

3. **Advanced Retry**
   - Need more than 3 retry attempts
   - Want longer retry intervals (minutes vs seconds)

4. **Message Persistence**
   - Compliance requires message durability
   - Need audit trail of all messages

### How to Add SQS:

Use **SQS → Step Functions (Direct Integration)**:
```
S3 → EventBridge → SQS → Step Functions (native)
```

**NOT**:
```
S3 → EventBridge → SQS → Lambda Poller → Step Functions
```

---

## Error Handling

### Without SQS:

```
Step Functions Retry:
- Attempt 1: Wait 2s → Retry
- Attempt 2: Wait 4s → Retry
- Attempt 3: Wait 8s → Retry
- If all fail: Mark as FAILED

CloudWatch Alarm:
- Triggers on ExecutionsFailed > 5
- Sends alert to SNS → New Relic/Email

EventBridge DLQ (optional):
- Captures events that fail to trigger Step Functions
- Separate from Step Functions failures
```

### With SQS (if added):

```
Step Functions Retry:
- 3 attempts with exponential backoff

SQS Retry:
- Message becomes visible after 15 minutes
- 3 attempts total

DLQ:
- After 3 SQS retries, move to DLQ
- DLQ Analyzer sends detailed alert

Total: Up to 9 retry attempts
```

---

## Monitoring

### Key Metrics:

**Without SQS:**
- Step Functions: ExecutionsStarted, ExecutionsFailed, ExecutionThrottled
- Lambda: Invocations, Errors, Duration, Throttles
- BDA: Job completion time, confidence scores
- API Gateway: Request count, 4xx, 5xx errors

**With SQS (if added):**
- All above metrics, plus:
- SQS: Queue depth, message age, messages in flight
- DLQ: Message count

### Critical Alarms:

1. **Step Functions Failed** > 5 (5 min period)
2. **Step Functions Throttled** > 10 (5 min period) → Consider adding SQS
3. **Lambda Error Rate** > 5%
4. **API Gateway 5xx** > 50
5. **EventBridge DLQ Messages** > 0 (if configured)

---

## Migration Path

### If You Have SQS Currently:

**Option A: Keep SQS (if traffic requires it)**
- Remove Lambda Poller
- Use SQS → Step Functions direct integration
- Keep DLQ

**Option B: Remove SQS (simplify)**
- Remove SQS Queue
- Remove Lambda Poller
- Remove DLQ
- Use EventBridge → Step Functions direct
- Monitor for throttling

### Recommendation:

Start with **Option B** (no SQS), monitor for 2-4 weeks:
- If no throttling: Keep simple architecture
- If throttling occurs: Add SQS with direct integration

---

## Performance Targets

| Metric | Target | Notes |
|--------|--------|-------|
| **Throughput** | 100 docs/min | Without SQS |
| **Throughput** | 500+ docs/min | With SQS buffering |
| **API Response** | < 500ms | Presigned URL generation |
| **Processing Time** | 8-12s | BDA processing (average) |
| **End-to-End** | < 15s | Upload to completion |
| **Classification Accuracy** | > 95% | BDA confidence |
| **Extraction Accuracy** | > 90% | BDA confidence |
| **Availability** | 99.9% | Step Functions SLA |

---

## Deployment Checklist

### Infrastructure:
- [ ] VPC and subnets (provided by Evoke)
- [ ] 5 Lambda functions deployed
- [ ] Step Functions state machine created
- [ ] EventBridge rule configured
- [ ] API Gateway endpoints created
- [ ] S3 buckets created
- [ ] DynamoDB tables created
- [ ] BDA project configured
- [ ] CloudWatch alarms set up
- [ ] IAM roles configured

### Optional (SQS):
- [ ] SQS queue created (if needed)
- [ ] DLQ created (if needed)
- [ ] Step Functions updated for SQS integration
- [ ] SQS alarms configured

### Testing:
- [ ] Upload test document
- [ ] Verify EventBridge triggers Step Functions
- [ ] Verify BDA processing completes
- [ ] Verify output saved to S3
- [ ] Verify DynamoDB updated
- [ ] Verify callback triggered (if configured)
- [ ] Test failure scenarios
- [ ] Verify CloudWatch alarms

---

## Summary

### Recommended Approach:

1. **Start Simple**: EventBridge → Step Functions (no SQS)
   - 5 Lambda functions
   - ~25 AWS resources
   - ~$270/1M documents
   - Sufficient for most use cases

2. **Monitor**: Watch for throttling and traffic patterns

3. **Add SQS if needed**: Use direct Step Functions integration
   - Still 5 Lambda functions
   - ~28 AWS resources
   - ~$270.40/1M documents
   - Better for high traffic

4. **Never add SQS "just in case"**: YAGNI principle

### Key Benefits:

- ✅ **40% fewer resources** (25 vs 42)
- ✅ **44% fewer Lambda functions** (5 vs 9)
- ✅ **Simpler to maintain**
- ✅ **Lower cost**
- ✅ **Faster processing**
- ✅ **Still highly reliable**
- ✅ **Easy to add SQS later if needed**

---

## References

- [SQS-DECISION-GUIDE.md](SQS-DECISION-GUIDE.md) - Detailed SQS decision guide
- [BDA-ONLY-ARCHITECTURE.md](BDA-ONLY-ARCHITECTURE.md) - BDA handles all processing
- [COMPLETE-INFRA-DIAGRAM.md](COMPLETE-INFRA-DIAGRAM.md) - Detailed infrastructure
- [00-Architecture-Diagrams.md](00-Architecture-Diagrams.md) - Visual diagrams

---

**Document Status**: Final Recommendation  
**Last Updated**: January 14, 2026  
**Architecture**: Simplified (EventBridge → Step Functions)  
**SQS**: Optional (add only if traffic patterns require it)
