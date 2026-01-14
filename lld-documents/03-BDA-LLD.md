# Bedrock Data Automation (BDA) Low Level Design
## Intelligent Document Processing (IDP) Solution

### Document Control
- **Version**: 1.0
- **Last Updated**: January 14, 2026
- **Status**: Draft

---

## 1. BDA Overview

### 1.1 What is Bedrock Data Automation?

Amazon Bedrock Data Automation is a fully managed service that:
- Extracts structured data from documents
- Classifies documents by type
- Detects language and layout
- Uses pre-trained and custom blueprints
- Supports multiple document formats (PDF, JPEG, PNG, TIFF)

### 1.2 BDA in IDP Architecture

**Role**: Core AI/ML engine for document processing

**Responsibilities**:
1. Document classification
2. Text extraction (OCR)
3. Entity extraction
4. Schema-based data extraction
5. Confidence scoring

---

## 2. BDA Project Structure

### 2.1 Project Configuration

```json
{
  "project_name": "idp-document-processing",
  "project_arn": "arn:aws:bedrock:eu-west-1:123456789012:data-automation-project/idp-doc-proc",
  "region": "eu-west-1",
  "description": "IDP document classification and extraction",
  "created_date": "2026-01-14",
  "blueprints": [
    {
      "blueprint_id": "bp-passport-classification",
      "blueprint_type": "classification"
    },
    {
      "blueprint_id": "bp-passport-extraction",
      "blueprint_type": "extraction"
    }
  ]
}
```

### 2.2 Blueprint Types

**Classification Blueprints**:
- Purpose: Identify document type
- Input: Document image/PDF
- Output: Document type + confidence score

**Extraction Blueprints**:
- Purpose: Extract structured data
- Input: Document + schema
- Output: JSON with extracted fields + confidence scores

---

## 3. Blueprint Design

### 3.1 Classification Blueprint

#### 3.1.1 Passport Classification Blueprint

**Blueprint ID**: `bp-passport-classification`

**Configuration**:
```json
{
  "blueprint_id": "bp-passport-classification",
  "blueprint_name": "Passport Document Classifier",
  "blueprint_type": "classification",
  "model_id": "anthropic.claude-3-sonnet-20240229-v1:0",
  "document_types": [
    "passport",
    "national_id",
    "drivers_license",
    "residence_permit",
    "citizenship_card"
  ],
  "classification_prompt": "Analyze this identity document and classify it as one of the following types: passport, national_id, drivers_license, residence_permit, or citizenship_card. Return the classification with a confidence score.",
  "features": [
    "document_layout",
    "text_patterns",
    "visual_elements",
    "security_features"
  ]
}
```


#### 3.1.2 Document Type Blueprints

**Verification Documents**:
- `bp-id-classification`: National ID, Driver's License, Passport
- `bp-poa-classification`: Utility bills, Bank statements, Lease agreements
- `bp-sof-classification`: Payslips, Bank statements, Tax declarations
- `bp-phid-classification`: Photo holding ID verification

**Legal Claims Documents**:
- `bp-uk-claims-classification`: UK court documents
- `bp-malta-claims-classification`: Malta enforcement documents
- `bp-sweden-claims-classification`: Swedish enforcement documents
- `bp-spain-claims-classification`: Spanish court documents
- `bp-germany-claims-classification`: German court documents

### 3.2 Extraction Blueprints

#### 3.2.1 Passport Extraction Blueprint

**Blueprint ID**: `bp-passport-extraction`

**Schema Definition**:
```json
{
  "blueprint_id": "bp-passport-extraction",
  "blueprint_name": "Passport Data Extractor",
  "blueprint_type": "extraction",
  "model_id": "anthropic.claude-3-sonnet-20240229-v1:0",
  "schema": {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
      "passport_number": {
        "type": "string",
        "description": "Passport number (alphanumeric)",
        "pattern": "^[A-Z0-9]{6,9}$"
      },
      "surname": {
        "type": "string",
        "description": "Family name / Last name"
      },
      "given_names": {
        "type": "string",
        "description": "First and middle names"
      },
      "date_of_birth": {
        "type": "string",
        "format": "date",
        "description": "Date of birth in YYYY-MM-DD format"
      },
      "nationality": {
        "type": "string",
        "description": "Nationality (ISO 3166-1 alpha-3 code)",
        "pattern": "^[A-Z]{3}$"
      },
      "sex": {
        "type": "string",
        "enum": ["M", "F", "X"],
        "description": "Gender"
      },
      "place_of_birth": {
        "type": "string",
        "description": "Place of birth"
      },
      "date_of_issue": {
        "type": "string",
        "format": "date",
        "description": "Issue date in YYYY-MM-DD format"
      },
      "date_of_expiry": {
        "type": "string",
        "format": "date",
        "description": "Expiry date in YYYY-MM-DD format"
      },
      "issuing_authority": {
        "type": "string",
        "description": "Authority that issued the passport"
      }
    },
    "required": [
      "passport_number",
      "surname",
      "given_names",
      "date_of_birth",
      "nationality"
    ]
  },
  "extraction_prompt": "Extract all visible information from this passport document. Focus on the machine-readable zone (MRZ) and the visual inspection zone (VIZ). Return data in the specified JSON schema format."
}
```

#### 3.2.2 Bank Statement Extraction Blueprint

**Blueprint ID**: `bp-bank-statement-extraction`

**Schema Definition**:
```json
{
  "blueprint_id": "bp-bank-statement-extraction",
  "blueprint_name": "Bank Statement Data Extractor",
  "blueprint_type": "extraction",
  "model_id": "anthropic.claude-3-sonnet-20240229-v1:0",
  "schema": {
    "type": "object",
    "properties": {
      "account_holder_name": {
        "type": "string",
        "description": "Name of the account holder"
      },
      "account_number": {
        "type": "string",
        "description": "Bank account number"
      },
      "bank_name": {
        "type": "string",
        "description": "Name of the bank"
      },
      "statement_period": {
        "type": "object",
        "properties": {
          "start_date": {"type": "string", "format": "date"},
          "end_date": {"type": "string", "format": "date"}
        }
      },
      "address": {
        "type": "object",
        "properties": {
          "street": {"type": "string"},
          "city": {"type": "string"},
          "postal_code": {"type": "string"},
          "country": {"type": "string"}
        }
      },
      "opening_balance": {
        "type": "number",
        "description": "Opening balance amount"
      },
      "closing_balance": {
        "type": "number",
        "description": "Closing balance amount"
      },
      "currency": {
        "type": "string",
        "description": "Currency code (ISO 4217)"
      },
      "transactions": {
        "type": "array",
        "items": {
          "type": "object",
          "properties": {
            "date": {"type": "string", "format": "date"},
            "description": {"type": "string"},
            "amount": {"type": "number"},
            "balance": {"type": "number"}
          }
        }
      }
    },
    "required": [
      "account_holder_name",
      "account_number",
      "bank_name",
      "statement_period"
    ]
  }
}
```

#### 3.2.3 Utility Bill Extraction Blueprint

**Blueprint ID**: `bp-utility-bill-extraction`

**Schema Definition**:
```json
{
  "blueprint_id": "bp-utility-bill-extraction",
  "blueprint_name": "Utility Bill Data Extractor",
  "blueprint_type": "extraction",
  "model_id": "anthropic.claude-3-sonnet-20240229-v1:0",
  "schema": {
    "type": "object",
    "properties": {
      "customer_name": {
        "type": "string",
        "description": "Name of the customer"
      },
      "service_address": {
        "type": "object",
        "properties": {
          "street": {"type": "string"},
          "city": {"type": "string"},
          "postal_code": {"type": "string"},
          "country": {"type": "string"}
        }
      },
      "utility_type": {
        "type": "string",
        "enum": ["electricity", "gas", "water", "internet", "phone"],
        "description": "Type of utility service"
      },
      "provider_name": {
        "type": "string",
        "description": "Utility provider name"
      },
      "account_number": {
        "type": "string",
        "description": "Customer account number"
      },
      "bill_date": {
        "type": "string",
        "format": "date",
        "description": "Bill issue date"
      },
      "billing_period": {
        "type": "object",
        "properties": {
          "start_date": {"type": "string", "format": "date"},
          "end_date": {"type": "string", "format": "date"}
        }
      },
      "total_amount": {
        "type": "number",
        "description": "Total amount due"
      },
      "due_date": {
        "type": "string",
        "format": "date",
        "description": "Payment due date"
      },
      "currency": {
        "type": "string",
        "description": "Currency code"
      }
    },
    "required": [
      "customer_name",
      "service_address",
      "utility_type",
      "bill_date"
    ]
  }
}
```

---

## 4. BDA API Integration

### 4.1 API Operations

#### 4.1.1 Start Document Analysis

```python
import boto3
import json

bedrock_client = boto3.client('bedrock-data-automation', region_name='eu-west-1')

def start_document_analysis(s3_bucket, s3_key, blueprint_id, project_arn):
    """
    Start asynchronous document analysis with BDA
    """
    try:
        response = bedrock_client.start_document_analysis(
            ProjectArn=project_arn,
            DocumentLocation={
                'S3Object': {
                    'Bucket': s3_bucket,
                    'Name': s3_key
                }
            },
            BlueprintId=blueprint_id,
            OutputConfig={
                'S3OutputConfiguration': {
                    'BucketName': f'{s3_bucket}-output',
                    'Prefix': 'bda-results/'
                }
            }
        )
        
        return {
            'job_id': response['JobId'],
            'status': response['JobStatus']
        }
    
    except Exception as e:
        logger.error(f"Failed to start BDA analysis: {str(e)}")
        raise
```

#### 4.1.2 Get Document Analysis Results

```python
def get_document_analysis(job_id):
    """
    Retrieve document analysis results
    """
    try:
        response = bedrock_client.get_document_analysis(
            JobId=job_id
        )
        
        return {
            'job_id': job_id,
            'status': response['JobStatus'],
            'results': response.get('Results', {}),
            'confidence': response.get('Confidence', 0.0),
            'error_message': response.get('StatusMessage', None)
        }
    
    except Exception as e:
        logger.error(f"Failed to get BDA results: {str(e)}")
        raise
```

#### 4.1.3 Poll for Completion

```python
import time

def poll_for_completion(job_id, max_wait_time=300, poll_interval=5):
    """
    Poll BDA job until completion or timeout
    """
    start_time = time.time()
    
    while time.time() - start_time < max_wait_time:
        result = get_document_analysis(job_id)
        
        if result['status'] == 'SUCCEEDED':
            return result
        elif result['status'] == 'FAILED':
            raise Exception(f"BDA job failed: {result['error_message']}")
        
        time.sleep(poll_interval)
    
    raise TimeoutError(f"BDA job {job_id} did not complete within {max_wait_time} seconds")
```

### 4.2 Error Handling

```python
from botocore.exceptions import ClientError

def handle_bda_errors(func):
    """
    Decorator for BDA error handling
    """
    def wrapper(*args, **kwargs):
        try:
            return func(*args, **kwargs)
        
        except ClientError as e:
            error_code = e.response['Error']['Code']
            
            if error_code == 'ThrottlingException':
                logger.warning("BDA throttling, implementing backoff")
                time.sleep(2)
                return func(*args, **kwargs)
            
            elif error_code == 'ValidationException':
                logger.error(f"Invalid BDA request: {e}")
                raise ValueError(f"Invalid BDA parameters: {e}")
            
            elif error_code == 'ResourceNotFoundException':
                logger.error(f"BDA resource not found: {e}")
                raise
            
            else:
                logger.error(f"BDA error: {error_code} - {e}")
                raise
    
    return wrapper

@handle_bda_errors
def invoke_bda_with_retry(s3_bucket, s3_key, blueprint_id, project_arn):
    return start_document_analysis(s3_bucket, s3_key, blueprint_id, project_arn)
```

---

## 5. Blueprint Management

### 5.1 Blueprint Lifecycle

**Creation**:
1. Define schema and extraction requirements
2. Create blueprint via BDA console or API
3. Test with sample documents
4. Validate extraction accuracy
5. Deploy to production

**Updates**:
1. Version control for blueprints
2. Test new version with validation set
3. Gradual rollout (canary deployment)
4. Monitor performance metrics

**Deprecation**:
1. Mark blueprint as deprecated
2. Migrate to new blueprint
3. Archive old blueprint

### 5.2 Blueprint Versioning

```python
class BlueprintManager:
    def __init__(self, dynamodb_client, config_bucket):
        self.dynamodb = dynamodb_client
        self.config_bucket = config_bucket
    
    def get_active_blueprint(self, document_type, blueprint_type):
        """
        Get the active blueprint for a document type
        """
        response = self.dynamodb.query(
            TableName='idp-blueprint-registry',
            KeyConditionExpression='document_type = :dt AND blueprint_type = :bt',
            FilterExpression='is_active = :active',
            ExpressionAttributeValues={
                ':dt': document_type,
                ':bt': blueprint_type,
                ':active': True
            }
        )
        
        if response['Items']:
            return response['Items'][0]
        else:
            raise ValueError(f"No active blueprint found for {document_type}/{blueprint_type}")
    
    def create_blueprint_version(self, document_type, blueprint_config):
        """
        Create a new version of a blueprint
        """
        version = self._generate_version()
        
        blueprint_item = {
            'document_type': document_type,
            'blueprint_id': blueprint_config['blueprint_id'],
            'version': version,
            'is_active': False,  # Requires manual activation
            'created_at': datetime.utcnow().isoformat(),
            'config': blueprint_config
        }
        
        self.dynamodb.put_item(
            TableName='idp-blueprint-registry',
            Item=blueprint_item
        )
        
        return version
```

---

## 6. Model Selection

### 6.1 Supported Models

**Anthropic Claude Models** (EU-compliant):
- `anthropic.claude-3-opus-20240229-v1:0` - Highest accuracy, slower
- `anthropic.claude-3-sonnet-20240229-v1:0` - Balanced (recommended)
- `anthropic.claude-3-haiku-20240307-v1:0` - Fastest, lower cost

**Amazon Models**:
- `amazon.titan-text-premier-v1:0` - Good for structured extraction
- `amazon.nova-pro-v1:0` - Latest generation (if available in EU)

### 6.2 Model Selection Strategy

```python
def select_model_for_document(document_type, complexity):
    """
    Select appropriate model based on document complexity
    """
    model_mapping = {
        'simple': {
            'classification': 'anthropic.claude-3-haiku-20240307-v1:0',
            'extraction': 'anthropic.claude-3-haiku-20240307-v1:0'
        },
        'medium': {
            'classification': 'anthropic.claude-3-sonnet-20240229-v1:0',
            'extraction': 'anthropic.claude-3-sonnet-20240229-v1:0'
        },
        'complex': {
            'classification': 'anthropic.claude-3-sonnet-20240229-v1:0',
            'extraction': 'anthropic.claude-3-opus-20240229-v1:0'
        }
    }
    
    return model_mapping.get(complexity, model_mapping['medium'])
```


---

## 7. Confidence Scoring

### 7.1 Confidence Calculation

```python
def calculate_overall_confidence(field_confidences, required_fields):
    """
    Calculate overall extraction confidence
    """
    if not field_confidences:
        return 0.0
    
    # Weight required fields more heavily
    required_confidence = []
    optional_confidence = []
    
    for field, confidence in field_confidences.items():
        if field in required_fields:
            required_confidence.append(confidence)
        else:
            optional_confidence.append(confidence)
    
    # Weighted average: 70% required, 30% optional
    if required_confidence:
        req_avg = sum(required_confidence) / len(required_confidence)
    else:
        req_avg = 0.0
    
    if optional_confidence:
        opt_avg = sum(optional_confidence) / len(optional_confidence)
    else:
        opt_avg = req_avg  # If no optional fields, use required average
    
    overall = (req_avg * 0.7) + (opt_avg * 0.3)
    
    return round(overall, 4)
```

### 7.2 Confidence Thresholds

```python
CONFIDENCE_THRESHOLDS = {
    'classification': {
        'high': 0.90,      # Auto-accept
        'medium': 0.75,    # Accept with review flag
        'low': 0.60        # Reject or manual review
    },
    'extraction': {
        'high': 0.85,      # Auto-accept
        'medium': 0.70,    # Accept with review flag
        'low': 0.55        # Reject or manual review
    }
}

def evaluate_confidence(confidence, operation_type):
    """
    Evaluate confidence level
    """
    thresholds = CONFIDENCE_THRESHOLDS[operation_type]
    
    if confidence >= thresholds['high']:
        return 'HIGH', 'auto_accept'
    elif confidence >= thresholds['medium']:
        return 'MEDIUM', 'review_recommended'
    elif confidence >= thresholds['low']:
        return 'LOW', 'manual_review_required'
    else:
        return 'VERY_LOW', 'reject'
```

---

## 8. Performance Optimization

### 8.1 Batch Processing

```python
def batch_process_documents(document_list, blueprint_id, project_arn):
    """
    Process multiple documents in parallel
    """
    from concurrent.futures import ThreadPoolExecutor, as_completed
    
    results = []
    
    with ThreadPoolExecutor(max_workers=10) as executor:
        futures = {
            executor.submit(
                start_document_analysis,
                doc['bucket'],
                doc['key'],
                blueprint_id,
                project_arn
            ): doc for doc in document_list
        }
        
        for future in as_completed(futures):
            doc = futures[future]
            try:
                result = future.result()
                results.append({
                    'document_id': doc['document_id'],
                    'job_id': result['job_id'],
                    'status': 'SUBMITTED'
                })
            except Exception as e:
                results.append({
                    'document_id': doc['document_id'],
                    'status': 'FAILED',
                    'error': str(e)
                })
    
    return results
```

### 8.2 Caching Strategy

```python
class BDAResultCache:
    """
    Cache BDA results to avoid reprocessing identical documents
    """
    def __init__(self, dynamodb_client):
        self.dynamodb = dynamodb_client
        self.table_name = 'idp-bda-result-cache'
    
    def get_cached_result(self, document_hash, blueprint_id):
        """
        Retrieve cached result if available
        """
        response = self.dynamodb.get_item(
            TableName=self.table_name,
            Key={
                'document_hash': document_hash,
                'blueprint_id': blueprint_id
            }
        )
        
        if 'Item' in response:
            # Check if cache is still valid (e.g., < 30 days old)
            cached_time = datetime.fromisoformat(response['Item']['cached_at'])
            if (datetime.utcnow() - cached_time).days < 30:
                return response['Item']['result']
        
        return None
    
    def cache_result(self, document_hash, blueprint_id, result):
        """
        Cache BDA result
        """
        self.dynamodb.put_item(
            TableName=self.table_name,
            Item={
                'document_hash': document_hash,
                'blueprint_id': blueprint_id,
                'result': result,
                'cached_at': datetime.utcnow().isoformat(),
                'ttl': int((datetime.utcnow() + timedelta(days=30)).timestamp())
            }
        )
```

---

## 9. Multi-Language Support

### 9.1 Language-Specific Prompts

```python
LANGUAGE_PROMPTS = {
    'en': {
        'extraction': 'Extract the following information from this document in English...',
        'classification': 'Classify this document as one of the following types...'
    },
    'es': {
        'extraction': 'Extrae la siguiente información de este documento en español...',
        'classification': 'Clasifica este documento como uno de los siguientes tipos...'
    },
    'fr': {
        'extraction': 'Extrayez les informations suivantes de ce document en français...',
        'classification': 'Classez ce document comme l\'un des types suivants...'
    },
    'de': {
        'extraction': 'Extrahieren Sie die folgenden Informationen aus diesem Dokument auf Deutsch...',
        'classification': 'Klassifizieren Sie dieses Dokument als einen der folgenden Typen...'
    }
}

def get_localized_prompt(language, prompt_type):
    """
    Get language-specific prompt
    """
    return LANGUAGE_PROMPTS.get(language, LANGUAGE_PROMPTS['en'])[prompt_type]
```

### 9.2 Language Detection Integration

```python
def detect_and_process_document(s3_bucket, s3_key, document_type):
    """
    Detect language and use appropriate blueprint
    """
    # Detect language
    language = detect_document_language(s3_bucket, s3_key)
    
    # Get language-specific blueprint
    blueprint_id = get_blueprint_for_language(document_type, language)
    
    # Process with appropriate blueprint
    result = start_document_analysis(
        s3_bucket=s3_bucket,
        s3_key=s3_key,
        blueprint_id=blueprint_id,
        project_arn=PROJECT_ARN
    )
    
    return result
```

---

## 10. Quality Assurance

### 10.1 Validation Rules

```python
class ExtractionValidator:
    """
    Validate extracted data against business rules
    """
    
    def validate_passport(self, extracted_data):
        """
        Validate passport extraction
        """
        errors = []
        
        # Check passport number format
        if 'passport_number' in extracted_data:
            if not re.match(r'^[A-Z0-9]{6,9}$', extracted_data['passport_number']):
                errors.append('Invalid passport number format')
        
        # Check date validity
        if 'date_of_birth' in extracted_data:
            dob = datetime.fromisoformat(extracted_data['date_of_birth'])
            if dob > datetime.now():
                errors.append('Date of birth cannot be in the future')
            if (datetime.now() - dob).days < 365 * 18:
                errors.append('Passport holder must be at least 18 years old')
        
        # Check expiry date
        if 'date_of_expiry' in extracted_data:
            expiry = datetime.fromisoformat(extracted_data['date_of_expiry'])
            if expiry < datetime.now():
                errors.append('Passport has expired')
        
        # Check nationality code
        if 'nationality' in extracted_data:
            if not re.match(r'^[A-Z]{3}$', extracted_data['nationality']):
                errors.append('Invalid nationality code (must be ISO 3166-1 alpha-3)')
        
        return {
            'is_valid': len(errors) == 0,
            'errors': errors
        }
    
    def validate_bank_statement(self, extracted_data):
        """
        Validate bank statement extraction
        """
        errors = []
        
        # Check account number
        if 'account_number' not in extracted_data:
            errors.append('Missing required field: account_number')
        
        # Check statement period
        if 'statement_period' in extracted_data:
            period = extracted_data['statement_period']
            start = datetime.fromisoformat(period['start_date'])
            end = datetime.fromisoformat(period['end_date'])
            
            if start >= end:
                errors.append('Statement start date must be before end date')
            
            if (end - start).days > 90:
                errors.append('Statement period exceeds 90 days')
        
        # Check balance consistency
        if all(k in extracted_data for k in ['opening_balance', 'closing_balance', 'transactions']):
            calculated_balance = extracted_data['opening_balance']
            for txn in extracted_data['transactions']:
                calculated_balance += txn['amount']
            
            if abs(calculated_balance - extracted_data['closing_balance']) > 0.01:
                errors.append('Balance calculation mismatch')
        
        return {
            'is_valid': len(errors) == 0,
            'errors': errors
        }
```

### 10.2 Quality Metrics

```python
class QualityMetrics:
    """
    Track BDA extraction quality metrics
    """
    
    def __init__(self, cloudwatch_client):
        self.cloudwatch = cloudwatch_client
    
    def record_extraction_quality(self, document_type, confidence, is_valid):
        """
        Record quality metrics to CloudWatch
        """
        self.cloudwatch.put_metric_data(
            Namespace='IDP/BDA',
            MetricData=[
                {
                    'MetricName': 'ExtractionConfidence',
                    'Value': confidence,
                    'Unit': 'None',
                    'Dimensions': [
                        {'Name': 'DocumentType', 'Value': document_type}
                    ]
                },
                {
                    'MetricName': 'ValidationSuccess',
                    'Value': 1 if is_valid else 0,
                    'Unit': 'Count',
                    'Dimensions': [
                        {'Name': 'DocumentType', 'Value': document_type}
                    ]
                }
            ]
        )
```

---

## 11. Cost Optimization

### 11.1 Cost Factors

**BDA Pricing Components**:
1. Document analysis requests
2. Pages processed
3. Model inference time
4. Data transfer

### 11.2 Cost Optimization Strategies

```python
class CostOptimizer:
    """
    Optimize BDA usage costs
    """
    
    def select_cost_effective_model(self, document_complexity, accuracy_requirement):
        """
        Select model based on cost vs accuracy tradeoff
        """
        if accuracy_requirement >= 0.95:
            # High accuracy required - use Opus
            return 'anthropic.claude-3-opus-20240229-v1:0'
        elif document_complexity == 'simple' and accuracy_requirement >= 0.85:
            # Simple document - use Haiku (cheapest)
            return 'anthropic.claude-3-haiku-20240307-v1:0'
        else:
            # Default - use Sonnet (balanced)
            return 'anthropic.claude-3-sonnet-20240229-v1:0'
    
    def should_use_cache(self, document_hash):
        """
        Determine if document should use cached results
        """
        # Check if document has been processed before
        cached_result = self.cache.get_cached_result(document_hash)
        
        if cached_result:
            # Use cache to save costs
            return True, cached_result
        
        return False, None
    
    def estimate_processing_cost(self, document_pages, model_id):
        """
        Estimate BDA processing cost
        """
        # Pricing per 1000 pages (example rates)
        pricing = {
            'anthropic.claude-3-opus-20240229-v1:0': 15.00,
            'anthropic.claude-3-sonnet-20240229-v1:0': 3.00,
            'anthropic.claude-3-haiku-20240307-v1:0': 0.25
        }
        
        cost_per_page = pricing.get(model_id, 3.00) / 1000
        estimated_cost = document_pages * cost_per_page
        
        return estimated_cost
```

---

## 12. Monitoring & Troubleshooting

### 12.1 BDA Metrics

```python
def publish_bda_metrics(job_id, status, duration_ms, confidence):
    """
    Publish BDA-specific metrics
    """
    cloudwatch = boto3.client('cloudwatch')
    
    cloudwatch.put_metric_data(
        Namespace='IDP/BDA',
        MetricData=[
            {
                'MetricName': 'ProcessingDuration',
                'Value': duration_ms,
                'Unit': 'Milliseconds'
            },
            {
                'MetricName': 'JobStatus',
                'Value': 1 if status == 'SUCCEEDED' else 0,
                'Unit': 'Count'
            },
            {
                'MetricName': 'Confidence',
                'Value': confidence,
                'Unit': 'None'
            }
        ]
    )
```

### 12.2 Common Issues & Solutions

**Issue 1: Low Confidence Scores**
- **Cause**: Poor document quality, unclear text
- **Solution**: Implement pre-processing (image enhancement, deskewing)

**Issue 2: Throttling Errors**
- **Cause**: Exceeding BDA rate limits
- **Solution**: Implement exponential backoff, request limit increase

**Issue 3: Incorrect Extractions**
- **Cause**: Blueprint not optimized for document variant
- **Solution**: Refine blueprint, add more training examples

**Issue 4: Timeout Errors**
- **Cause**: Large documents, complex processing
- **Solution**: Increase Lambda timeout, split large documents

---

## 13. Testing Strategy

### 13.1 Blueprint Testing

```python
def test_blueprint_accuracy(blueprint_id, test_dataset):
    """
    Test blueprint accuracy against ground truth
    """
    results = []
    
    for test_case in test_dataset:
        # Process document
        extraction = process_with_blueprint(
            test_case['s3_bucket'],
            test_case['s3_key'],
            blueprint_id
        )
        
        # Compare with ground truth
        accuracy = calculate_accuracy(
            extraction['extracted_data'],
            test_case['ground_truth']
        )
        
        results.append({
            'document_id': test_case['document_id'],
            'accuracy': accuracy,
            'confidence': extraction['overall_confidence']
        })
    
    # Calculate aggregate metrics
    avg_accuracy = sum(r['accuracy'] for r in results) / len(results)
    avg_confidence = sum(r['confidence'] for r in results) / len(results)
    
    return {
        'blueprint_id': blueprint_id,
        'test_count': len(results),
        'average_accuracy': avg_accuracy,
        'average_confidence': avg_confidence,
        'pass_rate': sum(1 for r in results if r['accuracy'] >= 0.90) / len(results)
    }
```

### 13.2 Regression Testing

```python
def run_regression_tests(blueprint_id, baseline_results):
    """
    Ensure blueprint changes don't degrade performance
    """
    current_results = test_blueprint_accuracy(blueprint_id, TEST_DATASET)
    
    # Compare with baseline
    accuracy_delta = current_results['average_accuracy'] - baseline_results['average_accuracy']
    
    if accuracy_delta < -0.05:  # 5% degradation threshold
        raise ValueError(f"Blueprint regression detected: accuracy dropped by {accuracy_delta:.2%}")
    
    return {
        'status': 'PASSED',
        'accuracy_delta': accuracy_delta,
        'current_accuracy': current_results['average_accuracy'],
        'baseline_accuracy': baseline_results['average_accuracy']
    }
```

---

## 14. Deployment Checklist

### 14.1 Pre-Deployment

- [ ] All blueprints created and tested
- [ ] Test dataset validated with > 90% accuracy
- [ ] Confidence thresholds configured
- [ ] Model selection finalized
- [ ] Cost estimates reviewed
- [ ] IAM permissions configured
- [ ] Monitoring dashboards created

### 14.2 Post-Deployment

- [ ] Smoke tests passed
- [ ] Metrics flowing to CloudWatch
- [ ] Error rates within acceptable range
- [ ] Confidence scores meeting thresholds
- [ ] Cost tracking enabled
- [ ] Alerts configured

---

**End of BDA LLD**
