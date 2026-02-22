# AWS-Control-Policies

## Key Considerations

### 1. **Environment Types**
| Environment | Notes |
|---|---|
| **AWS GovCloud (US)** | Kinesis Data Streams & Firehose available |
| **AWS Outposts** | Limited Kinesis support; check service availability |
| **AWS Snow Family** | Not natively supported |
| **Isolated/On-Prem (SCIF, etc.)** | Requires alternative architectures |

### 2. **Service Components**
- **Kinesis Data Streams (KDS)** – Real-time data ingestion
- **Kinesis Data Firehose** – Delivery to destinations (S3, Redshift, etc.)
- **Kinesis Data Analytics** – Real-time processing
- **Kinesis Video Streams** – Video ingestion/processing

---

## Air-Gap Enablement Steps

### Network & Endpoint Configuration
```plaintext
1. VPC Endpoints (PrivateLink)
   - com.amazonaws.<region>.kinesis-streams
   - com.amazonaws.<region>.kinesis-firehose
   
2. DNS Resolution
   - Private hosted zones for Kinesis endpoints
   
3. Security Groups / NACLs
   - Allow HTTPS (443) to VPC endpoints
```

### IAM & Authentication
```json
{
  "Effect": "Allow",
  "Action": [
    "kinesis:PutRecord",
    "kinesis:PutRecords",
    "kinesis:GetRecords",
    "kinesis:GetShardIterator",
    "kinesis:DescribeStream",
    "kinesis:ListStreams"
  ],
  "Resource": "arn:aws:kinesis:<region>:<account>:stream/*"
}
```

### Encryption (Required for Air-Gap)
- **At rest**: AWS KMS (CMK) encryption on streams
- **In transit**: TLS 1.2+ enforced
- **KMS VPC Endpoint**: `com.amazonaws.<region>.kms`

---

## Air-Gap Alternative Architectures

If Kinesis is **not available** in your air-gapped environment:

### Option A: Apache Kafka (Self-Managed)
```
Producers → Kafka Cluster → Consumers
         (on-prem / isolated VPC)
```
- **Amazon MSK** if available in region
- Open-source Kafka on EC2/containers

### Option B: Redis Streams
```
Producers → Redis Cluster → Consumers
```

### Option C: Apache Pulsar / NATS
- Lightweight messaging alternatives

---

## Deployment Checklist

- [ ] Verify Kinesis service availability in target partition/region
- [ ] Create VPC Interface Endpoints for Kinesis + KMS
- [ ] Configure private DNS resolution
- [ ] Set up KMS CMK for stream encryption
- [ ] Configure IAM roles with least-privilege
- [ ] Deploy producers/consumers with SDK configured for VPC endpoints
- [ ] Validate no internet-bound traffic (VPC Flow Logs)
- [ ] Set up CloudWatch VPC Endpoint for monitoring
- [ ] Test failover and stream scaling
- [ ] Conduct security review / ATO documentation

## SDK Configuration (Air-Gap Endpoint Override)

```python
import boto3

client = boto3.client(
    'kinesis',
    region_name='us-gov-west-1',
    endpoint_url='https://vpce-0abc123-kinesis.us-gov-west-1.vpce.amazonaws.com',
    verify='/path/to/custom-ca-bundle.pem'
)

response = client.put_record(
    StreamName='my-airgap-stream',
    Data=b'payload',
    PartitionKey='key1'
)
```

---

## Monitoring in Air-Gap

- **CloudWatch** (via VPC endpoint) for stream metrics
- **Enhanced Fan-Out** for low-latency consumers
- **Custom metrics** pushed via CloudWatch VPC endpoint
- **Shard-level metrics** for capacity planning

---

Would you like me to dive deeper into any specific aspect—such as a particular air-gapped environment, compliance framework (IL4/IL5/IL6), or alternative streaming architecture?
User: what are preventative and detect controls you would put on Kinesis think permission sets, permission boundaries, SCP, RCP and Config and and remediaiton. Also note logging is essential and data must be encrypted at rest and transit
Model: # Kinesis Security Controls Framework

## Comprehensive Preventative, Detective & Remediation Controls

---

## 1. SERVICE CONTROL POLICIES (SCPs)

### 1.1 Enforce Encryption at Rest

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyUnencryptedKinesisStreams",
      "Effect": "Deny",
      "Action": [
        "kinesis:CreateStream",
        "kinesis:UpdateStream"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "kinesis:StreamEncryptionType": "KMS"
        }
      }
    }
  ]
}
```

### 1.2 Enforce KMS CMK (Deny AWS Managed Keys)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyAWSManagedKeyForKinesis",
      "Effect": "Deny",
      "Action": [
        "kinesis:StartStreamEncryption"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "kinesis:KeyId": "alias/aws/kinesis"
        }
      }
    }
  ]
}
```

### 1.3 Restrict Kinesis to Approved Regions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyKinesisOutsideApprovedRegions",
      "Effect": "Deny",
      "Action": [
        "kinesis:*",
        "firehose:*",
        "kinesisanalytics:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:RequestedRegion": [
            "us-gov-west-1",
            "us-gov-east-1"
          ]
        }
      }
    }
  ]
}
```

### 1.4 Deny Disabling Encryption

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyStopStreamEncryption",
      "Effect": "Deny",
      "Action": [
        "kinesis:StopStreamEncryption",
        "kinesis:DisableEnhancedMonitoring"
      ],
      "Resource": "*"
    }
  ]
}
```

### 1.5 Deny Public Access / Cross-Account Sharing (Without Approval)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyKinesisCrossAccountAccess",
      "Effect": "Deny",
      "Action": [
        "kinesis:PutResourcePolicy"
      ],
      "Resource": "*",
      "Condition": {
        "ForAnyValue:StringNotEquals": {
          "aws:PrincipalOrgID": "o-yourorgid"
        }
      }
    }
  ]
}
```

### 1.6 Enforce VPC Endpoint Usage Only

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyKinesisAccessOutsideVPCE",
      "Effect": "Deny",
      "Action": [
        "kinesis:*"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": [
            "vpce-0abc123def456",
            "vpce-0xyz789ghi012"
          ]
        }
      }
    }
  ]
}
```

---

## 2. RESOURCE CONTROL POLICIES (RCPs)

### 2.1 Kinesis Stream Resource Policy - Deny External Principals

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyExternalPrincipals",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "kinesis:*",
      "Resource": "arn:aws:kinesis:*:*:stream/*",
      "Condition": {
        "StringNotEquals": {
          "aws:PrincipalOrgID": "o-yourorgid"
        }
      }
    }
  ]
}
```

### 2.2 Enforce TLS in Transit

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonTLSAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "kinesis:*",
      "Resource": "arn:aws:kinesis:*:*:stream/*",
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### 2.3 Deny Access Unless Via VPC Endpoint

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EnforceVPCEndpointAccess",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "kinesis:*",
      "Resource": "arn:aws:kinesis:*:*:stream/*",
      "Condition": {
        "StringNotEquals": {
          "aws:sourceVpce": "vpce-0abc123def456"
        }
      }
    }
  ]
}
```

### 2.4 Firehose Resource Policy - Restrict Destinations

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "RestrictFirehoseDestinations",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "firehose:CreateDeliveryStream",
        "firehose:UpdateDestination"
      ],
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "firehose:DestinationType": [
            "S3",
            "Redshift"
          ]
        }
      }
    }
  ]
}
```

---

## 3. PERMISSION BOUNDARIES

### 3.1 Kinesis Developer Permission Boundary

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowKinesisDataOperations",
      "Effect": "Allow",
      "Action": [
        "kinesis:PutRecord",
        "kinesis:PutRecords",
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:DescribeStreamSummary",
        "kinesis:ListShards",
        "kinesis:ListStreams",
        "kinesis:SubscribeToShard"
      ],
      "Resource": "arn:aws:kinesis:*:*:stream/app-*"
    },
    {
      "Sid": "DenyAdministrativeActions",
      "Effect": "Deny",
      "Action": [
        "kinesis:CreateStream",
        "kinesis:DeleteStream",
        "kinesis:StopStreamEncryption",
        "kinesis:DisableEnhancedMonitoring",
        "kinesis:MergeShards",
        "kinesis:SplitShard",
        "kinesis:UpdateShardCount",
        "kinesis:UpdateStreamMode",
        "kinesis:DeregisterStreamConsumer",
        "kinesis:PutResourcePolicy",
        "kinesis:DeleteResourcePolicy"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyKMSKeyDeletion",
      "Effect": "Deny",
      "Action": [
        "kms:ScheduleKeyDeletion",
        "kms:DisableKey",
        "kms:DeleteAlias"
      ],
      "Resource": "*"
    },
    {
      "Sid": "DenyCloudTrailTampering",
      "Effect": "Deny",
      "Action": [
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "cloudtrail:UpdateTrail"
      ],
      "Resource": "*"
    }
  ]
}
```

### 3.2 Kinesis Admin Permission Boundary

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowKinesisAdmin",
      "Effect": "Allow",
      "Action": [
        "kinesis:*",
        "firehose:*",
        "kinesisanalytics:*"
      ],
      "Resource": "*"
    },
    {
      "Sid": "HardDenyGuardrails",
      "Effect": "Deny",
      "Action": [
        "kinesis:StopStreamEncryption",
        "kinesis:DeleteResourcePolicy",
        "kinesis:DisableEnhancedMonitoring",
        "cloudtrail:StopLogging",
        "cloudtrail:DeleteTrail",
        "config:StopConfigurationRecorder",
        "config:DeleteConfigurationRecorder"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EnforceEncryptionOnCreate",
      "Effect": "Deny",
      "Action": "kinesis:CreateStream",
      "Resource": "*",
      "Condition": {
        "StringNotEquals": {
          "kinesis:StreamEncryptionType": "KMS"
        }
      }
    }
  ]
}
```

---

## 4. IAM PERMISSION SETS (Identity Center / SSO)

### 4.1 Kinesis Read-Only Permission Set

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KinesisReadOnly",
      "Effect": "Allow",
      "Action": [
        "kinesis:DescribeStream",
        "kinesis:DescribeStreamSummary",
        "kinesis:DescribeStreamConsumer",
        "kinesis:ListStreams",
        "kinesis:ListShards",
        "kinesis:ListStreamConsumers",
        "kinesis:ListTagsForStream",
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:SubscribeToShard",
        "firehose:DescribeDeliveryStream",
        "firehose:ListDeliveryStreams",
        "firehose:ListTagsForDeliveryStream"
      ],
      "Resource": "*"
    },
    {
      "Sid": "CloudWatchReadForKinesis",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:GetMetricData",
        "cloudwatch:GetMetricStatistics",
        "cloudwatch:ListMetrics",
        "cloudwatch:DescribeAlarms"
      ],
      "Resource": "*"
    }
  ]
}
```

### 4.2 Kinesis Producer Permission Set

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KinesisProducerAccess",
      "Effect": "Allow",
      "Action": [
        "kinesis:PutRecord",
        "kinesis:PutRecords",
        "kinesis:DescribeStream",
        "kinesis:DescribeStreamSummary"
      ],
      "Resource": "arn:aws:kinesis:*:*:stream/${aws:PrincipalTag/project}-*"
    },
    {
      "Sid": "KMSForEncryption",
      "Effect": "Allow",
      "Action": [
        "kms:GenerateDataKey",
        "kms:Decrypt"
      ],
      "Resource": "arn:aws:kms:*:*:key/*",
      "Condition": {
        "StringEquals": {
          "kms:ViaService": "kinesis.*.amazonaws.com"
        }
      }
    }
  ]
}
```

### 4.3 Kinesis Consumer Permission Set

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KinesisConsumerAccess",
      "Effect": "Allow",
      "Action": [
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:DescribeStreamSummary",
        "kinesis:ListShards",
        "kinesis:SubscribeToShard",
        "kinesis:DescribeStreamConsumer",
        "kinesis:RegisterStreamConsumer"
      ],
      "Resource": "arn:aws:kinesis:*:*:stream/${aws:PrincipalTag/project}-*"
    },
    {
      "Sid": "DynamoDBForKCL",
      "Effect": "Allow",
      "Action": [
        "dynamodb:CreateTable",
        "dynamodb:DescribeTable",
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Scan"
      ],
      "Resource": "arn:aws:dynamodb:*:*:table/kcl-*"
    },
    {
      "Sid": "CloudWatchForKCL",
      "Effect": "Allow",
      "Action": [
        "cloudwatch:PutMetricData"
      ],
      "Resource": "*"
    }
  ]
}
```

### 4.4 Security Auditor Permission Set

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "KinesisAuditAccess",
      "Effect": "Allow",
      "Action": [
        "kinesis:Describe*",
        "kinesis:List*",
        "kinesis:GetResourcePolicy",
        "firehose:Describe*",
        "firehose:List*",
        "cloudtrail:LookupEvents",
        "cloudtrail:GetTrailStatus",
        "config:GetComplianceDetailsByResource",
        "config:GetResourceConfigHistory",
        "cloudwatch:GetMetricData",
        "cloudwatch:DescribeAlarms"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 5. AWS CONFIG RULES

### 5.1 Custom & Managed Config Rules

```yaml
# AWS Config Rules for Kinesis Compliance
ConfigRules:

  # --- ENCRYPTION AT REST ---
  - RuleName: kinesis-stream-encrypted
    Source: AWS_MANAGED
    Identifier: KINESIS_STREAM_ENCRYPTED
    Description: "Ensures all Kinesis streams are encrypted at rest"
    Scope:
      ResourceTypes:
        - AWS::Kinesis::Stream
    Remediation: SSM-EnableKinesisEncryption

  # --- ENCRYPTION WITH CMK (Not AWS Managed) ---
  - RuleName: kinesis-stream-encrypted-with-cmk
    Source: CUSTOM_LAMBDA
    Description: "Ensures Kinesis uses customer-managed KMS key"
    EvaluationMode: DETECTIVE
    Scope:
      ResourceTypes:
        - AWS::Kinesis::Stream

  # --- ENHANCED MONITORING ENABLED ---
  - RuleName: kinesis-enhanced-monitoring-enabled
    Source: CUSTOM_LAMBDA
    Description: "Ensures enhanced shard-level monitoring is enabled"
    Scope:
      ResourceTypes:
        - AWS::Kinesis::Stream

  # --- FIREHOSE ENCRYPTION ---
  - RuleName: firehose-delivery-stream-encrypted
    Source: CUSTOM_LAMBDA
    Description: "Ensures Firehose delivery streams use SSE with CMK"
    Scope:
      ResourceTypes:
        - AWS::KinesisFirehose::DeliveryStream

  # --- KINESIS TAGGED ---
  - RuleName: kinesis-required-tags
    Source: AWS_MANAGED
    Identifier: REQUIRED_TAGS
    InputParameters:
      tag1Key: "Environment"
      tag2Key: "DataClassification"
      tag3Key: "Owner"
      tag4Key: "CostCenter"
    Scope:
      ResourceTypes:
        - AWS::Kinesis::Stream

  # --- VPC ENDPOINT CHECK ---
  - RuleName: kinesis-vpc-endpoint-exists
    Source: CUSTOM_LAMBDA
    Description: "Ensures VPC endpoint exists for Kinesis"
   
  # --- CLOUDTRAIL DATA EVENTS FOR KINESIS ---
  - RuleName: kinesis-cloudtrail-data-events-enabled
    Source: CUSTOM_LAMBDA
    Description: "Ensures CloudTrail data events are enabled for Kinesis"
```

### 5.2 Custom Config Rule Lambda - CMK Encryption Check

```python
import boto3
import json

def lambda_handler(event, context):
    config = boto3.client('config')
    kinesis = boto3.client('kinesis')
   
    configuration_item = json.loads(event['invokingEvent'])['configurationItem']
    stream_name = configuration_item['resourceName']
   
    try:
        response = kinesis.describe_stream(StreamName=stream_name)
        stream_desc = response['StreamDescription']
       
        encryption_type = stream_desc.get('EncryptionType', 'NONE')
        key_id = stream_desc.get('KeyId', '')
       
        # Check encryption is KMS with CMK (not aws/kinesis)
        if encryption_type == 'KMS' and 'alias/aws/kinesis' not in key_id:
            compliance = 'COMPLIANT'
            annotation = f'Stream encrypted with CMK: {key_id}'
        elif encryption_type == 'KMS':
            compliance = 'NON_COMPLIANT'
            annotation = 'Stream using AWS managed key - CMK required'
        else:
            compliance = 'NON_COMPLIANT'
            annotation = 'Stream encryption is not enabled'
           
    except Exception as e:
        compliance = 'NOT_APPLICABLE'
        annotation = str(e)
   
    config.put_evaluations(
        Evaluations=[{
            'ComplianceResourceType': configuration_item['resourceType'],
            'ComplianceResourceId': configuration_item['resourceId'],
            'ComplianceType': compliance,
            'Annotation': annotation,
            'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
        }],
        ResultToken=event['resultToken']
    )
```

### 5.3 Custom Config Rule - Enhanced Monitoring Check

```python
import boto3
import json

def lambda_handler(event, context):
    config = boto3.client('config')
    kinesis = boto3.client('kinesis')
   
    configuration_item = json.loads(event['invokingEvent'])['configurationItem']
    stream_name = configuration_item['resourceName']
   
    required_metrics = [
        'IncomingBytes', 'IncomingRecords', 'OutgoingBytes',
        'OutgoingRecords', 'WriteProvisionedThroughputExceeded',
        'ReadProvisionedThroughputExceeded', 'IteratorAgeMilliseconds'
    ]
   
    try:
        response = kinesis.describe_stream(StreamName=stream_name)
        enhanced_monitoring = response['StreamDescription'].get('EnhancedMonitoring', [])
       
        enabled_metrics = []
        for monitor in enhanced_monitoring:
            enabled_metrics.extend(monitor.get('ShardLevelMetrics', []))
       
        if 'ALL' in enabled_metrics or all(m in enabled_metrics for m in required_metrics):
            compliance = 'COMPLIANT'
            annotation = 'Enhanced monitoring enabled'
        else:
            compliance = 'NON_COMPLIANT'
            missing = set(required_metrics) - set(enabled_metrics)
            annotation = f'Missing shard-level metrics: {missing}'
           
    except Exception as e:
        compliance = 'NOT_APPLICABLE'
        annotation = str(e)
   
    config.put_evaluations(
        Evaluations=[{
            'ComplianceResourceType': configuration_item['resourceType'],
            'ComplianceResourceId': configuration_item['resourceId'],
            'ComplianceType': compliance,
            'Annotation': annotation,
            'OrderingTimestamp': configuration_item['configurationItemCaptureTime']
        }],
        ResultToken=event['resultToken']
    )
```

---

## 6. AUTOMATED REMEDIATION

### 6.1 SSM Automation - Enable Kinesis Encryption

```yaml
description: 'Enable KMS encryption on Kinesis Data Stream'
schemaVersion: '0.3'
assumeRole: '{{AutomationAssumeRole}}'
parameters:
  StreamName:
    type: String
    description: 'Name of the Kinesis stream'
  KMSKeyId:
    type: String
    description: 'KMS CMK ARN for encryption'
    default: 'arn:aws:kms:us-gov-west-1:123456789012:key/your-cmk-id'
  AutomationAssumeRole:
    type: String
mainSteps:
  - name: CheckCurrentEncryption
    action: 'aws:executeAwsApi'
    inputs:
      Service: kinesis
      Api: DescribeStream
      StreamName: '{{StreamName}}'
    outputs:
      - Name: EncryptionType
        Selector: '$.StreamDescription.EncryptionType'
        Type: String

  - name: EnableEncryption
    action: 'aws:executeAwsApi'
    inputs:
      Service: kinesis
      Api: StartStreamEncryption
      StreamName: '{{StreamName}}'
      EncryptionType: 'KMS'
      KeyId: '{{KMSKeyId}}'

  - name: VerifyEncryption
    action: 'aws:waitForAwsResourceProperty'
    inputs:
      Service: kinesis
      Api: DescribeStream
      StreamName: '{{StreamName}}'
      PropertySelector: '$.StreamDescription.EncryptionType'
      DesiredValues:
        - 'KMS'
    timeoutSeconds: 120

  - name: EnableEnhancedMonitoring
    action: 'aws:executeAwsApi'
    inputs:
      Service: kinesis
      Api: EnableEnhancedMonitoring
      StreamName: '{{StreamName}}'
      ShardLevelMetrics:
        - 'ALL'
```

### 6.2 SSM Automation - Enable Firehose Encryption

```yaml
description: 'Enable SSE on Kinesis Firehose Delivery Stream'
schemaVersion: '0.3'
assumeRole: '{{AutomationAssumeRole}}'
parameters:
  DeliveryStreamName:
    type: String
  KMSKeyARN:
    type: String
    default: 'arn:aws:kms:us-gov-west-1:123456789012:key/your-cmk-id'
  AutomationAssumeRole:
    type: String
mainSteps:
  - name: EnableFirehoseEncryption
    action: 'aws:executeAwsApi'
    inputs:
      Service: firehose
      Api: StartDeliveryStreamEncryption
      DeliveryStreamName: '{{DeliveryStreamName}}'
      DeliveryStreamEncryptionInput:
        KeyARN: '{{KMSKeyARN}}'
        KeyType: 'CUSTOMER_MANAGED_CMK'

  - name: WaitForEncryption
    action: 'aws:waitForAwsResourceProperty'
    inputs:
      Service: firehose
      Api: DescribeDeliveryStream
      DeliveryStreamName: '{{DeliveryStreamName}}'
      PropertySelector: '$.DeliveryStreamDescription.DeliveryStreamEncryptionConfiguration.Status'
      DesiredValues:
        - 'ENABLED'
    timeoutSeconds: 180
```

### 6.3 Config Remediation Configuration

```json
{
  "RemediationConfigurations": [
    {
      "ConfigRuleName": "kinesis-stream-encrypted",
      "TargetType": "SSM_DOCUMENT",
      "TargetId": "Custom-EnableKinesisEncryption",
      "Parameters": {
        "StreamName": {
          "ResourceValue": {
            "Value": "RESOURCE_ID"
          }
        },
        "AutomationAssumeRole": {
          "StaticValue": {
            "Values": [
              "arn:aws:iam::123456789012:role/ConfigRemediationRole"
            ]
          }
        }
      },
      "Automatic": true,
      "MaximumAutomaticAttempts": 3,
      "RetryAttemptSeconds": 60
    }
  ]
}
```

### 6.4 EventBridge Auto-Remediation Rules

```json
{
  "Source": ["aws.config"],
  "DetailType": ["Config Rules Compliance Change"],
  "Detail": {
    "messageType": ["ComplianceChangeNotification"],
    "configRuleName": [
      "kinesis-stream-encrypted",
      "kinesis-enhanced-monitoring-enabled",
      "kinesis-stream-encrypted-with-cmk"
    ],
    "newEvaluationResult": {
      "complianceType": ["NON_COMPLIANT"]
    }
  }
}
```

**Target: Lambda Auto-Remediation**

```python
import boto3
import json

kinesis = boto3.client('kinesis')
sns = boto3.client('sns')
ssm = boto3.client('ssm')

CMK_KEY_ID = 'arn:aws:kms:us-gov-west-1:123456789012:key/your-cmk-id'
SNS_TOPIC = 'arn:aws:sns:us-gov-west-1:123456789012:security-alerts'

def lambda_handler(event, context):
    detail = event['detail']
    rule_name = detail['configRuleName']
    resource_id = detail['resourceId']
    resource_type = detail['resourceType']
   
    remediation_actions = {
        'kinesis-stream-encrypted': remediate_encryption,
        'kinesis-enhanced-monitoring-enabled': remediate_monitoring,
        'kinesis-stream-encrypted-with-cmk': remediate_cmk_encryption
    }
   
    action = remediation_actions.get(rule_name)
    if action:
        try:
            action(resource_id)
            notify(f"AUTO-REMEDIATED: {rule_name} for {resource_id}", "SUCCESS")
        except Exception as e:
            notify(f"REMEDIATION FAILED: {rule_name} for {resource_id}: {str(e)}", "FAILURE")
            raise

def remediate_encryption(stream_name):
    kinesis.start_stream_encryption(
        StreamName=stream_name,
        EncryptionType='KMS',
        KeyId=CMK_KEY_ID
    )

def remediate_monitoring(stream_name):
    kinesis.enable_enhanced_monitoring(
        StreamName=stream_name,
        ShardLevelMetrics=['ALL']
    )

def remediate_cmk_encryption(stream_name):
    # Stop AWS managed key encryption, re-encrypt with CMK
    try:
        kinesis.stop_stream_encryption(
            StreamName=stream_name,
            EncryptionType='KMS',
            KeyId='alias/aws/kinesis'
        )
    except:
        pass
   
    import time
    time.sleep(10)
   
    kinesis.start_stream_encryption(
        StreamName=stream_name,
        EncryptionType='KMS',
        KeyId=CMK_KEY_ID
    )

def notify(message, status):
    sns.publish(
        TopicArn=SNS_TOPIC,
        Subject=f'Kinesis Remediation - {status}',
        Message=json.dumps({
            'message': message,
            'status': status,
            'service': 'kinesis'
        })
    )
```

---

## 7. LOGGING FRAMEWORK

### 7.1 CloudTrail - Management & Data Events

```json
{
  "Trail": {
    "Name": "kinesis-audit-trail",
    "S3BucketName": "central-logging-bucket",
    "IsMultiRegionTrail": true,
    "EnableLogFileValidation": true,
    "KMSKeyId": "arn:aws:kms:us-gov-west-1:123456789012:key/trail-cmk",
    "EventSelectors": [
      {
        "ReadWriteType": "All",
        "IncludeManagementEvents": true,
        "DataResources": [
          {
            "Type": "AWS::Kinesis::Stream",
            "Values": ["arn:aws:kinesis:*:*:stream/*"]
          },
          {
            "Type": "AWS::KinesisFirehose::DeliveryStream",
            "Values": ["arn:aws:firehose:*:*:deliverystream/*"]
          }
        ]
      }
    ],
    "AdvancedEventSelectors": [
      {
        "Name": "KinesisDataEvents",
        "FieldSelectors": [
          {
            "Field": "eventCategory",
            "Equals": ["Data"]
          },
          {
            "Field": "resources.type",
            "Equals": ["AWS::Kinesis::Stream"]
          }
        ]
      }
    ]
  }
}
```

### 7.2 CloudWatch Alarms & Metrics

```yaml
CloudWatchAlarms:

  # Unauthorized API Calls
  - AlarmName: Kinesis-UnauthorizedAPICalls
    MetricName: KinesisUnauthorizedCalls
    Namespace: CloudTrailMetrics
    FilterPattern: |
      { ($.eventSource = "kinesis.amazonaws.com") &&
        ($.errorCode = "*UnauthorizedAccess*" ||
         $.errorCode = "AccessDenied*") }
    Period: 300
    Threshold: 1
    AlarmActions:
      - arn:aws:sns:us-gov-west-1:123456789012:security-alerts

  # Encryption Disabled
  - AlarmName: Kinesis-EncryptionStopped
    FilterPattern: |
      { ($.eventSource = "kinesis.amazonaws.com") &&
        ($.eventName = "StopStreamEncryption") }
    Period: 60
    Threshold: 1
    AlarmActions:
      - arn:aws:sns:us-gov-west-1:123456789012:critical-security

  # Stream Deletion
  - AlarmName: Kinesis-StreamDeleted
    FilterPattern: |
      { ($.eventSource = "kinesis.amazonaws.com") &&
        ($.eventName = "DeleteStream") }
    Period: 60
    Threshold: 1
    AlarmActions:
      - arn:aws:sns:us-gov-west-1:123456789012:critical-security

  # Resource Policy Changes
  - AlarmName: Kinesis-ResourcePolicyChanged
    FilterPattern: |
      { ($.eventSource = "kinesis.amazonaws.com") &&
        ($.eventName = "PutResourcePolicy" ||
         $.eventName = "DeleteResourcePolicy") }
    Period: 60
    Threshold: 1
    AlarmActions:
      - arn:aws:sns:us-gov-west-1:123456789012:security-alerts

  # Throttling Detection
  - AlarmName: Kinesis-WriteThrottled
    Namespace: AWS/Kinesis
    MetricName: WriteProvisionedThroughputExceeded
    Statistic: Sum
    Period: 300
    Threshold: 10
    AlarmActions:
      - arn:aws:sns:us-gov-west-1:123456789012:ops-alerts

  # Iterator Age (Consumer Lag)
  - AlarmName: Kinesis-HighIteratorAge
    Namespace: AWS/Kinesis
    MetricName: GetRecords.IteratorAgeMilliseconds
    Statistic: Maximum
    Period: 300
    Threshold: 3600000  # 1 hour
    AlarmActions:
      - arn:aws:sns:us-gov-west-1:123456789012:ops-alerts
```

### 7.3 VPC Flow Logs for Kinesis Endpoint Traffic

```json
{
  "FlowLogConfig": {
    "ResourceType": "VPC",
    "TrafficType": "ALL",
    "LogDestination": "arn:aws:s3:::vpc-flow-logs-bucket/kinesis/",
    "LogFormat": "${version} ${account-id} ${interface-id} ${srcaddr} ${dstaddr} ${srcport} ${dstport} ${protocol} ${packets} ${bytes} ${start} ${end} ${action} ${log-status} ${vpc-id} ${subnet-id} ${az-id} ${flow-direction}",
    "MaxAggregationInterval": 60
  }
}
```

---

## 8. CONTROL MATRIX SUMMARY

```
┌─────────────────────────────────────┬──────────┬──────────┬─────────────┐
│ Control                             │ Prevent  │ Detect   │ Remediate   │
├─────────────────────────────────────┼──────────┼──────────┼─────────────┤
│ Encryption at Rest (KMS CMK)        │ SCP/RCP  │ Config   │ SSM Auto    │
│ Encryption in Transit (TLS)         │ RCP      │ Config   │ Manual      │
│ VPC Endpoint Only Access            │ SCP/RCP  │ Config   │ Manual      │
│ No Cross-Account Access             │ SCP/RCP  │ Config   │ Lambda Auto │
│ Region Restriction                  │ SCP      │ Config   │ N/A         │
│ Enhanced Monitoring                 │ SCP*     │ Config   │ SSM Auto    │
│ CloudTrail Data Events              │ SCP*     │ Config   │ SSM Auto    │
│ Required Tagging                    │ SCP      │ Config   │ Lambda Auto │
│ Prevent Encryption Disable          │ SCP      │ CW Alarm │ EventBridge │
│ Prevent Stream Deletion             │ SCP/PB   │ CW Alarm │ Notify      │
│ Prevent Logging Tampering           │ PB/SCP   │ CW Alarm │ EventBridge │
│ Least Privilege (Producer/Consumer) │ PS/PB    │ IAM AA   │ Manual      │
│ Unauthorized Access Attempts        │ IAM      │ CW Alarm │ Notify      │
│ Consumer Lag Monitoring             │ N/A      │ CW Alarm │ Auto-Scale  │
│ Throttling Detection                │ N/A      │ CW Alarm │ Auto-Scale  │
└─────────────────────────────────────┴──────────┴──────────┴─────────────┘

Legend: SCP=Service Control Policy, RCP=Resource Control Policy,
        PB=Permission Boundary, PS=Permission Set,
        CW=CloudWatch, AA=Access Analyzer
```

---

## 9. VPC ENDPOINT POLICIES (Additional Layer)

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowOrgOnlyKinesisAccess",
      "Effect": "Allow",
      "Principal": {
        "AWS": "*"
      },
      "Action": [
        "kinesis:PutRecord",
        "kinesis:PutRecords",
        "kinesis:GetRecords",
        "kinesis:GetShardIterator",
        "kinesis:DescribeStream",
        "kinesis:ListStreams",
        "kinesis:ListShards"
      ],
      "Resource": "arn:aws:kinesis:*:*:stream/*",
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-yourorgid"
        }
      }
    },
    {
      "Sid": "DenyDangerousActionsViaEndpoint",
      "Effect": "Deny",
      "Principal": "*",
      "Action": [
        "kinesis:DeleteStream",
        "kinesis:StopStreamEncryption",
        "kinesis:PutResourcePolicy",
        "kinesis:DeleteResourcePolicy"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## 10. COMPLIANCE VALIDATION SCRIPT

```python
#!/usr/bin/env python3
"""Kinesis Security Compliance Validator"""

import boto3
import json
from datetime import datetime

class KinesisComplianceChecker:
    def __init__(self):
        self.kinesis = boto3.client('kinesis')
        self.firehose = boto3.client('firehose')
        self.cloudtrail = boto3.client('cloudtrail')
        self.results = []

    def check_all_streams(self):
        streams = self.kinesis.list_streams()['StreamNames']
        for stream in streams:
            self.check_stream(stream)
        return self.results

    def check_stream(self, stream_name):
        desc = self.kinesis.describe_stream(StreamName=stream_name)['StreamDescription']
       
        # Check 1: Encryption at rest
        enc_type = desc.get('EncryptionType', 'NONE')
        self._add_result(stream_name, 'ENCRYPTION_AT_REST',
                        enc_type == 'KMS',
                        f'Encryption: {enc_type}')
       
        # Check 2: CMK (not AWS managed)
        key_id = desc.get('KeyId', '')
        is_cmk = enc_type == 'KMS' and 'alias/aws/kinesis' not in key_id
        self._add_result(stream_name, 'CMK_ENCRYPTION',
                        is_cmk,
                        f'Key: {key_id}')
       
        # Check 3: Enhanced monitoring
        monitoring = desc.get('EnhancedMonitoring', [])
        metrics = []
        for m in monitoring:
            metrics.extend(m.get('ShardLevelMetrics', []))
        has_monitoring = 'ALL' in metrics or len(metrics) >= 7
        self._add_result(stream_name, 'ENHANCED_MONITORING',
                        has_monitoring,
                        f'Metrics: {metrics}')
       
        # Check 4: Tags
        tags = self.kinesis.list_tags_for_stream(StreamName=stream_name)['Tags']
        tag_keys = [t['Key'] for t in tags]
        required_tags = ['Environment', 'DataClassification', 'Owner']
        has_tags = all(t in tag_keys for t in required_tags)
        self._add_result(stream_name, 'REQUIRED_TAGS',
                        has_tags,
                        f'Tags: {tag_keys}')

    def _add_result(self, resource, check, compliant, detail):
        self.results.append({
            'resource': resource,
            'check': check,
            'compliant': compliant,
            'detail': detail,
            'timestamp': datetime.utcnow().isoformat()
        })

    def generate_report(self):
        self.check_all_streams()
        compliant = sum(1 for r in self.results if r['compliant'])
        total = len(self.results)
       
        report = {
            'summary': {
                'total_checks': total,
                'compliant': compliant,
                'non_compliant': total - compliant,
                'compliance_percentage': round((compliant/total)*100, 2) if total > 0 else 0
            },
            'findings': self.results
        }
       
        print(json.dumps(report, indent=2))
        return report

if __name__ == '__main__':
    checker = KinesisComplianceChecker()
    checker.generate_report()
```

---

This framework provides **defense in depth** across all layers: identity, resource, network, encryption, logging, and automated remediation. Would you like me to expand on any specific area such as Firehose-specific controls, Kinesis Video Streams, or integration with a specific compliance framework (FedRAMP, IL5, NIST 800-53
