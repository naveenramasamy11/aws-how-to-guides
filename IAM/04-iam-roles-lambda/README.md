# Lab 04 — IAM Roles for Lambda (Execution Roles) 🟡

## 🎯 Goal
Create a Lambda function with a least-privilege execution role. Understand the difference between execution roles (identity-based) and resource-based policies, and how to allow other services to invoke Lambda.

---

## 🧠 Concepts

### Lambda Execution Role
Every Lambda function **must** have an execution role. This role defines what AWS services the function can call. Lambda assumes this role during execution.

### Resource-Based Policy (Lambda Policy)
Unlike EC2, Lambda also has a **resource-based policy** that controls **who can invoke the function**:
- Other AWS services (S3, SNS, EventBridge, API Gateway)
- Other accounts
- IAM users/roles

### Two Types of IAM Controls for Lambda
```
Who can INVOKE Lambda?     →  Resource-Based Policy (attached to the function)
What can Lambda DO?        →  Execution Role (identity-based, attached to the role)
```

---

## 🏗️ Architecture

```
EventBridge Rule (cron)
    │ (invoke permission via resource-based policy)
    ▼
Lambda Function: s3-inventory-fn
    │ (assumes execution role)
    ▼
Execution Role: LambdaS3InventoryRole
    ├── AWSLambdaBasicExecutionRole  ← CloudWatch Logs
    └── S3ReadOnly policy            ← List S3 buckets
```

---

## 🧪 PoC

### Step 1: Create the Lambda Execution Role
```bash
# Trust policy — only Lambda service can assume this role
cat > /tmp/lambda-trust-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Service": "lambda.amazonaws.com"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
EOF

# Create the role
ROLE_ARN=$(aws iam create-role \
  --role-name LambdaS3InventoryRole \
  --assume-role-policy-document file:///tmp/lambda-trust-policy.json \
  --description "Execution role for s3-inventory Lambda function" \
  --tags Key=ManagedBy,Value=manual Key=Purpose,Value=LambdaS3Inventory \
  --query 'Role.Arn' \
  --output text)

echo "Created execution role: $ROLE_ARN ✅"
```

### Step 2: Attach Permissions Policies
```bash
# Basic execution role: allows writing logs to CloudWatch
aws iam attach-role-policy \
  --role-name LambdaS3InventoryRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# S3 read access
aws iam attach-role-policy \
  --role-name LambdaS3InventoryRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

echo "Policies attached ✅"

# Wait for IAM eventual consistency
sleep 10
```

### Step 3: Create the Lambda Function Code
```bash
cat > /tmp/lambda_function.py << 'EOF'
import boto3
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def lambda_handler(event, context):
    """List all S3 buckets and their regions — PoC for IAM role testing."""
    s3 = boto3.client('s3')

    try:
        response = s3.list_buckets()
        buckets = response.get('Buckets', [])

        bucket_info = []
        for bucket in buckets:
            try:
                location = s3.get_bucket_location(Bucket=bucket['Name'])
                region = location['LocationConstraint'] or 'us-east-1'
            except Exception as e:
                region = f"error: {str(e)}"

            bucket_info.append({
                'name': bucket['Name'],
                'created': bucket['CreationDate'].isoformat(),
                'region': region
            })

        result = {
            'statusCode': 200,
            'total_buckets': len(buckets),
            'buckets': bucket_info
        }

        logger.info(f"Found {len(buckets)} S3 buckets")
        return result

    except Exception as e:
        logger.error(f"Error listing buckets: {str(e)}")
        raise
EOF

# Package the function
cd /tmp && zip lambda-s3-inventory.zip lambda_function.py
echo "Lambda package created ✅"
```

### Step 4: Create the Lambda Function
```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

FUNCTION_ARN=$(aws lambda create-function \
  --function-name s3-inventory-fn \
  --runtime python3.12 \
  --role $ROLE_ARN \
  --handler lambda_function.lambda_handler \
  --zip-file fileb:///tmp/lambda-s3-inventory.zip \
  --timeout 30 \
  --memory-size 128 \
  --description "PoC: Lambda with IAM execution role to list S3 buckets" \
  --environment Variables='{LOG_LEVEL=INFO}' \
  --tags Purpose=IAMRolePoC,ManagedBy=manual \
  --query 'FunctionArn' \
  --output text)

echo "Created Lambda: $FUNCTION_ARN ✅"

# Wait for active state
aws lambda wait function-active --function-name s3-inventory-fn
echo "Lambda is active ✅"
```

### Step 5: Invoke the Lambda Function
```bash
# Synchronous invocation
aws lambda invoke \
  --function-name s3-inventory-fn \
  --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/lambda-output.json

echo "Lambda output:"
cat /tmp/lambda-output.json | python3 -m json.tool
```

### Step 6: View CloudWatch Logs
```bash
# Get the latest log stream
LOG_GROUP="/aws/lambda/s3-inventory-fn"
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name $LOG_GROUP \
  --order-by LastEventTime \
  --descending \
  --query 'logStreams[0].logStreamName' \
  --output text)

echo "Log stream: $LOG_STREAM"

# Get the log events
aws logs get-log-events \
  --log-group-name $LOG_GROUP \
  --log-stream-name "$LOG_STREAM" \
  --query 'events[*].message' \
  --output text
```

### Step 7: Add Resource-Based Policy (allow EventBridge to invoke)
```bash
# Allow EventBridge to invoke this Lambda
aws lambda add-permission \
  --function-name s3-inventory-fn \
  --statement-id AllowEventBridgeInvoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn "arn:aws:events:us-east-1:${ACCOUNT_ID}:rule/s3-inventory-schedule"

echo "Resource-based policy added ✅"

# View the resource-based policy
aws lambda get-policy \
  --function-name s3-inventory-fn \
  --query 'Policy' \
  --output text | python3 -m json.tool
```

### Step 8: Create EventBridge Rule to trigger Lambda on schedule
```bash
# Create a rule (runs every 5 minutes)
RULE_ARN=$(aws events put-rule \
  --name s3-inventory-schedule \
  --schedule-expression "rate(5 minutes)" \
  --state ENABLED \
  --description "Trigger s3-inventory-fn every 5 minutes" \
  --query 'RuleArn' \
  --output text)

echo "EventBridge rule: $RULE_ARN ✅"

# Add Lambda as a target
aws events put-targets \
  --rule s3-inventory-schedule \
  --targets "Id=LambdaTarget,Arn=$FUNCTION_ARN"

echo "Lambda added as EventBridge target ✅"
```

### Step 9: Test — what happens if Lambda tries to write to S3?
```bash
# Create a test function that tries to PUT to S3 (should fail with current role)
cat > /tmp/test_s3_write.py << 'EOF'
import boto3
import json

def lambda_handler(event, context):
    s3 = boto3.client('s3')
    try:
        s3.put_object(
            Bucket='some-bucket',
            Key='test.txt',
            Body=b'test'
        )
        return {'statusCode': 200, 'result': 'Write succeeded'}
    except Exception as e:
        return {'statusCode': 403, 'result': f'Write DENIED: {str(e)}'}
EOF

# Package and update
cd /tmp && zip test-write.zip test_s3_write.py

aws lambda update-function-code \
  --function-name s3-inventory-fn \
  --zip-file fileb:///tmp/test-write.zip \
  --query 'FunctionArn' --output text

aws lambda wait function-updated --function-name s3-inventory-fn

# Update handler
aws lambda update-function-configuration \
  --function-name s3-inventory-fn \
  --handler test_s3_write.lambda_handler

aws lambda wait function-updated --function-name s3-inventory-fn

aws lambda invoke \
  --function-name s3-inventory-fn \
  --payload '{}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/write-output.json

cat /tmp/write-output.json
# Expected: Write DENIED — AmazonS3ReadOnlyAccess blocks PutObject ✅
```

---

## ✅ Cleanup

```bash
# Delete EventBridge rule & targets
aws events remove-targets --rule s3-inventory-schedule --ids LambdaTarget
aws events delete-rule --name s3-inventory-schedule

# Delete Lambda function
aws lambda delete-function --function-name s3-inventory-fn

# Detach policies from role
aws iam detach-role-policy \
  --role-name LambdaS3InventoryRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam detach-role-policy \
  --role-name LambdaS3InventoryRole \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

# Delete role
aws iam delete-role --role-name LambdaS3InventoryRole

# Delete CloudWatch log group
aws logs delete-log-group --log-group-name /aws/lambda/s3-inventory-fn

echo "All resources cleaned up ✅"
```

---

## 🔒 Security Best Practices
1. **Use `AWSLambdaBasicExecutionRole`** — minimal CloudWatch logging permissions
2. **Scope Resource ARNs** — never use `*` for sensitive operations
3. **Use Lambda Layers** for shared dependencies (reduces attack surface)
4. **Enable VPC mode** for Lambda accessing private resources
5. **Use `aws:SourceAccount` condition** in resource-based policies to prevent confused deputy attacks

---

## ➡️ Next Lab
[Lab 05 — Cross-Account Access →](../05-cross-account-access/README.md)
