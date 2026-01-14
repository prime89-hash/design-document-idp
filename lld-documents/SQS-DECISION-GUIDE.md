# SQS Integration Decision Guide
## When and How to Use SQS in IDP Architecture

### Document Control
- **Version**: 1.0
- **Last Updated**: January 14, 2026
- **Purpose**: Guide for deciding whether to use SQS and which integration pattern to use

---

## Recommended Architecture Approach

### **Phase 1: Start Simple (RECOMMENDED)** ✅

```
S3 → EventBridge → Step Functions → Processing
```

**Components:**
- No SQS
- No SQS Poller Lambda
- No DLQ
- Direct EventBridge to Step Functions integration

**Lambda Functions: 5**
1. Presigned URL Generator
2. BDA Orchestrator
3. Output Processor
4. Status Updater
5. Document Cleanup

**Benefits:**
- ✅ Simplest architecture
- ✅ Lowest cost (~$25/1M documents)
- ✅ Fastest processing (no queue delay)
- ✅ Easiest to maintain
- ✅ Sufficient for most use cases

**When This Works:**
- Document upload rate: < 500 documents/minute
- Traffic is relatively steady
- Step Functions concurrency (1000) is sufficient
- 3 retries with exponential backoff is enough

---

### **Phase 2: Add SQS If Needed** (Optional)

```
S3 → EventBridge → SQS → Step Functions → Processing
```

**Add SQS only if you experience:**
- Traffic spikes causing Step Functions throttling
- Need for rate limiting/cost control
- Requirement for message persistence beyond Step Functions
- Need for more than 3 retry attempts

**Components Added:**
- SQS Queue
- SQS DLQ
- (No Lambda Poller - use native Step Functions integration)

**Lambda Functions: Still 5** (no change)

**Benefits:**
- ✅ Message buffering
- ✅ Rate limiting
- ✅ Advanced retry logic
- ✅ DLQ for failed messages
- ✅ Still relatively simple

**Cost:** ~$25.40/1M documents (+$0.40 for SQS)

---

## Three Integration Options (If SQS is Needed)

### Option 1: SQS → Lambda → Step Functions ❌ NOT RECOMMENDED

```
EventBridge → SQS → Lambda (Poller) → Step Functions
```

**Pros:** Full control, custom logic
**Cons:** Extra Lambda, higher cost, more complexity
**Cost:** ~$27.40/1M documents
**Use When:** Need complex message transformation

---

### Option 2: SQS → Step Functions (Direct) ✅ RECOMMENDED IF USING SQS

```
EventBridge → SQS → Step Functions (native integration)
```

**Pros:** Simple, native AWS, no extra Lambda
**Cons:** Less flexibility
**Cost:** ~$25.40/1M documents
**Use When:** Need SQS benefits without complexity

---

### Option 3: EventBridge → Step Functions ✅ RECOMMENDED TO START

```
EventBridge → Step Functions (direct)
```

**Pros:** Simplest, cheapest, fastest
**Cons:** No buffering, no rate limiting
**Cost:** ~$25/1M documents
**Use When:** Standard traffic patterns

---

## Decision Tree

```
START: Do you need SQS?
    │
    ├─ NO → Use Option 3 (EventBridge → Step Functions) ✅
    │        Start here for all new deployments
    │
    └─ YES → Why do you need it?
         │
         ├─ Traffic spikes > 500 docs/min → Use Option 2 (SQS → SF Direct) ✅
         │
         ├─ Need rate limiting → Use Option 2 (SQS → SF Direct) ✅
         │
         ├─ Need custom message logic → Use Option 1 (SQS → Lambda → SF)
         │
         └─ "Just in case" → DON'T USE SQS ❌
                             Start with Option 3, add later if needed
```

---

## Migration Path

### Current State → Recommended State

**If you have SQS + Lambda Poller:**
```
Current:  S3 → EB → SQS → Lambda → Step Functions
                              ↓
Remove:   Lambda Poller
                              ↓
New:      S3 → EB → SQS → Step Functions (direct)
```

**If you want simplest:**
```
Current:  S3 → EB → SQS → Lambda → Step Functions
                              ↓
Remove:   SQS + Lambda Poller
                              ↓
New:      S3 → EB → Step Functions (direct)
```

---

## Implementation Guide

### Recommended: EventBridge → Step Functions (No SQS)

**EventBridge Rule:**
```hcl
resource "aws_cloudwatch_event_rule" "s3_to_stepfunctions" {
  name = "idp-s3-to-stepfunctions-${var.environment}"
  
  event_pattern = jsonencode({
    source      = ["aws.s3"]
    detail-type = ["Object Created"]
    detail = {
      bucket = {
        name = [aws_s3_bucket.input_documents.id]
      }
    }
  })
  
  # Retry configuration
  retry_policy {
    maximum_retry_attempts = 3
    maximum_event_age      = 3600
  }
  
  # Optional: Dead letter queue for EventBridge
  dead_letter_config {
    arn = aws_sqs_queue.eventbridge_dlq.arn
  }
}

resource "aws_cloudwatch_event_target" "stepfunctions" {
  rule      = aws_cloudwatch_event_rule.s3_to_stepfunctions.name
  target_id = "TriggerStepFunctions"
  arn       = aws_sfn_state_machine.document_processing.arn
  role_arn  = aws_iam_role.eventbridge_stepfunctions.arn
  
  # Transform S3 event to Step Functions input
  input_transformer {
    input_paths = {
      bucket = "$.detail.bucket.name"
      key    = "$.detail.object.key"
    }
    
    input_template = jsonencode({
      document_id = "<key>"
      s3_bucket   = "<bucket>"
      s3_key      = "<key>"
    })
  }
  
  # Retry configuration
  retry_policy {
    maximum_retry_attempts       = 3
    maximum_event_age_in_seconds = 3600
  }
}
```

**Step Functions State Machine:**
```json
{
  "Comment": "IDP Document Processing - Direct EventBridge Integration",
  "StartAt": "BDAOrchestrator",
  "States": {
    
    "BDAOrchestrator": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-bda-orchestrator-env",
      "ResultPath": "$.bdaResult",
      "Retry": [
        {
          "ErrorEquals": ["States.TaskFailed"],
          "IntervalSeconds": 2,
          "MaxAttempts": 3,
          "BackoffRate": 2.0
        }
      ],
      "Catch": [
        {
          "ErrorEquals": ["States.ALL"],
          "ResultPath": "$.error",
          "Next": "HandleError"
        }
      ],
      "Next": "CheckConfidence"
    },
    
    "CheckConfidence": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.bdaResult.classification_confidence",
          "NumericGreaterThanEquals": 0.85,
          "Next": "OutputProcessor"
        }
      ],
      "Default": "LowConfidence"
    },
    
    "OutputProcessor": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-output-processor-env",
      "ResultPath": "$.outputResult",
      "Next": "UpdateStatus"
    },
    
    "UpdateStatus": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-status-updater-env",
      "End": true
    },
    
    "LowConfidence": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-status-updater-env",
      "Parameters": {
        "document_id.$": "$.document_id",
        "status": "LOW_CONFIDENCE"
      },
      "End": true
    },
    
    "HandleError": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-status-updater-env",
      "Parameters": {
        "document_id.$": "$.document_id",
        "status": "FAILED",
        "error_message.$": "$.error.Cause"
      },
      "End": true
    }
  }
}
```

---

## Monitoring Without SQS

**CloudWatch Alarms:**
```hcl
# Step Functions Failed Executions
resource "aws_cloudwatch_metric_alarm" "stepfunctions_failed" {
  alarm_name          = "idp-stepfunctions-failed-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ExecutionsFailed"
  namespace           = "AWS/States"
  period              = 300
  statistic           = "Sum"
  threshold           = 5
  alarm_description   = "Alert when Step Functions executions fail"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    StateMachineArn = aws_sfn_state_machine.document_processing.arn
  }
}

# Step Functions Throttled
resource "aws_cloudwatch_metric_alarm" "stepfunctions_throttled" {
  alarm_name          = "idp-stepfunctions-throttled-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 2
  metric_name         = "ExecutionThrottled"
  namespace           = "AWS/States"
  period              = 300
  statistic           = "Sum"
  threshold           = 10
  alarm_description   = "Alert when Step Functions is throttled - consider adding SQS"
  alarm_actions       = [aws_sns_topic.alerts.arn]
  
  dimensions = {
    StateMachineArn = aws_sfn_state_machine.document_processing.arn
  }
}

# EventBridge DLQ Messages (if configured)
resource "aws_cloudwatch_metric_alarm" "eventbridge_dlq" {
  alarm_name          = "idp-eventbridge-dlq-${var.environment}"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = 1
  metric_name         = "ApproximateNumberOfMessagesVisible"
  namespace           = "AWS/SQS"
  period              = 60
  statistic           = "Average"
  threshold           = 0
  alarm_description   = "CRITICAL: Events failed to trigger Step Functions"
  alarm_actions       = [aws_sns_topic.critical_alerts.arn]
  
  dimensions = {
    QueueName = aws_sqs_queue.eventbridge_dlq.name
  }
}
```

---

## Summary

### Recommended Approach

1. **Start with EventBridge → Step Functions** (No SQS)
   - Simplest, cheapest, fastest
   - 5 Lambda functions
   - ~$25/1M documents

2. **Monitor for:**
   - Step Functions throttling
   - High failure rates
   - Traffic spikes

3. **Add SQS if needed:**
   - Use SQS → Step Functions (direct integration)
   - No Lambda Poller
   - Still 5 Lambda functions
   - ~$25.40/1M documents

4. **Never add SQS "just in case"**
   - YAGNI principle
   - Add complexity only when proven necessary

---

**Document Status**: Complete  
**Last Updated**: January 14, 2026  
**Recommendation**: Start without SQS, add only if needed
