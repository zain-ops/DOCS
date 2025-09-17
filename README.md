# Multi-Tenant Loan Application - Complete Architecture Plan

## 🏗️ System Overview

A scalable, serverless loan application system supporting multiple tenants (banks) with dynamic model calculations and automated file processing capabilities.

### Core Features
- **Multi-Tenant Support**: HDFC, SBI, ICICI, etc.
- **Dynamic Models**: Home loan, personal loan, car loan
- **File Processing**: .xls generation and .csv conversion
- **Auto-Scaling**: Handle unlimited tenants and requests

---

## 🎯 Architecture Diagram

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  CloudFront │───▶│ API Gateway │───▶│  Lambda 1   │───▶│     SQS     │
│     +S3     │    │   + Auth    │    │ Calculator  │    │   Queue     │
└─────────────┘    └─────────────┘    └─────────────┘    └─────────────┘
                                              │                    │
                                              ▼                    ▼
                                    ┌─────────────┐      ┌─────────────┐
                                    │ DynamoDB    │      │  Lambda 2   │
                                    │Model Config │      │File Handler │
                                    └─────────────┘      └─────────────┘
                                              │                    │
                                              ▼                    ▼
                                    ┌─────────────┐      ┌─────────────┐
                                    │     S3      │      │EC2 Windows  │
                                    │Model Store  │      │.exe Runner  │
                                    └─────────────┘      └─────────────┘
                                                                   │
                                                                   ▼
                                                         ┌─────────────┐
                                                         │     S3      │
                                                         │.csv Output  │
                                                         └─────────────┘
```

---

## 🔧 Component Details

### 1. Frontend Layer

```yaml
Technology: React.js
Hosting: CloudFront + S3
Features:
  - Tenant selection dropdown
  - Loan type selection (home/personal/car)
  - Form submission with validation
  - Real-time calculation results
  - File download capabilities
```

### 2. API Gateway Configuration

```yaml
Authentication: JWT tokens
Rate Limiting: Per tenant quotas
Headers Required:
  - x-tenant-id: "hdfc_bank"
  - Authorization: "Bearer <jwt>"
  - Content-Type: "application/json"

Endpoints:
  - POST /calculate: Loan calculation
  - GET /models: Available models per tenant
  - GET /results/{requestId}: Get calculation status
  - GET /download/{requestId}: Download converted files
```

### 3. Lambda 1 - Multi-Tenant Calculator

#### WHO & WHAT Identification
```python
def lambda_handler(event, context):
    # WHO: Extract tenant from header
    tenant_id = event['headers']['x-tenant-id']  # "hdfc_bank"

    # WHAT: Extract model from body
    body = json.loads(event['body'])
    model_type = body['modelType']      # "home_loan"
    model_version = body['modelVersion'] # "v2.1"

    # Load tenant-specific configuration
    config = load_tenant_config(tenant_id, model_type, model_version)

    # Load tenant-specific calculator
    calculator = load_calculator_from_s3(tenant_id, model_type, model_version)

    # Execute calculation
    result = calculator.calculate(loan_data, config)
```

#### Dynamic Model Loading Strategy
```yaml
DynamoDB Structure:
  Table: LoanModelRegistry
  Partition Key: tenantId
  Sort Key: modelKey (modelType_version)

S3 Structure:
  loan-models/
  ├── hdfc_bank/
  │   ├── home_loan/v2.1/calculator.py
  │   └── personal_loan/v1.5/calculator.py
  └── sbi_bank/
      ├── home_loan/v1.8/calculator.py
      └── car_loan/v1.2/calculator.py
```

### 4. Configuration Management (DynamoDB)

#### Sample Tenant Configuration
```json
{
  "tenantId": "hdfc_bank",
  "modelKey": "home_loan_v2.1",
  "modelType": "home_loan",
  "version": "v2.1",
  "isActive": true,
  "configuration": {
    "interestRate": 8.5,
    "maxLoanAmount": 10000000,
    "minCreditScore": 650,
    "maxTenure": 25,
    "processingFee": 0.5,
    "incomeMultiplier": 60,
    "formulaType": "advanced"
  },
  "s3Location": {
    "bucket": "loan-models",
    "key": "hdfc_bank/home_loan/v2.1/calculator.py"
  },
  "metadata": {
    "createdAt": "2024-01-15T10:30:00Z",
    "createdBy": "admin@hdfc.com"
  }
}
```

### 5. Sample Calculator Implementation

#### HDFC Bank Calculator
```python
# S3: hdfc_bank/home_loan/v2.1/calculator.py
class LoanCalculator:
    def __init__(self):
        self.bank_name = "HDFC Bank"

    def calculate(self, loan_data, config):
        amount = loan_data['amount']
        income = loan_data['income']
        credit_score = loan_data['creditScore']

        rules = config['configuration']

        # HDFC specific business rules
        max_eligible = income * rules['incomeMultiplier']  # 60x for HDFC

        if credit_score < rules['minCreditScore']:
            return {'approved': False, 'reason': 'Credit score too low'}

        if amount > max_eligible:
            return {'approved': False, 'reason': 'Income insufficient'}

        # HDFC specific EMI calculation
        rate = rules['interestRate'] / (12 * 100)
        tenure = 20 * 12
        emi = amount * rate * ((1 + rate) ** tenure) / (((1 + rate) ** tenure) - 1)

        return {
            'approved': True,
            'bank': self.bank_name,
            'loanAmount': amount,
            'emi': round(emi, 2),
            'interestRate': rules['interestRate'],
            'processingFee': amount * rules['processingFee'] / 100
        }
```

### 6. File Processing Pipeline

#### Lambda 1 - Excel Generation
```python
def create_excel_report(tenant_id, request_id, result, loan_data):
    data = {
        'Field': ['Tenant ID', 'Request ID', 'Loan Amount', 'Income',
                 'Approved', 'EMI', 'Interest Rate'],
        'Value': [tenant_id, request_id, loan_data.get('amount', 0),
                 loan_data.get('income', 0), result.get('approved', False),
                 result.get('emi', 0), result.get('interestRate', 0)]
    }

    df = pd.DataFrame(data)
    excel_buffer = BytesIO()
    with pd.ExcelWriter(excel_buffer, engine='openpyxl') as writer:
        df.to_excel(writer, sheet_name='Loan Details', index=False)

    # Upload to S3
    s3_key = f"excel-reports/{tenant_id}/{request_id}.xls"
    s3.put_object(Bucket='loan-processing-bucket', Key=s3_key, Body=excel_buffer.getvalue())

    # Send to conversion queue
    sqs.send_message(
        QueueUrl='file-conversion-queue',
        MessageBody=json.dumps({
            'requestId': request_id,
            'tenantId': tenant_id,
            'xlsS3Key': s3_key
        })
    )
```

### 7. Windows Conversion System

#### EC2 Windows Auto Scaling Setup
```yaml
Launch Template:
  ImageId: ami-windows-2019
  InstanceType: t3.medium
  KeyName: windows-conversion-key
  IamInstanceProfile: EC2-S3-SQS-Role
  InstanceMarketOptions:
    MarketType: spot
    SpotOptions:
      MaxPrice: "0.05"

Auto Scaling Group:
  MinSize: 1
  MaxSize: 10
  DesiredCapacity: 2
  TargetGroupARNs: []
  HealthCheckType: EC2

CloudWatch Scaling Policy:
  MetricName: ApproximateNumberOfMessages
  Namespace: AWS/SQS
  Threshold: 5 messages
  ScaleUp: +2 instances
  ScaleDown: -1 instance
```

#### PowerShell Conversion Script
```powershell
# Script running on EC2 Windows instances
while ($true) {
    try {
        # Poll SQS for conversion jobs
        $message = aws sqs receive-message --queue-url "https://sqs.region.amazonaws.com/account/file-conversion-queue" --max-number-of-messages 1 | ConvertFrom-Json

        if ($message.Messages) {
            $job = $message.Messages[0].Body | ConvertFrom-Json
            $receiptHandle = $message.Messages[0].ReceiptHandle

            Write-Host "Processing conversion for: $($job.requestId)"

            # Download .xls from S3
            aws s3 cp "s3://loan-processing-bucket/$($job.xlsS3Key)" "C:\temp\$($job.requestId).xls"

            # Run conversion software
            & "C:\tools\ExcelConverter.exe" "C:\temp\$($job.requestId).xls" "C:\temp\$($job.requestId).csv"

            if ($LASTEXITCODE -eq 0) {
                # Upload .csv back to S3
                aws s3 cp "C:\temp\$($job.requestId).csv" "s3://loan-processing-bucket/csv-reports/$($job.tenantId)/$($job.requestId).csv"

                # Delete SQS message
                aws sqs delete-message --queue-url "https://sqs.region.amazonaws.com/account/file-conversion-queue" --receipt-handle $receiptHandle

                Write-Host "Conversion completed successfully"
            } else {
                Write-Error "Conversion failed with exit code: $LASTEXITCODE"
            }

            # Cleanup
            Remove-Item "C:\temp\$($job.requestId).xls", "C:\temp\$($job.requestId).csv" -ErrorAction SilentlyContinue
        }
    }
    catch {
        Write-Error "Error processing message: $_"
    }

    Start-Sleep -Seconds 5
}
```

---

## 📊 Scaling Strategy

### Tenant Scaling
```yaml
Current: Single Lambda handles all tenants
Scaling: Auto-scaling based on:
  - SQS queue depth
  - Lambda concurrency metrics
  - DynamoDB read/write capacity

Cost Optimization:
  - Lambda Provisioned Concurrency for frequent tenants
  - Regular Lambda for sporadic usage
  - S3 Intelligent Tiering for model storage
```

### Model Scaling
```yaml
New Model Addition:
  1. Upload calculator.py to S3
  2. Insert configuration in DynamoDB
  3. Zero downtime deployment
  4. A/B testing support

Version Management:
  - Multiple versions per model
  - Gradual rollout capability
  - Rollback support
```

---

## 🔒 Security & Compliance

### Tenant Isolation
```yaml
Data Isolation:
  - Tenant-specific DynamoDB partitions
  - S3 bucket policies with tenant prefixes
  - IAM roles with tenant-scoped permissions

Network Security:
  - VPC endpoints for internal communication
  - Security groups restricting access
  - Encryption at rest and in transit
```

### Authentication & Authorization
```yaml
API Gateway:
  - JWT token validation
  - Rate limiting per tenant
  - Request/response logging

Lambda Functions:
  - Tenant validation in every request
  - Audit logging for all operations
  - Error handling without data leakage
```

---

## 💰 Cost Optimization

### Infrastructure Costs
```yaml
Lambda:
  - Pay per request model
  - Provisioned concurrency for high-volume tenants
  - Memory optimization based on usage

EC2 Windows:
  - Spot instances (50-90% cost savings)
  - Auto-scaling based on queue depth
  - Schedule-based scaling for predictable loads

Storage:
  - S3 Intelligent Tiering
  - Lifecycle policies for old files
  - DynamoDB On-Demand billing
```

### Monitoring & Alerts
```yaml
CloudWatch Metrics:
  - Lambda execution time and errors
  - SQS queue depth and processing time
  - EC2 instance utilization
  - Cost anomaly detection

Alarms:
  - High error rates
  - Processing delays
  - Cost threshold breaches
  - Security violations
```

---

## 🚀 Deployment Strategy

### Phase 1: Core Infrastructure
- [ ] Set up DynamoDB table for model registry
- [ ] Create S3 buckets for model storage and file processing
- [ ] Deploy Lambda 1 (Calculator) with basic functionality
- [ ] Configure API Gateway with authentication

### Phase 2: Multi-Tenant Implementation
- [ ] Implement dynamic model loading
- [ ] Create sample tenant configurations
- [ ] Deploy tenant-specific calculator models
- [ ] Add comprehensive error handling

### Phase 3: File Processing Pipeline
- [ ] Deploy Lambda 2 (File Handler)
- [ ] Set up SQS queues for async processing
- [ ] Configure EC2 Windows Auto Scaling Group
- [ ] Implement PowerShell conversion scripts

### Phase 4: Production Readiness
- [ ] Add monitoring and alerting
- [ ] Implement security best practices
- [ ] Performance testing and optimization
- [ ] Documentation and training

---

## 🔍 Monitoring & Troubleshooting

### Key Metrics
```yaml
Business Metrics:
  - Loan applications processed per day
  - Average processing time per tenant
  - Conversion success rate
  - File processing throughput

Technical Metrics:
  - Lambda cold starts and execution time
  - DynamoDB read/write latency
  - S3 upload/download speeds
  - EC2 instance health and utilization
```

### Common Issues & Solutions
```yaml
Model Loading Failures:
  - Check S3 bucket permissions
  - Verify model file syntax
  - Review DynamoDB configuration

File Conversion Delays:
  - Monitor SQS queue depth
  - Check EC2 instance availability
  - Verify Windows software licensing

High Costs:
  - Review Lambda memory allocation
  - Optimize EC2 instance types
  - Implement caching strategies
```

## 🎯 Success Criteria

### Performance Targets
- **Response Time**: < 2 seconds for loan calculations
- **Throughput**: 1000+ requests per minute
- **Availability**: 99.9% uptime
- **File Processing**: < 5 minutes for .xls to .csv conversion

### Business Metrics
- **Cost Efficiency**: 40% reduction compared to traditional architecture
- **Scalability**: Support 100+ tenants without infrastructure changes
- **Time to Market**: New tenant onboarding in < 1 day
- **Flexibility**: New model deployment in < 30 minutes

---

*This architecture provides a robust, scalable, and cost-effective solution for multi-tenant loan processing with automated file conversion capabilities.*
