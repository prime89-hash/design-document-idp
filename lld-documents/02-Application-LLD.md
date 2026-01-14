# Application Low Level Design (LLD)
## Intelligent Document Processing (IDP) Solution

### Document Control
- **Version**: 1.0
- **Last Updated**: January 14, 2026
- **Status**: Draft

---

## Application Processing Flow

```
┌─────────────────────────────────────────────────────────────────────────────────┐
│                        APPLICATION PROCESSING FLOW                               │
└─────────────────────────────────────────────────────────────────────────────────┘

PHASE 1: DOCUMENT UPLOAD
─────────────────────────
Client Application
    │
    │ POST /upload {document_type, callback_url}
    ▼
API Gateway
    │
    │ Invoke
    ▼
Lambda: Presigned URL Generator
    │
    ├─ Generate UUID (document_id)
    ├─ Create S3 pre-signed URL (PUT, 1 hour expiry)
    ├─ Store metadata in DynamoDB
    │  {document_id, presigned_url, callback_url, status: UPLOADED}
    │
    │ Return {document_id, upload_url, expires_in: 3600}
    ▼
Client uploads document to S3 using presigned URL


PHASE 2: EVENT TRIGGERING
──────────────────────────
S3 Bucket (Input Documents)
    │
    │ S3 Event: ObjectCreated
    ▼
EventBridge Rule
    │
    │ Send message
    ▼
SQS Queue (Document Processing)
    │
    │ Poll (batch: 10, long polling: 20s)
    ▼
Lambda: SQS Poller
    │
    ├─ Parse S3 event from SQS message
    ├─ Extract document_id, bucket, key
    ├─ Start Step Functions execution
    │  Input: {document_id, s3_bucket, s3_key}
    │
    │ On Success: Delete message from SQS
    │ On Failure: Return to SQS (retry after visibility timeout)
    ▼
Step Functions State Machine


PHASE 3: DOCUMENT PROCESSING (BDA HANDLES EVERYTHING)
──────────────────────────────────────────────────────
Step Functions Execution Start
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ STATE 1: BDA Document Analysis (All-in-One)                     │
│                                                                  │
│ Lambda: BDA Orchestrator                                        │
│   ├─ Download document from S3                                 │
│   ├─ Load document type config from S3/DynamoDB cache          │
│   ├─ Get BDA project ARN and blueprint IDs                     │
│   │                                                              │
│   ├─ Invoke BDA StartDocumentAnalysis API                      │
│   │  └─ BDA automatically performs:                            │
│   │     • Language Detection (built-in)                        │
│   │     • Document Classification (using blueprint)            │
│   │     • Schema Extraction (using blueprint)                  │
│   │     • OCR and text extraction                              │
│   │     • Confidence scoring                                   │
│   │                                                              │
│   ├─ Poll BDA GetDocumentAnalysis API for completion           │
│   │  └─ Wait for job status: SUCCEEDED                         │
│   │                                                              │
│   ├─ Parse BDA response:                                       │
│   │  • Detected language                                       │
│   │  • Document classification                                 │
│   │  • Extracted fields with confidence scores                 │
│   │                                                              │
│   └─ Return: {                                                  │
│       language: "en",                                           │
│       language_confidence: 0.98,                                │
│       document_type: "passport",                                │
│       classification_confidence: 0.96,                          │
│       extracted_data: {...},                                    │
│       field_confidences: {...},                                 │
│       overall_confidence: 0.96                                  │
│     }                                                            │
│                                                                  │
│ Output: Complete BDA analysis result                            │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ STATE 2: Confidence Check (Choice State)                        │
│                                                                  │
│ IF classification_confidence >= threshold (0.85):               │
│    └─ Continue to Validation & Output                          │
│ ELSE:                                                            │
│    └─ Mark as LOW_CONFIDENCE → Update Status → END             │
└─────────────────────────────────────────────────────────────────┘
    │
    │ (Assuming HIGH confidence)
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ STATE 3: Validation & Output Preparation                        │
│                                                                  │
│ Lambda: Output Processor                                        │
│   ├─ Validate extracted data against schema                    │
│   │  • Check required fields present                           │
│   │  • Validate field formats (dates, patterns)                │
│   │  • Apply business rules                                    │
│   │                                                              │
│   ├─ Calculate overall confidence score                        │
│   │  • Weighted average of field confidences                   │
│   │  • Required fields weighted higher (70%)                   │
│   │  • Optional fields weighted lower (30%)                    │
│   │                                                              │
│   ├─ Create output JSON structure                              │
│   │  • Document metadata                                       │
│   │  • Extracted data                                          │
│   │  • Confidence scores                                       │
│   │  • Validation results                                      │
│   │                                                              │
│   ├─ Save output JSON to S3 (output bucket)                    │
│   │                                                              │
│   └─ Return: {                                                  │
│       validation: {is_valid: true, errors: []},                │
│       s3_output_location: "s3://...",                           │
│       overall_confidence: 0.96                                  │
│     }                                                            │
│                                                                  │
│ Output: Validated and saved extraction result                   │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────────────────────────────┐
│ STATE 4: Status Update                                          │
│                                                                  │
│ Lambda: Status Updater                                          │
│   ├─ Update DynamoDB metadata table                            │
│   │  {status: COMPLETED, document_type, confidences, ...}      │
│   ├─ Retrieve callback_url from metadata                       │
│   ├─ If callback_url exists:                                   │
│   │  └─ POST to callback_url with processing result            │
│   └─ Log completion                                             │
│                                                                  │
│ Output: {status: "SUCCESS"}                                     │
└─────────────────────────────────────────────────────────────────┘
    │
    ▼
Step Functions Execution Complete


PHASE 4: ERROR HANDLING
────────────────────────
Any Lambda Failure
    │
    ▼
Step Functions Retry Logic
    ├─ Attempt 1: Wait 2s → Retry Lambda
    ├─ Attempt 2: Wait 4s → Retry Lambda
    └─ Attempt 3: Wait 8s → Retry Lambda
    │
    │ All retries failed
    ▼
Step Functions returns error to SQS Poller
    │
    ▼
SQS increments receive count
    │
    ├─ If receive_count < 3: Message visible again (retry)
    │
    └─ If receive_count >= 3: Move to DLQ
        │
        ▼
    Dead Letter Queue
        │
        │ Trigger
        ▼
    Lambda: DLQ Analyzer
        │
        ├─ Parse error details
        ├─ Extract document_id, error_message
        ├─ Create detailed alert
        └─ Publish to SNS
        │
        ▼
    SNS Topic → Email, New Relic, Slack


PHASE 5: OUTPUT & CLEANUP
──────────────────────────
S3 Output Bucket
    └─ outputs/{document_id}.json
       {
         document_id,
         processing_metadata: {timestamp, duration, config_version},
         document_info: {document_type, language, confidence},
         extracted_data: {...},
         confidence_scores: {overall, fields: {...}},
         validation: {is_valid, errors: []}
       }

DynamoDB Metadata Table
    └─ {
         document_id,
         status: COMPLETED,
         document_type,
         classification_confidence,
         extraction_confidence,
         s3_output_key,
         processing_duration_ms,
         updated_at
       }

Client Callback (Optional)
    └─ POST to callback_url
       {
         document_id,
         status: COMPLETED,
         document_type,
         output_location,
         processing_time_ms,
         timestamp
       }

Daily Cleanup (EventBridge Schedule)
    │
    ▼
Lambda: Document Cleanup
    ├─ List objects in input bucket older than retention period (7 days)
    ├─ Delete objects in batches
    └─ Log cleanup statistics
```

---

## 1. Application Architecture Overview

### 1.1 Design Principles
- Event-driven architecture
- Microservices pattern with Lambda functions
- Stateless processing
- Configuration-driven behavior
- Fail-fast with comprehensive error handling

### 1.2 Technology Stack
- **Language**: Python 3.12
- **Container**: Docker (multi-stage builds)
- **AWS SDK**: Boto3
- **Logging**: Python JSON Logger
- **Testing**: Pytest, Moto (AWS mocking)
- **Code Quality**: Black, Flake8, Pylint, Bandit

---

## 2. Application Components

### 2.1 Lambda Functions Detail

#### 2.1.1 Presigned URL Generator

**Purpose**: Generate pre-signed S3 URLs for secure document upload

**Input Event**:
```json
{
  "body": {
    "document_type": "passport",
    "callback_url": "https://client.example.com/callback",
    "metadata": {
      "user_id": "12345",
      "session_id": "abc-xyz"
    }
  }
}
```

**Processing Logic**:
1. Validate input parameters
2. Generate unique document_id (UUID)
3. Create S3 pre-signed URL (PUT operation, 1 hour expiry)
4. Store metadata in DynamoDB
5. Return response with presigned URL

**Output**:
```json
{
  "statusCode": 200,
  "body": {
    "document_id": "550e8400-e29b-41d4-a716-446655440000",
    "upload_url": "https://s3.amazonaws.com/...",
    "expires_in": 3600
  }
}
```

**Error Handling**:
- Invalid input: 400 Bad Request
- DynamoDB failure: 500 Internal Server Error
- S3 failure: 500 Internal Server Error


#### 2.1.2 BDA Orchestrator Lambda

**Purpose**: Orchestrate Bedrock Data Automation processing

**Input Event** (from Step Functions):
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "s3_bucket": "idp-input-documents-dev-123456",
  "s3_key": "documents/550e8400-e29b-41d4-a716-446655440000.pdf",
  "document_type": "passport",
  "language": "en"
}
```

**Processing Logic**:
1. Retrieve document from S3
2. Load configuration for document type
3. Invoke BDA with appropriate blueprint
4. Poll for BDA job completion (async)
5. Return extracted data

**BDA API Calls**:
```python
# Start document analysis
response = bedrock_client.start_document_analysis(
    ProjectArn=project_arn,
    DocumentLocation={
        'S3Object': {
            'Bucket': bucket,
            'Name': key
        }
    },
    BlueprintId=blueprint_id
)

# Get results
result = bedrock_client.get_document_analysis(
    JobId=job_id
)
```

**Output**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "bda_job_id": "bda-job-12345",
  "status": "COMPLETED",
  "raw_extraction": {
    "fields": [...],
    "confidence": 0.95
  }
}
```


#### 2.1.3 Language Detection Lambda

**Purpose**: Detect document language using AWS Comprehend or Bedrock

**Input Event**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "s3_bucket": "idp-input-documents-dev-123456",
  "s3_key": "documents/550e8400-e29b-41d4-a716-446655440000.pdf"
}
```

**Processing Logic**:
1. Extract text sample from document (first page)
2. Call AWS Comprehend DetectDominantLanguage
3. If confidence < 0.9, fallback to Bedrock LLM
4. Return detected language with confidence

**Output**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "language": "en",
  "language_confidence": 0.98,
  "detected_languages": [
    {"code": "en", "score": 0.98},
    {"code": "es", "score": 0.02}
  ]
}
```

#### 2.1.4 Document Classifier Lambda

**Purpose**: Classify document type with confidence score

**Input Event**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "s3_bucket": "idp-input-documents-dev-123456",
  "s3_key": "documents/550e8400-e29b-41d4-a716-446655440000.pdf",
  "language": "en"
}
```

**Processing Logic**:
1. Load classification configuration
2. Extract document features (text, layout)
3. Invoke BDA classification blueprint
4. Apply confidence threshold
5. Return classification result

**Classification Logic**:
```python
def classify_document(document_data, config):
    # Get all possible document types
    document_types = config['document_types']
    
    # Invoke BDA for classification
    classification_result = invoke_bda_classification(
        document_data=document_data,
        blueprint_id=config['classification_blueprint_id']
    )
    
    # Apply confidence threshold
    if classification_result['confidence'] >= config['confidence_threshold']:
        return {
            'document_type': classification_result['type'],
            'confidence': classification_result['confidence'],
            'status': 'CLASSIFIED'
        }
    else:
        return {
            'document_type': 'UNKNOWN',
            'confidence': classification_result['confidence'],
            'status': 'LOW_CONFIDENCE'
        }
```

**Output**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "document_type": "passport",
  "classification_confidence": 0.96,
  "alternative_types": [
    {"type": "national_id", "confidence": 0.03},
    {"type": "drivers_license", "confidence": 0.01}
  ],
  "status": "CLASSIFIED"
}
```


#### 2.1.5 Schema Extractor Lambda

**Purpose**: Extract structured data based on document type schema

**Input Event**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "s3_bucket": "idp-input-documents-dev-123456",
  "s3_key": "documents/550e8400-e29b-41d4-a716-446655440000.pdf",
  "document_type": "passport",
  "language": "en"
}
```

**Processing Logic**:
1. Load schema for document type
2. Load extraction prompt from config
3. Invoke BDA with extraction blueprint
4. Validate extracted fields against schema
5. Calculate field-level confidence scores
6. Save output to S3

**Schema Validation**:
```python
def validate_extraction(extracted_data, schema):
    validation_result = {
        'is_valid': True,
        'missing_required_fields': [],
        'invalid_fields': [],
        'field_confidences': {}
    }
    
    # Check required fields
    for field in schema['required']:
        if field not in extracted_data:
            validation_result['missing_required_fields'].append(field)
            validation_result['is_valid'] = False
    
    # Validate field types and formats
    for field, value in extracted_data.items():
        field_schema = schema['properties'].get(field)
        if field_schema:
            if not validate_field_type(value, field_schema):
                validation_result['invalid_fields'].append(field)
                validation_result['is_valid'] = False
    
    return validation_result
```

**Output**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "document_type": "passport",
  "extraction_status": "SUCCESS",
  "extracted_data": {
    "passport_number": "AB1234567",
    "given_names": "JOHN MICHAEL",
    "surname": "SMITH",
    "date_of_birth": "1985-03-15",
    "nationality": "GBR",
    "date_of_issue": "2020-01-10",
    "date_of_expiry": "2030-01-10",
    "place_of_birth": "LONDON"
  },
  "field_confidences": {
    "passport_number": 0.99,
    "given_names": 0.97,
    "surname": 0.98,
    "date_of_birth": 0.96,
    "nationality": 0.99,
    "date_of_issue": 0.95,
    "date_of_expiry": 0.95,
    "place_of_birth": 0.92
  },
  "overall_confidence": 0.96,
  "validation": {
    "is_valid": true,
    "missing_required_fields": [],
    "invalid_fields": []
  },
  "s3_output_location": "s3://idp-output-documents-dev-123456/outputs/550e8400-e29b-41d4-a716-446655440000.json"
}
```


#### 2.1.6 Status Updater Lambda

**Purpose**: Update DynamoDB with processing status and trigger callbacks

**Input Event**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "document_type": "passport",
  "classification_confidence": 0.96,
  "extraction_confidence": 0.96,
  "s3_output_key": "outputs/550e8400-e29b-41d4-a716-446655440000.json",
  "processing_duration_ms": 45000,
  "error_message": null
}
```

**Processing Logic**:
1. Update DynamoDB metadata table
2. Retrieve callback URL from metadata
3. If callback URL exists, invoke client callback
4. Log status update

**Callback Payload**:
```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "status": "COMPLETED",
  "document_type": "passport",
  "output_location": "s3://idp-output-documents-dev-123456/outputs/550e8400-e29b-41d4-a716-446655440000.json",
  "processing_time_ms": 45000,
  "timestamp": "2026-01-14T10:30:00Z"
}
```

#### 2.1.7 Document Cleanup Lambda

**Purpose**: Delete old documents based on retention policy

**Trigger**: EventBridge scheduled rule (daily at 2 AM UTC)

**Processing Logic**:
1. List objects in input bucket older than retention period
2. Delete objects in batches
3. Log cleanup statistics

---

## 3. Step Functions Workflow

### 3.1 State Machine Definition

```json
{
  "Comment": "IDP Document Processing Workflow",
  "StartAt": "LanguageDetection",
  "States": {
    "LanguageDetection": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-language-detector-env",
      "ResultPath": "$.language_result",
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
      "Next": "DocumentClassification"
    },
    "DocumentClassification": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-document-classifier-env",
      "ResultPath": "$.classification_result",
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
      "Next": "CheckClassificationConfidence"
    },
    "CheckClassificationConfidence": {
      "Type": "Choice",
      "Choices": [
        {
          "Variable": "$.classification_result.status",
          "StringEquals": "CLASSIFIED",
          "Next": "SchemaExtraction"
        }
      ],
      "Default": "LowConfidenceClassification"
    },
    "LowConfidenceClassification": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-status-updater-env",
      "Parameters": {
        "document_id.$": "$.document_id",
        "status": "LOW_CONFIDENCE",
        "classification_confidence.$": "$.classification_result.classification_confidence"
      },
      "End": true
    },
    "SchemaExtraction": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-schema-extractor-env",
      "ResultPath": "$.extraction_result",
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
      "Next": "UpdateStatusCompleted"
    },
    "UpdateStatusCompleted": {
      "Type": "Task",
      "Resource": "arn:aws:lambda:region:account:function:idp-status-updater-env",
      "Parameters": {
        "document_id.$": "$.document_id",
        "status": "COMPLETED",
        "document_type.$": "$.classification_result.document_type",
        "classification_confidence.$": "$.classification_result.classification_confidence",
        "extraction_confidence.$": "$.extraction_result.overall_confidence",
        "s3_output_key.$": "$.extraction_result.s3_output_location"
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


### 3.2 Workflow Execution Flow

```
1. S3 Upload Event → EventBridge → Step Functions
2. Language Detection (30s)
3. Document Classification (60s)
4. Confidence Check
   - If HIGH: Continue to Extraction
   - If LOW: Mark as LOW_CONFIDENCE, End
5. Schema Extraction (120s)
6. Update Status to COMPLETED
7. Trigger Callback (if configured)
```

### 3.3 Error Handling Strategy

**Retry Policy**:
- Exponential backoff: 2s, 4s, 8s
- Max attempts: 3
- Retry on: Transient errors, throttling, timeouts

**Error States**:
- `FAILED`: Unrecoverable error
- `LOW_CONFIDENCE`: Classification confidence below threshold
- `TIMEOUT`: Processing exceeded time limit

---

## 4. Configuration Management

### 4.1 Configuration Structure

**S3 Configuration Path**: `s3://idp-config-{env}/configs/{document-type}/`

**config.json**:
```json
{
  "document_type": "passport",
  "version": "1.0",
  "enabled": true,
  "classification": {
    "blueprint_id": "blueprint-passport-classification",
    "confidence_threshold": 0.85,
    "model": "anthropic.claude-3-sonnet-20240229-v1:0"
  },
  "extraction": {
    "blueprint_id": "blueprint-passport-extraction",
    "confidence_threshold": 0.80,
    "model": "anthropic.claude-3-sonnet-20240229-v1:0",
    "max_retries": 2
  },
  "language_detection": {
    "enabled": true,
    "supported_languages": ["en", "es", "fr", "de", "it", "pt"]
  },
  "validation": {
    "required_fields": [
      "passport_number",
      "surname",
      "given_names",
      "date_of_birth",
      "nationality"
    ],
    "optional_fields": [
      "place_of_birth",
      "date_of_issue",
      "date_of_expiry"
    ]
  }
}
```

**schema.json**:
```json
{
  "$schema": "http://json-schema.org/draft-07/schema#",
  "type": "object",
  "properties": {
    "passport_number": {
      "type": "string",
      "pattern": "^[A-Z0-9]{6,9}$",
      "description": "Passport number"
    },
    "surname": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "given_names": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100
    },
    "date_of_birth": {
      "type": "string",
      "format": "date",
      "pattern": "^\\d{4}-\\d{2}-\\d{2}$"
    },
    "nationality": {
      "type": "string",
      "pattern": "^[A-Z]{3}$",
      "description": "ISO 3166-1 alpha-3 country code"
    },
    "date_of_issue": {
      "type": "string",
      "format": "date"
    },
    "date_of_expiry": {
      "type": "string",
      "format": "date"
    },
    "place_of_birth": {
      "type": "string"
    }
  },
  "required": [
    "passport_number",
    "surname",
    "given_names",
    "date_of_birth",
    "nationality"
  ]
}
```

**prompt.txt**:
```
You are an expert document extraction AI. Extract the following information from the passport document:

1. Passport Number: The unique identifier for this passport
2. Surname: Family name as shown on the passport
3. Given Names: First and middle names
4. Date of Birth: In YYYY-MM-DD format
5. Nationality: Three-letter country code (ISO 3166-1 alpha-3)
6. Date of Issue: When the passport was issued (YYYY-MM-DD)
7. Date of Expiry: When the passport expires (YYYY-MM-DD)
8. Place of Birth: City or location of birth

Return the data in JSON format with the exact field names listed above.
Only extract information that is clearly visible in the document.
If a field is not visible or unclear, omit it from the response.
```

### 4.2 Configuration Loading

```python
class ConfigurationManager:
    def __init__(self, s3_client, dynamodb_client, config_bucket, cache_table):
        self.s3 = s3_client
        self.dynamodb = dynamodb_client
        self.config_bucket = config_bucket
        self.cache_table = cache_table
    
    def get_config(self, document_type):
        # Try cache first
        cached_config = self._get_from_cache(document_type)
        if cached_config and cached_config['is_active']:
            return cached_config
        
        # Load from S3
        config = self._load_from_s3(document_type)
        
        # Update cache
        self._update_cache(document_type, config)
        
        return config
    
    def _load_from_s3(self, document_type):
        config_key = f"configs/{document_type}/config.json"
        schema_key = f"configs/{document_type}/schema.json"
        prompt_key = f"configs/{document_type}/prompt.txt"
        
        config = json.loads(self.s3.get_object(
            Bucket=self.config_bucket,
            Key=config_key
        )['Body'].read())
        
        schema = json.loads(self.s3.get_object(
            Bucket=self.config_bucket,
            Key=schema_key
        )['Body'].read())
        
        prompt = self.s3.get_object(
            Bucket=self.config_bucket,
            Key=prompt_key
        )['Body'].read().decode('utf-8')
        
        return {
            'config': config,
            'schema': schema,
            'prompt': prompt,
            'version': config['version']
        }
```

---

## 5. Data Models

### 5.1 DynamoDB Document Metadata Schema

```python
from datetime import datetime
from enum import Enum

class DocumentStatus(Enum):
    UPLOADED = "UPLOADED"
    PROCESSING = "PROCESSING"
    COMPLETED = "COMPLETED"
    FAILED = "FAILED"
    LOW_CONFIDENCE = "LOW_CONFIDENCE"

class DocumentMetadata:
    def __init__(self):
        self.document_id: str
        self.upload_timestamp: int
        self.presigned_url: str
        self.callback_url: str
        self.status: DocumentStatus
        self.document_type: str
        self.classification_confidence: float
        self.extraction_confidence: float
        self.language: str
        self.s3_input_key: str
        self.s3_output_key: str
        self.error_message: str
        self.processing_duration_ms: int
        self.config_version: str
        self.created_at: str
        self.updated_at: str
        self.metadata: dict
    
    def to_dynamodb_item(self):
        return {
            'document_id': self.document_id,
            'upload_timestamp': self.upload_timestamp,
            'presigned_url': self.presigned_url,
            'callback_url': self.callback_url,
            'status': self.status.value,
            'document_type': self.document_type,
            'classification_confidence': self.classification_confidence,
            'extraction_confidence': self.extraction_confidence,
            'language': self.language,
            's3_input_key': self.s3_input_key,
            's3_output_key': self.s3_output_key,
            'error_message': self.error_message,
            'processing_duration_ms': self.processing_duration_ms,
            'config_version': self.config_version,
            'created_at': self.created_at,
            'updated_at': self.updated_at,
            'metadata': self.metadata
        }
```


### 5.2 Output JSON Structure

```json
{
  "document_id": "550e8400-e29b-41d4-a716-446655440000",
  "processing_metadata": {
    "timestamp": "2026-01-14T10:30:00Z",
    "processing_duration_ms": 45000,
    "config_version": "1.0"
  },
  "document_info": {
    "document_type": "passport",
    "language": "en",
    "classification_confidence": 0.96
  },
  "extracted_data": {
    "passport_number": "AB1234567",
    "surname": "SMITH",
    "given_names": "JOHN MICHAEL",
    "date_of_birth": "1985-03-15",
    "nationality": "GBR",
    "date_of_issue": "2020-01-10",
    "date_of_expiry": "2030-01-10",
    "place_of_birth": "LONDON"
  },
  "confidence_scores": {
    "overall": 0.96,
    "fields": {
      "passport_number": 0.99,
      "surname": 0.98,
      "given_names": 0.97,
      "date_of_birth": 0.96,
      "nationality": 0.99,
      "date_of_issue": 0.95,
      "date_of_expiry": 0.95,
      "place_of_birth": 0.92
    }
  },
  "validation": {
    "is_valid": true,
    "missing_required_fields": [],
    "invalid_fields": []
  }
}
```

---

## 6. Error Handling & Logging

### 6.1 Error Categories

**Transient Errors** (Retryable):
- AWS service throttling
- Network timeouts
- Temporary service unavailability

**Permanent Errors** (Non-retryable):
- Invalid document format
- Unsupported document type
- Schema validation failure
- Missing configuration

### 6.2 Logging Structure

```python
import logging
from pythonjsonlogger import jsonlogger

class CustomJsonFormatter(jsonlogger.JsonFormatter):
    def add_fields(self, log_record, record, message_dict):
        super().add_fields(log_record, record, message_dict)
        log_record['timestamp'] = datetime.utcnow().isoformat()
        log_record['environment'] = os.environ.get('ENVIRONMENT')
        log_record['function_name'] = os.environ.get('AWS_LAMBDA_FUNCTION_NAME')
        log_record['request_id'] = getattr(record, 'aws_request_id', None)

# Usage
logger = logging.getLogger()
handler = logging.StreamHandler()
formatter = CustomJsonFormatter('%(timestamp)s %(level)s %(name)s %(message)s')
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Log example
logger.info('Document processing started', extra={
    'document_id': document_id,
    'document_type': document_type,
    'operation': 'classification'
})
```

### 6.3 Structured Logging Examples

```python
# Success log
logger.info('Document classified successfully', extra={
    'document_id': '550e8400-e29b-41d4-a716-446655440000',
    'document_type': 'passport',
    'confidence': 0.96,
    'duration_ms': 1500
})

# Error log
logger.error('BDA invocation failed', extra={
    'document_id': '550e8400-e29b-41d4-a716-446655440000',
    'error_type': 'ThrottlingException',
    'error_message': 'Rate exceeded',
    'retry_attempt': 2
})

# Warning log
logger.warning('Low confidence classification', extra={
    'document_id': '550e8400-e29b-41d4-a716-446655440000',
    'confidence': 0.72,
    'threshold': 0.85,
    'document_type': 'unknown'
})
```

---

## 7. Testing Strategy

### 7.1 Unit Tests

**Test Coverage**: > 80%

**Example Test**:
```python
import pytest
from moto import mock_s3, mock_dynamodb
from lambda_functions.presigned_url_generator import handler

@mock_s3
@mock_dynamodb
def test_presigned_url_generation():
    # Setup
    event = {
        'body': json.dumps({
            'document_type': 'passport',
            'callback_url': 'https://example.com/callback'
        })
    }
    
    # Execute
    response = handler(event, {})
    
    # Assert
    assert response['statusCode'] == 200
    body = json.loads(response['body'])
    assert 'document_id' in body
    assert 'upload_url' in body
    assert 'expires_in' in body
```

### 7.2 Integration Tests

**Test Scenarios**:
1. End-to-end document processing
2. Error handling and retries
3. Configuration loading
4. Callback invocation

```python
@pytest.mark.integration
def test_end_to_end_processing():
    # Upload document
    upload_response = upload_document('test_passport.pdf')
    document_id = upload_response['document_id']
    
    # Wait for processing
    status = wait_for_completion(document_id, timeout=300)
    
    # Verify results
    assert status == 'COMPLETED'
    output = get_output(document_id)
    assert output['document_type'] == 'passport'
    assert output['confidence_scores']['overall'] > 0.85
```

### 7.3 LLM-Based Evaluation

```python
def evaluate_extraction_quality(extracted_data, ground_truth):
    """
    Use Bedrock LLM to evaluate extraction quality
    """
    prompt = f"""
    Compare the extracted data with the ground truth and provide a quality score.
    
    Extracted Data:
    {json.dumps(extracted_data, indent=2)}
    
    Ground Truth:
    {json.dumps(ground_truth, indent=2)}
    
    Evaluate:
    1. Field accuracy (0-100)
    2. Completeness (0-100)
    3. Format correctness (0-100)
    
    Return JSON with scores and explanation.
    """
    
    response = bedrock_client.invoke_model(
        modelId='anthropic.claude-3-sonnet-20240229-v1:0',
        body=json.dumps({
            'anthropic_version': 'bedrock-2023-05-31',
            'messages': [{'role': 'user', 'content': prompt}],
            'max_tokens': 1000
        })
    )
    
    return json.loads(response['body'].read())
```

---

## 8. Performance Optimization

### 8.1 Optimization Strategies

**Lambda Cold Start Reduction**:
- Use ARM64 architecture
- Minimize package size
- Use Lambda layers for shared dependencies
- Implement connection pooling for AWS clients

**S3 Performance**:
- Use S3 Transfer Acceleration for large files
- Implement multipart upload for files > 100MB
- Use byte-range fetches for partial reads

**DynamoDB Performance**:
- Use batch operations where possible
- Implement caching for configuration data
- Use GSIs for efficient queries

### 8.2 Caching Strategy

```python
from functools import lru_cache
import time

class ConfigCache:
    def __init__(self, ttl=300):
        self.cache = {}
        self.ttl = ttl
    
    def get(self, key):
        if key in self.cache:
            value, timestamp = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return value
            else:
                del self.cache[key]
        return None
    
    def set(self, key, value):
        self.cache[key] = (value, time.time())

# Global cache instance
config_cache = ConfigCache(ttl=300)
```

---

## 9. Security Implementation

### 9.1 Input Validation

```python
from jsonschema import validate, ValidationError

def validate_upload_request(request_body):
    schema = {
        "type": "object",
        "properties": {
            "document_type": {
                "type": "string",
                "enum": ["passport", "drivers_license", "national_id", ...]
            },
            "callback_url": {
                "type": "string",
                "format": "uri",
                "pattern": "^https://"
            },
            "metadata": {
                "type": "object"
            }
        }
    }
    
    try:
        validate(instance=request_body, schema=schema)
        return True, None
    except ValidationError as e:
        return False, str(e)
```

### 9.2 Secrets Management

```python
import boto3
from botocore.exceptions import ClientError

class SecretsManager:
    def __init__(self):
        self.client = boto3.client('secretsmanager')
        self.cache = {}
    
    def get_secret(self, secret_name):
        if secret_name in self.cache:
            return self.cache[secret_name]
        
        try:
            response = self.client.get_secret_value(SecretId=secret_name)
            secret = json.loads(response['SecretString'])
            self.cache[secret_name] = secret
            return secret
        except ClientError as e:
            logger.error(f"Failed to retrieve secret: {e}")
            raise
```

### 9.3 Data Sanitization

```python
def sanitize_output(data):
    """
    Remove sensitive information from logs and outputs
    """
    sensitive_fields = ['callback_url', 'presigned_url']
    
    sanitized = data.copy()
    for field in sensitive_fields:
        if field in sanitized:
            sanitized[field] = '***REDACTED***'
    
    return sanitized
```

---

## 10. Application Deployment

### 10.1 Docker Image Structure

**Dockerfile**:
```dockerfile
# Multi-stage build
FROM public.ecr.aws/lambda/python:3.12 AS builder

WORKDIR /app

# Install dependencies
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt -t /app/python

# Final stage
FROM public.ecr.aws/lambda/python:3.12

WORKDIR ${LAMBDA_TASK_ROOT}

# Copy dependencies
COPY --from=builder /app/python ./

# Copy application code
COPY src/ ./

# Set handler
CMD ["lambda_function.handler"]
```

### 10.2 Application Structure

```
src/
├── lambda_functions/
│   ├── presigned_url_generator/
│   │   ├── lambda_function.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   ├── bda_orchestrator/
│   │   ├── lambda_function.py
│   │   ├── requirements.txt
│   │   └── Dockerfile
│   ├── language_detector/
│   ├── document_classifier/
│   ├── schema_extractor/
│   ├── status_updater/
│   └── document_cleanup/
├── common/
│   ├── __init__.py
│   ├── config_manager.py
│   ├── secrets_manager.py
│   ├── logger.py
│   ├── validators.py
│   └── utils.py
├── tests/
│   ├── unit/
│   ├── integration/
│   └── fixtures/
└── requirements.txt
```

---

**End of Application LLD**
