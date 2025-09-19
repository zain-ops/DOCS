# Windows-based executable Processing using Ec2 + PowerShell

### 1. Windows Conversion System

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


*This document provides a robust, scalable, and cost-effective solution for windows-based automated file conversion capabilities.*
