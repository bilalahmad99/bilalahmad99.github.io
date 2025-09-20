---
layout: default
---

## Central Logging in AWS Multi-Account Organizations

A comprehensive guide to implementing centralized logging across AWS organizations using Control Tower, log archive accounts, and Microsoft Sentinel for security monitoring and alerting.

### Multi-Account Logging Architecture

```
┌─────────────────┐    ┌─────────────────┐    ┌─────────────────┐
│   Production    │    │    Staging     │    │   Development  │
│     Account     │    │    Account     │    │     Account    │
└─────────────────┘    └─────────────────┘    └─────────────────┘
         │                       │                       │
         └───────────────────────┼───────────────────────┘
                                 │
                    ┌─────────────────┐
                    │  Log Archive    │
                    │    Account      │
                    └─────────────────┘
                                 │
                    ┌─────────────────┐
                    │ Microsoft       │
                    │   Sentinel      │
                    └─────────────────┘
```

### AWS Control Tower Setup

#### Landing Zone Configuration
```hcl
# control-tower-config.tf
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

# Central logging S3 bucket
resource "aws_s3_bucket" "central_logs" {
  bucket = "aws-controltower-logs-${var.account_id}"
  
  tags = {
    Name        = "Central Logs Archive"
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

# S3 bucket versioning
resource "aws_s3_bucket_versioning" "central_logs" {
  bucket = aws_s3_bucket.central_logs.id
  versioning_configuration {
    status = "Enabled"
  }
}

# S3 bucket lifecycle configuration
resource "aws_s3_bucket_lifecycle_configuration" "central_logs" {
  bucket = aws_s3_bucket.central_logs.id

  rule {
    id     = "log_retention"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }
  }
}

# CloudWatch Logs destination
resource "aws_cloudwatch_log_destination" "central_logs" {
  name       = "centralized-logs"
  role_arn   = aws_iam_role.cloudwatch_logs_role.arn
  target_arn = aws_kinesis_stream.log_stream.arn
}

# Variables
variable "account_id" {
  description = "AWS Account ID"
  type        = string
}
```

#### Control Tower Customization
```hcl
# control-tower-customization.tf
# IAM role for CloudWatch Logs
resource "aws_iam_role" "cloudwatch_logs_role" {
  name = "cloudwatch-logs-central-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "logs.amazonaws.com"
        }
      }
    ]
  })
}

# IAM policy for CloudWatch Logs
resource "aws_iam_role_policy" "cloudwatch_logs_policy" {
  name = "cloudwatch-logs-central-policy"
  role = aws_iam_role.cloudwatch_logs_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "kinesis:PutRecord",
          "kinesis:PutRecords"
        ]
        Resource = aws_kinesis_stream.log_stream.arn
      }
    ]
  })
}

# Kinesis stream for log aggregation
resource "aws_kinesis_stream" "log_stream" {
  name             = "centralized-logs-stream"
  shard_count      = 1
  retention_period = 24

  shard_level_metrics = [
    "IncomingRecords",
    "OutgoingRecords",
  ]

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

# CloudWatch Log Group for centralized logs
resource "aws_cloudwatch_log_group" "central_logs" {
  name              = "/aws/centralized/logs"
  retention_in_days = 30

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}
```

### Log Archive Account Configuration

#### S3 Bucket Policy for Centralized Logs
```hcl
# s3-bucket-policy.tf
resource "aws_s3_bucket_policy" "central_logs_policy" {
  bucket = aws_s3_bucket.central_logs.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Sid    = "AllowCloudTrailLogging"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action = "s3:PutObject"
        Resource = "${aws_s3_bucket.central_logs.arn}/cloudtrail/*"
        Condition = {
          StringEquals = {
            "s3:x-amz-acl" = "bucket-owner-full-control"
          }
        }
      },
      {
        Sid    = "AllowCloudTrailBucketAccess"
        Effect = "Allow"
        Principal = {
          Service = "cloudtrail.amazonaws.com"
        }
        Action   = "s3:GetBucketAcl"
        Resource = aws_s3_bucket.central_logs.arn
      },
      {
        Sid    = "AllowCrossAccountLogging"
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::111111111111:root",
            "arn:aws:iam::222222222222:root",
            "arn:aws:iam::333333333333:root"
          ]
        }
        Action = [
          "s3:PutObject",
          "s3:PutObjectAcl"
        ]
        Resource = "${aws_s3_bucket.central_logs.arn}/*"
      }
    ]
  })
}
```

#### CloudWatch Logs Destination
```hcl
# cloudwatch-logs-destination.tf
resource "aws_cloudwatch_log_destination_policy" "central_logs_policy" {
  destination_name = aws_cloudwatch_log_destination.central_logs.name

  access_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::111111111111:root",
            "arn:aws:iam::222222222222:root",
            "arn:aws:iam::333333333333:root"
          ]
        }
        Action   = "logs:PutSubscriptionFilter"
        Resource = aws_cloudwatch_log_destination.central_logs.arn
      }
    ]
  })
}

# Subscription filter for cross-account log forwarding
resource "aws_cloudwatch_log_subscription_filter" "central_logs_filter" {
  count = length(var.source_accounts)

  name            = "central-logs-filter-${count.index}"
  log_group_name  = "/aws/cloudtrail"
  destination_arn = aws_cloudwatch_log_destination.central_logs.arn
  filter_pattern  = ""
  role_arn        = aws_iam_role.log_subscription_role.arn
}

# Variables for source accounts
variable "source_accounts" {
  description = "List of source account IDs for log forwarding"
  type        = list(string)
  default     = ["111111111111", "222222222222", "333333333333"]
}
```

### CloudTrail Configuration

#### Organization-Wide CloudTrail
```hcl
# organization-cloudtrail.tf
resource "aws_cloudtrail" "organization_trail" {
  name                          = "OrganizationCloudTrail"
  s3_bucket_name                = aws_s3_bucket.central_logs.id
  s3_key_prefix                 = "cloudtrail/"
  include_global_service_events = true
  is_multi_region_trail        = true
  enable_log_file_validation    = true

  event_selector {
    read_write_type                 = "All"
    include_management_events      = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::*/*"]
    }
    
    data_resource {
      type   = "AWS::S3::Bucket"
      values = ["arn:aws:s3:::*"]
    }
  }

  event_selector {
    read_write_type                 = "All"
    include_management_events      = true
    
    data_resource {
      type   = "AWS::IAM::User"
      values = ["arn:aws:iam::*:user/*"]
    }
    
    data_resource {
      type   = "AWS::IAM::Role"
      values = ["arn:aws:iam::*:role/*"]
    }
  }

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

# CloudTrail log group
resource "aws_cloudwatch_log_group" "cloudtrail_logs" {
  name              = "/aws/cloudtrail"
  retention_in_days = 30

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

# CloudTrail log stream
resource "aws_cloudwatch_log_stream" "cloudtrail_log_stream" {
  name           = "cloudtrail-log-stream"
  log_group_name = aws_cloudwatch_log_group.cloudtrail_logs.name
}
```

#### CloudTrail Event Filtering
```hcl
# cloudtrail-event-selectors.tf
resource "aws_cloudtrail_event_selector" "sensitive_operations" {
  trail_name = aws_cloudtrail.organization_trail.name

  event_selector {
    read_write_type                 = "All"
    include_management_events      = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::*/*"]
    }
    
    data_resource {
      type   = "AWS::IAM::User"
      values = ["arn:aws:iam::*:user/*"]
    }
    
    data_resource {
      type   = "AWS::IAM::Role"
      values = ["arn:aws:iam::*:role/*"]
    }
    
    data_resource {
      type   = "AWS::KMS::Key"
      values = ["arn:aws:kms:*:*:key/*"]
    }
  }
}

# Additional event selector for Lambda functions
resource "aws_cloudtrail_event_selector" "lambda_operations" {
  trail_name = aws_cloudtrail.organization_trail.name

  event_selector {
    read_write_type                 = "All"
    include_management_events      = true
    
    data_resource {
      type   = "AWS::Lambda::Function"
      values = ["arn:aws:lambda:*:*:function/*"]
    }
  }
}
```

### Microsoft Sentinel Integration

#### Sentinel Data Connector Configuration
```hcl
# sentinel-data-connector.tf
# IAM user for Sentinel integration
resource "aws_iam_user" "sentinel_user" {
  name = "sentinel-integration-user"
  path = "/"

  tags = {
    Environment = "production"
    Purpose     = "sentinel-integration"
  }
}

# IAM access key for Sentinel
resource "aws_iam_access_key" "sentinel_key" {
  user = aws_iam_user.sentinel_user.name
}

# IAM policy for Sentinel S3 access
resource "aws_iam_user_policy" "sentinel_s3_policy" {
  name = "sentinel-s3-policy"
  user = aws_iam_user.sentinel_user.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:GetObject",
          "s3:ListBucket"
        ]
        Resource = [
          aws_s3_bucket.central_logs.arn,
          "${aws_s3_bucket.central_logs.arn}/*"
        ]
      }
    ]
  })
}

# IAM policy for Sentinel CloudTrail access
resource "aws_iam_user_policy" "sentinel_cloudtrail_policy" {
  name = "sentinel-cloudtrail-policy"
  user = aws_iam_user.sentinel_user.name

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "cloudtrail:GetTrail",
          "cloudtrail:DescribeTrails",
          "cloudtrail:GetEventSelectors"
        ]
        Resource = aws_cloudtrail.organization_trail.arn
      }
    ]
  })
}

# Output values for Sentinel configuration
output "sentinel_access_key_id" {
  value       = aws_iam_access_key.sentinel_key.id
  description = "AWS Access Key ID for Sentinel integration"
  sensitive   = true
}

output "sentinel_secret_access_key" {
  value       = aws_iam_access_key.sentinel_key.secret
  description = "AWS Secret Access Key for Sentinel integration"
  sensitive   = true
}

output "s3_bucket_name" {
  value       = aws_s3_bucket.central_logs.bucket
  description = "S3 bucket name for CloudTrail logs"
}
```

#### Sentinel Analytics Rules
```hcl
# sentinel-analytics-rules.tf
# Note: These are KQL queries for Sentinel analytics rules
# The actual Sentinel rules would be configured in the Azure portal

# Local file for storing KQL queries
resource "local_file" "sentinel_queries" {
  filename = "sentinel-analytics-queries.kql"
  content = <<-EOT
    // Suspicious AWS Console Login
    AWSCloudTrail
    | where EventName == "ConsoleLogin"
    | where UserIdentityType == "Root"
    | where SourceIPAddress !in (dynamic(['192.168.1.0/24', '10.0.0.0/8']))
    | summarize count() by bin(TimeGenerated, 1h), SourceIPAddress, UserIdentityUserName
    | where count_ > 5

    // AWS Privilege Escalation
    AWSCloudTrail
    | where EventName in ("AttachUserPolicy", "PutRolePolicy", "CreateRole")
    | where UserIdentityType == "AssumedRole"
    | summarize count() by bin(TimeGenerated, 1h), UserIdentityUserName, EventName
    | where count_ > 3

    // S3 Bucket Tampering
    AWSCloudTrail
    | where EventName in ("PutBucketAcl", "PutBucketPolicy", "DeleteBucket")
    | where UserIdentityType == "AssumedRole"
    | where UserIdentityUserName !in (dynamic(['admin', 'deployment']))
    | summarize count() by bin(TimeGenerated, 1h), UserIdentityUserName, EventName
    | where count_ > 1

    // Cross-Account Activity Detection
    AWSCloudTrail
    | where EventName == "AssumeRole"
    | where UserIdentityType == "AssumedRole"
    | where UserIdentityUserName contains "cross-account"
    | summarize count() by bin(TimeGenerated, 1h), UserIdentityUserName, SourceIPAddress
    | where count_ > 10

    // Failed Authentication Attempts
    AWSCloudTrail
    | where EventName == "ConsoleLogin"
    | where ResponseElements.LoginResult == "Failure"
    | where UserIdentityType == "Root"
    | summarize count() by bin(TimeGenerated, 1h), SourceIPAddress, UserIdentityUserName
    | where count_ > 3
  EOT
}

# Output the queries for manual Sentinel configuration
output "sentinel_queries_file" {
  value       = local_file.sentinel_queries.filename
  description = "Path to KQL queries file for Sentinel analytics rules"
}
```

### Advanced Alerting Scenarios

#### Cross-Account Activity Detection
```kql
// Cross-account activity detection
AWSCloudTrail
| where EventName == "AssumeRole"
| where UserIdentityType == "AssumedRole"
| where UserIdentityUserName contains "cross-account"
| summarize count() by bin(TimeGenerated, 1h), UserIdentityUserName, SourceIPAddress
| where count_ > 10
| project TimeGenerated, UserIdentityUserName, SourceIPAddress, count_
```

#### Unusual API Usage Patterns
```kql
// Unusual API usage patterns
AWSCloudTrail
| where EventName in ("CreateUser", "CreateRole", "AttachUserPolicy")
| where UserIdentityType == "AssumedRole"
| where TimeGenerated > ago(24h)
| summarize count() by bin(TimeGenerated, 1h), UserIdentityUserName, EventName
| where count_ > 5
| project TimeGenerated, UserIdentityUserName, EventName, count_
```

#### Failed Authentication Attempts
```kql
// Failed authentication attempts
AWSCloudTrail
| where EventName == "ConsoleLogin"
| where ResponseElements.LoginResult == "Failure"
| where UserIdentityType == "Root"
| summarize count() by bin(TimeGenerated, 1h), SourceIPAddress, UserIdentityUserName
| where count_ > 3
| project TimeGenerated, SourceIPAddress, UserIdentityUserName, count_
```

### Automation and Response

#### Sentinel Playbooks
```hcl
# sentinel-playbooks.tf
# Note: Sentinel playbooks are configured in Azure portal
# This file contains the playbook logic for reference

resource "local_file" "sentinel_playbooks" {
  filename = "sentinel-playbook-logic.json"
  content = jsonencode({
    displayName = "AWS Incident Response"
    description = "Automated response to AWS security incidents"
    triggers = [
      {
        condition = "AWSCloudTrail | where EventName == 'ConsoleLogin' | where UserIdentityType == 'Root'"
      }
    ]
    actions = [
      {
        type = "SendEmail"
        parameters = {
          to      = "security-team@company.com"
          subject = "AWS Security Alert: Root Login Detected"
          body    = "Root login detected at {{TimeGenerated}} from {{SourceIPAddress}}"
        }
      },
      {
        type = "CreateIncident"
        parameters = {
          title       = "AWS Root Login Alert"
          severity    = "High"
          description = "Root login detected from {{SourceIPAddress}}"
        }
      },
      {
        type = "InvokeWebhook"
        parameters = {
          url    = "https://hooks.slack.com/services/YOUR/SLACK/WEBHOOK"
          method = "POST"
          body   = jsonencode({
            text = "🚨 AWS Security Alert: Root login detected from {{SourceIPAddress}}"
          })
        }
      }
    ]
  })
}

# Output playbook configuration
output "sentinel_playbook_file" {
  value       = local_file.sentinel_playbooks.filename
  description = "Path to Sentinel playbook configuration file"
}
```

#### AWS Lambda Response Function
```hcl
# lambda-response.tf
# Lambda function for security event response
resource "aws_lambda_function" "security_response" {
  filename         = "lambda-response.zip"
  function_name    = "security-event-response"
  role            = aws_iam_role.lambda_role.arn
  handler         = "lambda-response.lambda_handler"
  runtime         = "python3.9"
  timeout         = 30

  environment {
    variables = {
      SNS_TOPIC_ARN = aws_sns_topic.security_alerts.arn
      LOG_LEVEL     = "INFO"
    }
  }

  tags = {
    Environment = "production"
    Purpose     = "security-response"
  }
}

# IAM role for Lambda function
resource "aws_iam_role" "lambda_role" {
  name = "security-response-lambda-role"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Action = "sts:AssumeRole"
        Effect = "Allow"
        Principal = {
          Service = "lambda.amazonaws.com"
        }
      }
    ]
  })
}

# IAM policy for Lambda function
resource "aws_iam_role_policy" "lambda_policy" {
  name = "security-response-lambda-policy"
  role = aws_iam_role.lambda_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "arn:aws:logs:*:*:*"
      },
      {
        Effect = "Allow"
        Action = [
          "sns:Publish"
        ]
        Resource = aws_sns_topic.security_alerts.arn
      },
      {
        Effect = "Allow"
        Action = [
          "cloudwatch:PutMetricAlarm",
          "cloudwatch:PutMetricData"
        ]
        Resource = "*"
      }
    ]
  })
}

# SNS topic for security alerts
resource "aws_sns_topic" "security_alerts" {
  name = "security-alerts"

  tags = {
    Environment = "production"
    Purpose     = "security-alerts"
  }
}

# SNS topic subscription
resource "aws_sns_topic_subscription" "security_alerts_email" {
  topic_arn = aws_sns_topic.security_alerts.arn
  protocol  = "email"
  endpoint   = "security-team@company.com"
}

# Lambda function code
resource "local_file" "lambda_code" {
  filename = "lambda-response.py"
  content = <<-EOT
import json
import boto3
import logging
import os

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """
    AWS Lambda function to respond to security events
    """
    try:
        # Parse the event from Sentinel
        event_data = json.loads(event['body'])
        
        # Extract relevant information
        event_name = event_data.get('EventName')
        user_identity = event_data.get('UserIdentity', {})
        source_ip = event_data.get('SourceIPAddress')
        
        # Determine response based on event type
        if event_name == 'ConsoleLogin' and user_identity.get('type') == 'Root':
            response = handle_root_login(source_ip, event_data)
        elif event_name in ['CreateUser', 'CreateRole', 'AttachUserPolicy']:
            response = handle_privilege_escalation(event_data)
        else:
            response = handle_generic_event(event_data)
        
        return {
            'statusCode': 200,
            'body': json.dumps(response)
        }
        
    except Exception as e:
        logger.error(f"Error processing event: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': str(e)})
        }

def handle_root_login(source_ip, event_data):
    """Handle root login events"""
    sns = boto3.client('sns')
    sns.publish(
        TopicArn=os.environ['SNS_TOPIC_ARN'],
        Message=f"Root login detected from {source_ip}",
        Subject="AWS Security Alert: Root Login"
    )
    
    logger.info(f"Root login detected from {source_ip}")
    
    return {
        'action': 'root_login_detected',
        'source_ip': source_ip,
        'timestamp': event_data.get('EventTime')
    }

def handle_privilege_escalation(event_data):
    """Handle privilege escalation events"""
    cloudwatch = boto3.client('cloudwatch')
    cloudwatch.put_metric_alarm(
        AlarmName='PrivilegeEscalationDetected',
        ComparisonOperator='GreaterThanThreshold',
        EvaluationPeriods=1,
        MetricName='PrivilegeEscalationEvents',
        Namespace='AWS/Security',
        Period=300,
        Statistic='Sum',
        Threshold=1.0,
        ActionsEnabled=True,
        AlarmActions=[os.environ['SNS_TOPIC_ARN']]
    )
    
    return {
        'action': 'privilege_escalation_detected',
        'event_name': event_data.get('EventName'),
        'timestamp': event_data.get('EventTime')
    }

def handle_generic_event(event_data):
    """Handle generic security events"""
    logger.info(f"Generic security event: {event_data.get('EventName')}")
    
    return {
        'action': 'generic_event_logged',
        'event_name': event_data.get('EventName'),
        'timestamp': event_data.get('EventTime')
    }
  EOT
}
```

### Cost Optimization Strategies

#### S3 Lifecycle Policies
```hcl
# s3-lifecycle-policies.tf
resource "aws_s3_bucket_lifecycle_configuration" "central_logs_lifecycle" {
  bucket = aws_s3_bucket.central_logs.id

  rule {
    id     = "CloudTrailLogRetention"
    status = "Enabled"

    transition {
      days          = 30
      storage_class = "STANDARD_IA"
    }

    transition {
      days          = 90
      storage_class = "GLACIER"
    }

    transition {
      days          = 365
      storage_class = "DEEP_ARCHIVE"
    }

    expiration {
      days = 2555  # 7 years retention
    }
  }

  rule {
    id     = "IncompleteMultipartUploadCleanup"
    status = "Enabled"

    abort_incomplete_multipart_upload {
      days_after_initiation = 7
    }
  }
}
```

#### CloudWatch Logs Retention
```hcl
# cloudwatch-logs-retention.tf
resource "aws_cloudwatch_log_group" "cloudtrail_logs" {
  name              = "/aws/cloudtrail"
  retention_in_days = 30

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

resource "aws_cloudwatch_log_group" "vpc_flow_logs" {
  name              = "/aws/vpc/flowlogs"
  retention_in_days = 14

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

resource "aws_cloudwatch_log_group" "security_hub_logs" {
  name              = "/aws/securityhub"
  retention_in_days = 90

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}
```

### Security Best Practices

#### IAM Roles for Cross-Account Access
```hcl
# iam-cross-account-roles.tf
resource "aws_iam_role" "cross_account_logging_role" {
  name = "CrossAccountLoggingRole"

  assume_role_policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Principal = {
          AWS = [
            "arn:aws:iam::111111111111:root",
            "arn:aws:iam::222222222222:root",
            "arn:aws:iam::333333333333:root"
          ]
        }
        Action = "sts:AssumeRole"
        Condition = {
          StringEquals = {
            "sts:ExternalId" = "unique-external-id-${random_string.external_id.result}"
          }
        }
      }
    ]
  })

  tags = {
    Environment = "production"
    Purpose     = "cross-account-logging"
  }
}

# Random external ID for security
resource "random_string" "external_id" {
  length  = 16
  special = false
  upper   = true
}

# IAM policy for cross-account logging
resource "aws_iam_role_policy" "cross_account_logging_policy" {
  name = "CrossAccountLoggingPolicy"
  role = aws_iam_role.cross_account_logging_role.id

  policy = jsonencode({
    Version = "2012-10-17"
    Statement = [
      {
        Effect = "Allow"
        Action = [
          "s3:PutObject",
          "s3:PutObjectAcl"
        ]
        Resource = "${aws_s3_bucket.central_logs.arn}/*"
      },
      {
        Effect = "Allow"
        Action = [
          "logs:CreateLogGroup",
          "logs:CreateLogStream",
          "logs:PutLogEvents"
        ]
        Resource = "${aws_cloudwatch_log_group.central_logs.arn}:*"
      }
    ]
  })
}
```

#### Encryption Configuration
```hcl
# encryption-config.tf
# KMS key for encryption
resource "aws_kms_key" "central_logs_key" {
  description             = "KMS key for central logs encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

resource "aws_kms_alias" "central_logs_key_alias" {
  name          = "alias/central-logs-key"
  target_key_id = aws_kms_key.central_logs_key.key_id
}

# S3 bucket encryption
resource "aws_s3_bucket_server_side_encryption_configuration" "central_logs_encryption" {
  bucket = aws_s3_bucket.central_logs.id

  rule {
    apply_server_side_encryption_by_default {
      kms_master_key_id = aws_kms_key.central_logs_key.arn
      sse_algorithm     = "aws:kms"
    }
    bucket_key_enabled = true
  }
}

# CloudTrail encryption
resource "aws_cloudtrail" "organization_trail" {
  name                          = "OrganizationCloudTrail"
  s3_bucket_name                = aws_s3_bucket.central_logs.id
  s3_key_prefix                 = "cloudtrail/"
  include_global_service_events = true
  is_multi_region_trail        = true
  enable_log_file_validation    = true
  kms_key_id                    = aws_kms_key.central_logs_key.arn

  event_selector {
    read_write_type                 = "All"
    include_management_events      = true
    
    data_resource {
      type   = "AWS::S3::Object"
      values = ["arn:aws:s3:::*/*"]
    }
  }

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

# CloudWatch Logs encryption
resource "aws_cloudwatch_log_group" "central_logs" {
  name              = "/aws/centralized/logs"
  retention_in_days = 30
  kms_key_id        = aws_kms_key.central_logs_key.arn

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}
```

### Monitoring and Alerting

#### CloudWatch Alarms
```hcl
# cloudwatch-alarms.tf
# CloudTrail logging alarm
resource "aws_cloudwatch_metric_alarm" "cloudtrail_logging_disabled" {
  alarm_name          = "CloudTrailLoggingDisabled"
  comparison_operator = "LessThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "CloudTrailLoggingEnabled"
  namespace           = "AWS/CloudTrail"
  period              = "300"
  statistic           = "Sum"
  threshold           = "1.0"
  alarm_description   = "This metric monitors CloudTrail logging"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

# S3 bucket access alarm
resource "aws_cloudwatch_metric_alarm" "s3_bucket_access" {
  alarm_name          = "S3BucketUnauthorizedAccess"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "BucketAccessCount"
  namespace           = "AWS/S3"
  period              = "300"
  statistic           = "Sum"
  threshold           = "10.0"
  alarm_description   = "This metric monitors unauthorized S3 bucket access"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]

  dimensions = {
    BucketName = aws_s3_bucket.central_logs.bucket
  }

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}

# Lambda function error alarm
resource "aws_cloudwatch_metric_alarm" "lambda_function_errors" {
  alarm_name          = "LambdaFunctionErrors"
  comparison_operator = "GreaterThanThreshold"
  evaluation_periods  = "1"
  metric_name         = "Errors"
  namespace           = "AWS/Lambda"
  period              = "300"
  statistic           = "Sum"
  threshold           = "5.0"
  alarm_description   = "This metric monitors Lambda function errors"
  alarm_actions       = [aws_sns_topic.security_alerts.arn]

  dimensions = {
    FunctionName = aws_lambda_function.security_response.function_name
  }

  tags = {
    Environment = "production"
    Purpose     = "centralized-logging"
  }
}
```

### Troubleshooting Common Issues

#### CloudTrail Not Logging
```bash
# Check CloudTrail status
aws cloudtrail describe-trails --trail-name-list OrganizationCloudTrail

# Verify S3 bucket permissions
aws s3api get-bucket-policy --bucket centralized-logs-123456789012

# Check CloudWatch Logs
aws logs describe-log-groups --log-group-name-prefix /aws/cloudtrail
```

#### Sentinel Data Ingestion Issues
```bash
# Check Sentinel data connector status
az sentinel data-connector show --resource-group rg-sentinel --workspace-name sentinel-ws --data-connector-id aws-cloudtrail

# Verify AWS credentials
aws sts get-caller-identity

# Check S3 bucket access
aws s3 ls s3://centralized-logs-123456789012/cloudtrail/
```

### Lessons Learned from Production

#### 1. Log Volume Management
**Lesson**: CloudTrail logs can generate massive volumes - plan storage and costs accordingly.

```yaml
# Log volume estimation
estimated-daily-logs: "50GB"
retention-period: "90 days"
total-storage-needed: "4.5TB"
monthly-cost: "$200"
```

#### 2. Alert Fatigue Prevention
**Lesson**: Too many alerts can overwhelm security teams - use intelligent filtering.

```kql
// Intelligent alert filtering
AWSCloudTrail
| where EventName == "ConsoleLogin"
| where UserIdentityType == "Root"
| where SourceIPAddress in (dynamic(['192.168.1.0/24', '10.0.0.0/8']))
| summarize count() by bin(TimeGenerated, 1h), SourceIPAddress
| where count_ > 10  // Only alert on high frequency
```

#### 3. Cross-Account Complexity
**Lesson**: Cross-account logging adds complexity - document all dependencies.

```yaml
# Cross-account dependencies
dependencies:
  - source-account: "111111111111"
    target-account: "123456789012"
    service: "CloudTrail"
    permission: "s3:PutObject"
  - source-account: "222222222222"
    target-account: "123456789012"
    service: "CloudWatch Logs"
    permission: "logs:PutSubscriptionFilter"
```

#### 4. Data Privacy Compliance
**Lesson**: Ensure logging practices comply with data privacy regulations.

```yaml
# Data privacy compliance
compliance-requirements:
  - gdpr: "Data retention limits"
  - hipaa: "Encryption requirements"
  - sox: "Audit trail requirements"
  - pci-dss: "Access logging requirements"
```

### Best Practices Summary

1. **Start with Control Tower**: Use AWS Control Tower for consistent multi-account setup
2. **Centralize Logging**: Implement log archive account for centralized storage
3. **Enable CloudTrail**: Configure organization-wide CloudTrail with proper event selection
4. **Use Sentinel**: Integrate Microsoft Sentinel for advanced analytics and alerting
5. **Implement Automation**: Use playbooks and Lambda functions for automated response
6. **Monitor Costs**: Set up lifecycle policies and retention rules
7. **Test Regularly**: Regularly test alerting and response procedures
8. **Document Everything**: Maintain comprehensive documentation of the logging architecture

[back](../)
