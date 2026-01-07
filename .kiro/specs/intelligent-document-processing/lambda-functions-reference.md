# Lambda Functions and Step Functions Orchestration Reference

## Overview

This document provides detailed specifications for all Lambda functions and Step Functions components used in the Intelligent Document Processing (IDP) system. The system uses 7 Lambda functions orchestrated by AWS Step Functions to process documents through a complete workflow.

## Lambda Functions Detailed Specifications

### 1. Document Validator Lambda

**Function Name**: `idp-document-validator-{environment}`

**Purpose**: Validates uploaded documents for format, size, and accessibility before processing

**Runtime Configuration**:
- **Runtime**: Python 3.11
- **Memory**: 256 MB
- **Timeout**: 60 seconds
- **Handler**: `validator.handler`

**Input Parameters**:
```json
{
  "document_id": "uuid-string",
  "s3_bucket": "bucket-name",
  "s3_key": "document-path/filename.pdf",
  "client_id": "client-identifier",
  "callback_url": "https://client.com/callback"
}
```

**Validation Checks**:
- **File Format**: PDF, JPG, JPEG, TIFF only
- **File Size**: Maximum 10MB
- **File Accessibility**: S3 object exists and readable
- **Metadata Completeness**: Required fields present
- **File Integrity**: Basic file structure validation

**Output**:
```json
{
  "document_id": "uuid-string",
  "validation_status": "success|failed",
  "file_metadata": {
    "size_bytes": 1024000,
    "format": "pdf",
    "pages": 2,
    "created_date": "2024-01-07T10:00:00Z"
  },
  "validation_errors": ["array of error messages if any"]
}
```

**IAM Permissions Required**:
- `s3:GetObject` on input bucket
- `s3:GetObjectMetadata` on input bucket
- `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:UpdateItem` on metadata table
- `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents`

### 2. BDA Processor Lambda

**Function Name**: `idp-bda-processor-{environment}`

**Purpose**: Handles all document processing through Bedrock Data Automation including OCR, classification, and extraction

**Runtime Configuration**:
- **Runtime**: Python 3.11
- **Memory**: 1024 MB
- **Timeout**: 300 seconds (5 minutes)
- **Handler**: `bda_processor.handler`

**Input Parameters**:
```json
{
  "document_id": "uuid-string",
  "s3_bucket": "bucket-name",
  "s3_key": "document-path/filename.pdf",
  "validation_result": {
    "file_metadata": "object from validator"
  }
}
```

**Processing Steps**:
1. **Load Configuration**: Retrieve BDA blueprint from S3 config bucket
2. **Prepare BDA Request**: Format document reference and processing parameters
3. **Call BDA API**: Single API call for complete processing
4. **Parse Results**: Extract classification, field data, and confidence scores
5. **Update Metadata**: Store results in DynamoDB

**Output**:
```json
{
  "document_id": "uuid-string",
  "processing_status": "success|failed|partial",
  "language_detected": "en",
  "language_confidence": 0.98,
  "classification_result": {
    "document_type": "national_id",
    "confidence_score": 0.92,
    "classification_timestamp": "2024-01-07T10:05:00Z"
  },
  "extraction_result": {
    "extracted_fields": {
      "full_name": {
        "value": "John Doe",
        "confidence": 0.95
      },
      "document_number": {
        "value": "ID123456789",
        "confidence": 0.98
      },
      "expiry_date": {
        "value": "2030-12-31",
        "confidence": 0.89
      }
    },
    "extraction_timestamp": "2024-01-07T10:05:30Z"
  },
  "processing_duration_ms": 45000
}
```

**IAM Permissions Required**:
- `bedrock:InvokeModel` on all foundation models
- `bedrock-agent:*` for BDA operations
- `s3:GetObject` on input and config buckets
- `dynamodb:GetItem`, `dynamodb:PutItem`, `dynamodb:UpdateItem` on metadata table
- `secretsmanager:GetSecretValue` for API credentials

### 3. Quality Evaluator Lambda

**Function Name**: `idp-quality-evaluator-{environment}`

**Purpose**: Performs LLM-based quality assessment of extraction results against ground truth or business rules

**Runtime Configuration**:
- **Runtime**: Python 3.11
- **Memory**: 512 MB
- **Timeout**: 180 seconds (3 minutes)
- **Handler**: `evaluator.handler`

**Input Parameters**:
```json
{
  "document_id": "uuid-string",
  "bda_result": {
    "classification_result": "object",
    "extraction_result": "object"
  },
  "evaluation_criteria": {
    "document_type": "national_id",
    "required_fields": ["full_name", "document_number", "expiry_date"],
    "quality_threshold": 0.85
  }
}
```

**Evaluation Process**:
1. **Load Evaluation Model**: Initialize Claude Haiku for evaluation
2. **Prepare Evaluation Prompt**: Format extraction results for LLM assessment
3. **Execute Evaluation**: Call Bedrock with evaluation prompt
4. **Parse Evaluation Results**: Extract quality scores and feedback
5. **Generate Report**: Create detailed evaluation report

**Output**:
```json
{
  "document_id": "uuid-string",
  "evaluation_status": "completed",
  "quality_metrics": {
    "overall_score": 0.91,
    "classification_accuracy": 0.92,
    "field_precision": 0.94,
    "field_recall": 0.88,
    "confidence_compliance": true
  },
  "field_evaluations": {
    "full_name": {
      "accuracy_score": 0.95,
      "completeness": true,
      "format_valid": true
    },
    "document_number": {
      "accuracy_score": 0.98,
      "completeness": true,
      "format_valid": true
    }
  },
  "recommendations": ["array of improvement suggestions"],
  "evaluation_timestamp": "2024-01-07T10:06:00Z"
}
```

**IAM Permissions Required**:
- `bedrock:InvokeModel` for evaluation model
- `s3:GetObject` on config bucket for evaluation prompts
- `dynamodb:UpdateItem` on metadata table

### 4. Output Generator Lambda

**Function Name**: `idp-output-generator-{environment}`

**Purpose**: Creates final structured JSON output and stores it in the output S3 bucket

**Runtime Configuration**:
- **Runtime**: Python 3.11
- **Memory**: 256 MB
- **Timeout**: 60 seconds
- **Handler**: `output_generator.handler`

**Input Parameters**:
```json
{
  "document_id": "uuid-string",
  "bda_result": "object from BDA processor",
  "quality_evaluation": "object from quality evaluator"
}
```

**Output Generation Process**:
1. **Merge Results**: Combine BDA and evaluation results
2. **Format Output**: Structure data according to client requirements
3. **Add Metadata**: Include processing timestamps and system information
4. **Store Output**: Save JSON to S3 output bucket
5. **Update Status**: Mark document as completed in DynamoDB

**Final Output Format**:
```json
{
  "document_id": "uuid-string",
  "processing_metadata": {
    "processed_at": "2024-01-07T10:06:30Z",
    "processing_duration_total_ms": 90000,
    "system_version": "1.0.0"
  },
  "document_classification": {
    "type": "national_id",
    "confidence": 0.92,
    "language": "en"
  },
  "extracted_data": {
    "full_name": "John Doe",
    "document_number": "ID123456789",
    "expiry_date": "2030-12-31"
  },
  "confidence_scores": {
    "full_name": 0.95,
    "document_number": 0.98,
    "expiry_date": 0.89
  },
  "quality_assessment": {
    "overall_score": 0.91,
    "meets_threshold": true,
    "manual_review_required": false
  }
}
```

**IAM Permissions Required**:
- `s3:PutObject` on output bucket
- `dynamodb:UpdateItem` on metadata table

### 5. Cleanup Handler Lambda

**Function Name**: `idp-cleanup-handler-{environment}`

**Purpose**: Manages document lifecycle by cleaning up temporary files and updating processing status

**Runtime Configuration**:
- **Runtime**: Python 3.11
- **Memory**: 256 MB
- **Timeout**: 60 seconds
- **Handler**: `cleanup_handler.handler`

**Input Parameters**:
```json
{
  "document_id": "uuid-string",
  "s3_bucket": "input-bucket-name",
  "s3_key": "document-path/filename.pdf",
  "cleanup_options": {
    "delete_input_document": true,
    "retain_logs": true,
    "cleanup_temp_files": true
  }
}
```

**Cleanup Operations**:
1. **Delete Input Document**: Remove original file from S3 input bucket
2. **Clean Temporary Files**: Remove any intermediate processing files
3. **Update Metadata**: Mark document as cleaned up in DynamoDB
4. **Audit Logging**: Log cleanup operations for compliance

**Output**:
```json
{
  "document_id": "uuid-string",
  "cleanup_status": "completed",
  "operations_performed": [
    "deleted_input_document",
    "cleaned_temp_files",
    "updated_metadata"
  ],
  "cleanup_timestamp": "2024-01-07T10:07:00Z"
}
```

**IAM Permissions Required**:
- `s3:DeleteObject` on input bucket
- `s3:ListBucket` on input bucket
- `dynamodb:UpdateItem` on metadata table

### 6. Notification Handler Lambda

**Function Name**: `idp-notification-handler-{environment}`

**Purpose**: Sends completion, error, or status notifications to client applications

**Runtime Configuration**:
- **Runtime**: Python 3.11
- **Memory**: 256 MB
- **Timeout**: 30 seconds
- **Handler**: `notification_handler.handler`

**Input Parameters**:
```json
{
  "document_id": "uuid-string",
  "status": "completed|failed|manual_review_required",
  "callback_url": "https://client.com/callback",
  "notification_data": {
    "processing_result": "object or error details"
  }
}
```

**Notification Types**:
1. **Success Notification**: Document processed successfully
2. **Error Notification**: Processing failed with error details
3. **Manual Review Notification**: Low confidence, requires human review
4. **Status Update**: Intermediate processing status updates

**Notification Payload**:
```json
{
  "document_id": "uuid-string",
  "status": "completed",
  "timestamp": "2024-01-07T10:07:30Z",
  "result_location": "s3://output-bucket/results/uuid-string.json",
  "processing_summary": {
    "document_type": "national_id",
    "fields_extracted": 3,
    "quality_score": 0.91,
    "processing_time_ms": 90000
  }
}
```

**IAM Permissions Required**:
- `dynamodb:GetItem` on metadata table
- `s3:GetObject` on output bucket (for result URLs)
- Network access for HTTP callbacks

### 7. Error Handler Lambda

**Function Name**: `idp-error-handler-{environment}`

**Purpose**: Processes and logs error conditions, manages retry logic, and routes failed documents

**Runtime Configuration**:
- **Runtime**: Python 3.11
- **Memory**: 256 MB
- **Timeout**: 60 seconds
- **Handler**: `error_handler.handler`

**Input Parameters**:
```json
{
  "document_id": "uuid-string",
  "error_type": "validation_error|processing_error|timeout_error",
  "error_details": {
    "error_message": "string",
    "error_code": "string",
    "stack_trace": "string",
    "retry_count": 2
  },
  "original_input": "object from failed step"
}
```

**Error Handling Operations**:
1. **Error Classification**: Categorize error type and severity
2. **Retry Logic**: Determine if retry is appropriate
3. **Dead Letter Queue**: Route to DLQ if max retries exceeded
4. **Error Logging**: Log detailed error information
5. **Client Notification**: Inform client of processing failure

**Output**:
```json
{
  "document_id": "uuid-string",
  "error_handling_result": "retried|sent_to_dlq|manual_intervention",
  "error_classification": {
    "type": "processing_error",
    "severity": "high",
    "recoverable": false
  },
  "next_action": "send_to_manual_review",
  "error_timestamp": "2024-01-07T10:08:00Z"
}
```

**IAM Permissions Required**:
- `dynamodb:UpdateItem` on metadata table
- `sqs:SendMessage` to dead letter queue
- `logs:PutLogEvents` for error logging## Step
 Functions Orchestration Details

### State Machine Configuration

**State Machine Name**: `idp-document-processing-{environment}`

**Execution Role**: `idp-step-functions-role-{environment}`

**Logging Configuration**:
- **Log Level**: ALL
- **Include Execution Data**: true
- **Log Destination**: CloudWatch Log Group

### State Machine Flow

#### 1. ValidateDocument State

**Type**: Task (Lambda Invoke)

**Configuration**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "idp-document-validator-{env}",
    "Payload": {
      "document_id.$": "$.document_id",
      "s3_bucket.$": "$.s3_bucket",
      "s3_key.$": "$.s3_key"
    }
  }
}
```

**Retry Configuration**:
- **Error Types**: Lambda.ServiceException, Lambda.AWSLambdaException
- **Interval**: 2 seconds
- **Max Attempts**: 3
- **Backoff Rate**: 2.0

**Next State**: ProcessWithBDA (on success) | HandleValidationError (on failure)

#### 2. ProcessWithBDA State

**Type**: Task (Lambda Invoke)

**Configuration**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "TimeoutSeconds": 300,
  "Parameters": {
    "FunctionName": "idp-bda-processor-{env}",
    "Payload": {
      "document_id.$": "$.document_id",
      "s3_bucket.$": "$.s3_bucket",
      "s3_key.$": "$.s3_key",
      "validation_result.$": "$.validation_result"
    }
  }
}
```

**Retry Configuration**:
- **Error Types**: Lambda.ServiceException, Lambda.AWSLambdaException
- **Interval**: 5 seconds
- **Max Attempts**: 3
- **Backoff Rate**: 2.0

**Next State**: CheckConfidenceThreshold

#### 3. CheckConfidenceThreshold State

**Type**: Choice

**Configuration**:
```json
{
  "Type": "Choice",
  "Choices": [
    {
      "Variable": "$.bda_result.classification_confidence",
      "NumericGreaterThanEquals": 0.85,
      "Next": "EvaluateQuality"
    }
  ],
  "Default": "SendToManualReview"
}
```

**Decision Logic**:
- If confidence ≥ 0.85: Continue to quality evaluation
- If confidence < 0.85: Route to manual review queue

#### 4. EvaluateQuality State

**Type**: Task (Lambda Invoke)

**Configuration**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "idp-quality-evaluator-{env}",
    "Payload": {
      "document_id.$": "$.document_id",
      "bda_result.$": "$.bda_result"
    }
  }
}
```

**Retry Configuration**:
- **Error Types**: Lambda.ServiceException
- **Interval**: 2 seconds
- **Max Attempts**: 2
- **Backoff Rate**: 2.0

**Next State**: GenerateOutput

#### 5. GenerateOutput State

**Type**: Task (Lambda Invoke)

**Configuration**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "idp-output-generator-{env}",
    "Payload": {
      "document_id.$": "$.document_id",
      "bda_result.$": "$.bda_result",
      "quality_evaluation.$": "$.quality_evaluation"
    }
  }
}
```

**Next State**: CleanupResources

#### 6. CleanupResources State

**Type**: Task (Lambda Invoke)

**Configuration**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "idp-cleanup-handler-{env}",
    "Payload": {
      "document_id.$": "$.document_id",
      "s3_bucket.$": "$.s3_bucket",
      "s3_key.$": "$.s3_key"
    }
  }
}
```

**Next State**: NotifyCompletion

#### 7. NotifyCompletion State

**Type**: Task (Lambda Invoke)

**Configuration**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "idp-notification-handler-{env}",
    "Payload": {
      "document_id.$": "$.document_id",
      "status": "completed",
      "callback_url.$": "$.callback_url"
    }
  }
}
```

**End State**: true

#### 8. SendToManualReview State

**Type**: Task (SQS Send Message)

**Configuration**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::sqs:sendMessage",
  "Parameters": {
    "QueueUrl": "https://sqs.{region}.amazonaws.com/{account}/idp-manual-review-queue",
    "MessageBody": {
      "document_id.$": "$.document_id",
      "reason": "low_confidence",
      "bda_result.$": "$.bda_result"
    }
  }
}
```

**Next State**: NotifyManualReview

#### 9. Error Handling States

**HandleValidationError State**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "idp-error-handler-{env}",
    "Payload": {
      "document_id.$": "$.document_id",
      "error_type": "validation_error",
      "error_details.$": "$.error"
    }
  },
  "Next": "NotifyError"
}
```

**HandleProcessingError State**:
```json
{
  "Type": "Task",
  "Resource": "arn:aws:states:::lambda:invoke",
  "Parameters": {
    "FunctionName": "idp-error-handler-{env}",
    "Payload": {
      "document_id.$": "$.document_id",
      "error_type": "processing_error",
      "error_details.$": "$.error"
    }
  },
  "Next": "NotifyError"
}
```

## Integration Components

### EventBridge Integration

**Rule Name**: `idp-s3-document-upload-{environment}`

**Event Pattern**:
```json
{
  "source": ["aws.s3"],
  "detail-type": ["Object Created"],
  "detail": {
    "bucket": {
      "name": ["idp-input-documents-{environment}"]
    }
  }
}
```

**Target Configuration**:
- **Target**: Step Functions State Machine
- **Role**: EventBridge execution role
- **Input Transformer**: Converts S3 event to Step Functions input

### SQS Queues

#### Manual Review Queue
- **Queue Name**: `idp-manual-review-{environment}`
- **Visibility Timeout**: 300 seconds
- **Message Retention**: 14 days
- **Dead Letter Queue**: Configured after 3 receive attempts

#### Dead Letter Queue
- **Queue Name**: `idp-dead-letter-{environment}`
- **Message Retention**: 14 days
- **Monitoring**: CloudWatch alarms for message count

### DynamoDB Table Schema

**Table Name**: `idp-document-metadata-{environment}`

**Primary Key**: `document_id` (String)

**Global Secondary Index**: `status-index`
- **Partition Key**: `processing_status` (String)
- **Sort Key**: `upload_timestamp` (String)

**Attributes**:
- `document_id`: Unique identifier
- `client_id`: Client application identifier
- `upload_timestamp`: ISO 8601 timestamp
- `processing_status`: uploaded|processing|completed|failed|manual_review
- `file_metadata`: JSON object with file details
- `processing_results`: JSON object with BDA results
- `quality_metrics`: JSON object with evaluation results
- `callback_url`: Client notification endpoint

## Monitoring and Observability

### CloudWatch Metrics

**Step Functions Metrics**:
- Execution success/failure rates
- Execution duration
- State transition counts
- Error rates by state

**Lambda Metrics**:
- Invocation count
- Duration
- Error rate
- Memory utilization
- Cold start frequency

### CloudWatch Alarms

**Critical Alarms**:
- Step Functions execution failure rate > 5%
- Lambda error rate > 2%
- DLQ message count > 0
- Processing duration > 5 minutes

**Warning Alarms**:
- Manual review queue depth > 10
- Lambda cold start rate > 20%
- Memory utilization > 80%

### New Relic Integration

**Custom Metrics**:
- Document processing throughput
- Classification accuracy rates
- Field extraction success rates
- End-to-end processing latency

**Dashboards**:
- Real-time processing status
- Quality metrics trends
- Error analysis and troubleshooting
- Performance optimization insights

This comprehensive reference provides all the technical details needed for implementing and maintaining the Lambda functions and Step Functions orchestration in the IDP system.