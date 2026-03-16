# Lab 06 — Event-Driven Pipeline: S3 → Lambda → DynamoDB/SNS 🔴

## Goal

Build a fully automated, serverless pipeline: when a JSON file is uploaded to S3, a Lambda function parses the records, writes them to DynamoDB, and sends an SNS notification summary — all triggered by S3 event notifications.

---

## Concepts

### S3 Event Notification Targets

| Target | Use Case | Delivery |
|--------|----------|----------|
| AWS Lambda | Synchronous processing; transform/enrich data | Synchronous invoke |
| Amazon SQS | Queue for async/batch consumers | At-least-once delivery |
| Amazon SNS | Fan-out to multiple subscribers | Push notification |
| EventBridge | Complex routing with content-based filtering | Event bus routing |

### Event Notification Filters

```json
{
  "Filter": {
    "Key": {
      "FilterRules": [
        {"Name": "prefix", "Value": "uploads/"},
        {"Name": "suffix", "Value": ".json"}
      ]
    }
  }
}
```

Only objects whose keys start with `uploads/` AND end with `.json` trigger the event.

### Lambda Event Shape (S3 PUT)

```json
{
  "Records": [{
    "eventSource": "aws:s3",
    "eventName": "ObjectCreated:Put",
    "s3": {
      "bucket": {"name": "my-bucket"},
      "object": {
        "key": "uploads/2024/data.json",
        "size": 1024,
        "eTag": "abc123"
      }
    }
  }]
}
```

---

## Architecture Diagram

```
   User / App
      │
      │ aws s3 cp uploads/orders.json
      ▼
┌─────────────────────────────┐
│  S3 Bucket                  │
│  Event: ObjectCreated       │
│  Filter: prefix=uploads/    │
│          suffix=.json       │
└─────────────────────────────┘
              │
              │  Invoke (synchronous)
              ▼
┌─────────────────────────────────────┐
│  Lambda Function: process-s3-event  │
│                                     │
│  1. Parse S3 event → get key        │
│  2. s3.get_object → read JSON       │
│  3. Loop through records            │
│  4. dynamodb.put_item → write each  │
│  5. sns.publish → summary message   │
└─────────────────────────────────────┘
         │                │
         ▼                ▼
┌──────────────┐  ┌────────────────────┐
│  DynamoDB    │  │  SNS Topic         │
│  Table:      │  │  → Email / Slack / │
│  Orders      │  │    SQS subscriber  │
└──────────────┘  └────────────────────┘
```

---

## PoC Steps

### Step 1 — Set variables

```bash
export AWS_REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-howto-${ACCOUNT_ID}-${AWS_REGION}"
export LAMBDA_FUNCTION="s3-howto-processor"
export DYNAMO_TABLE="S3HowToOrders"
export SNS_TOPIC_NAME="S3HowToNotifications"
echo "Bucket: $BUCKET_NAME | Lambda: $LAMBDA_FUNCTION | Table: $DYNAMO_TABLE"
```

---

### Step 2 — Create DynamoDB table

```bash
aws dynamodb create-table \
  --table-name "$DYNAMO_TABLE" \
  --attribute-definitions \
    AttributeName=order_id,AttributeType=S \
  --key-schema \
    AttributeName=order_id,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --tags Key=Project,Value=S3HowTo Key=Lab,Value=06 \
  --region "$AWS_REGION"

# Wait for table to become active
aws dynamodb wait table-exists --table-name "$DYNAMO_TABLE"
echo "DynamoDB table is active"
```

---

### Step 3 — Create SNS topic

```bash
SNS_TOPIC_ARN=$(aws sns create-topic \
  --name "$SNS_TOPIC_NAME" \
  --tags Key=Project,Value=S3HowTo Key=Lab,Value=06 \
  --query TopicArn \
  --output text)
echo "SNS Topic ARN: $SNS_TOPIC_ARN"

# Optional: subscribe your email for notifications
# aws sns subscribe \
#   --topic-arn "$SNS_TOPIC_ARN" \
#   --protocol email \
#   --notification-endpoint your-email@example.com
```

---

### Step 4 — Write the Lambda function

```bash
mkdir -p /tmp/lambda-s3-processor

cat > /tmp/lambda-s3-processor/lambda_function.py << 'PYEOF'
"""
Lambda function: process-s3-event
- Triggered by S3 ObjectCreated events on prefix uploads/*.json
- Reads the uploaded JSON file from S3
- Writes each order record to DynamoDB
- Publishes a summary to SNS
"""
import json
import os
import boto3
from datetime import datetime, timezone

s3_client = boto3.client("s3")
dynamodb = boto3.resource("dynamodb")
sns_client = boto3.client("sns")

TABLE_NAME = os.environ["DYNAMO_TABLE"]
SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]


def lambda_handler(event, context):
    table = dynamodb.Table(TABLE_NAME)
    results = {"processed": 0, "errors": 0, "skipped": 0}

    for record in event.get("Records", []):
        bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]
        print(f"Processing s3://{bucket}/{key}")

        try:
            # Read the JSON file from S3
            response = s3_client.get_object(Bucket=bucket, Key=key)
            raw = response["Body"].read().decode("utf-8")
            orders = json.loads(raw)

            if not isinstance(orders, list):
                orders = [orders]

            # Write each order to DynamoDB
            with table.batch_writer() as batch:
                for order in orders:
                    if "order_id" not in order:
                        print(f"Skipping record without order_id: {order}")
                        results["skipped"] += 1
                        continue

                    # Enrich with processing metadata
                    order["processed_at"] = datetime.now(timezone.utc).isoformat()
                    order["source_key"] = key
                    order["source_bucket"] = bucket

                    batch.put_item(Item=order)
                    results["processed"] += 1

            print(f"Wrote {results['processed']} orders to DynamoDB")

        except Exception as e:
            print(f"ERROR processing {key}: {e}")
            results["errors"] += 1

    # Publish summary to SNS
    summary_message = (
        f"S3 Event Pipeline Summary\n"
        f"Source: s3://{bucket}/{key}\n"
        f"Processed: {results['processed']} records\n"
        f"Skipped: {results['skipped']} records\n"
        f"Errors: {results['errors']}\n"
        f"Time: {datetime.now(timezone.utc).isoformat()}"
    )

    sns_client.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject=f"S3 Pipeline: {results['processed']} records processed",
        Message=summary_message,
    )

    return {
        "statusCode": 200,
        "body": json.dumps(results),
    }
PYEOF

# Package the Lambda
cd /tmp/lambda-s3-processor
zip -q lambda.zip lambda_function.py
echo "Lambda package created: $(ls -lh lambda.zip | awk '{print $5}')"
```

---

### Step 5 — Create the Lambda IAM role and function

```bash
# Create the execution role
cat > /tmp/lambda-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

LAMBDA_ROLE_ARN=$(aws iam create-role \
  --role-name S3HowToLambdaRole \
  --assume-role-policy-document file:///tmp/lambda-trust.json \
  --tags Key=Project,Value=S3HowTo \
  --query Role.Arn \
  --output text)
echo "Lambda Role ARN: $LAMBDA_ROLE_ARN"

# Attach policies
aws iam attach-role-policy \
  --role-name S3HowToLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Inline policy for S3, DynamoDB, SNS access
cat > /tmp/lambda-inline-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "S3Read",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/uploads/*"
    },
    {
      "Sid": "DynamoDBWrite",
      "Effect": "Allow",
      "Action": [
        "dynamodb:PutItem",
        "dynamodb:BatchWriteItem"
      ],
      "Resource": "arn:aws:dynamodb:${AWS_REGION}:${ACCOUNT_ID}:table/${DYNAMO_TABLE}"
    },
    {
      "Sid": "SNSPublish",
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "${SNS_TOPIC_ARN}"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name S3HowToLambdaRole \
  --policy-name S3HowToLambdaPolicy \
  --policy-document file:///tmp/lambda-inline-policy.json

# Wait for IAM propagation
sleep 10

# Create the Lambda function
LAMBDA_ARN=$(aws lambda create-function \
  --function-name "$LAMBDA_FUNCTION" \
  --runtime python3.12 \
  --role "$LAMBDA_ROLE_ARN" \
  --handler lambda_function.lambda_handler \
  --zip-file fileb:///tmp/lambda-s3-processor/lambda.zip \
  --timeout 60 \
  --memory-size 256 \
  --environment "Variables={DYNAMO_TABLE=${DYNAMO_TABLE},SNS_TOPIC_ARN=${SNS_TOPIC_ARN}}" \
  --tags Project=S3HowTo,Lab=06 \
  --query FunctionArn \
  --output text)
echo "Lambda ARN: $LAMBDA_ARN"

# Wait until the function is active
aws lambda wait function-active --function-name "$LAMBDA_FUNCTION"
echo "Lambda function is active"
```

---

### Step 6 — Wire up S3 event notification

```bash
# Grant S3 permission to invoke the Lambda
aws lambda add-permission \
  --function-name "$LAMBDA_FUNCTION" \
  --statement-id S3HowToInvoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn "arn:aws:s3:::${BUCKET_NAME}" \
  --source-account "$ACCOUNT_ID"

# Configure S3 to invoke Lambda on PUT events for uploads/*.json
cat > /tmp/s3-notification.json << EOF
{
  "LambdaFunctionConfigurations": [{
    "LambdaFunctionArn": "${LAMBDA_ARN}",
    "Events": ["s3:ObjectCreated:*"],
    "Filter": {
      "Key": {
        "FilterRules": [
          {"Name": "prefix", "Value": "uploads/"},
          {"Name": "suffix", "Value": ".json"}
        ]
      }
    }
  }]
}
EOF

aws s3api put-bucket-notification-configuration \
  --bucket "$BUCKET_NAME" \
  --notification-configuration file:///tmp/s3-notification.json

# Verify
aws s3api get-bucket-notification-configuration \
  --bucket "$BUCKET_NAME" \
  --query 'LambdaFunctionConfigurations[0].{Function:LambdaFunctionArn,Events:Events,Filter:Filter}' \
  --output json
```

---

### Step 7 — Trigger the pipeline with a real upload

```bash
# Create a JSON file with 5 sample orders
cat > /tmp/orders.json << 'EOF'
[
  {"order_id": "ORD-001", "customer": "Alice Johnson", "product": "Widget A", "quantity": 3, "total": 89.97},
  {"order_id": "ORD-002", "customer": "Bob Smith", "product": "Widget B", "quantity": 1, "total": 24.99},
  {"order_id": "ORD-003", "customer": "Carol White", "product": "Gadget X", "quantity": 2, "total": 149.98},
  {"order_id": "ORD-004", "customer": "Dave Brown", "product": "Widget A", "quantity": 5, "total": 149.95},
  {"order_id": "ORD-005", "customer": "Eve Davis", "product": "Gadget Y", "quantity": 1, "total": 299.00}
]
EOF

# Upload to trigger the Lambda
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "uploads/orders-$(date +%Y%m%d%H%M%S).json" \
  --body /tmp/orders.json \
  --tags "Lab=06&Type=OrderBatch"

echo "File uploaded — waiting 5 seconds for Lambda to process..."
sleep 5

# Check Lambda logs
LOG_GROUP="/aws/lambda/${LAMBDA_FUNCTION}"
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name "$LOG_GROUP" \
  --order-by LastEventTime \
  --descending \
  --query 'logStreams[0].logStreamName' \
  --output text)

aws logs get-log-events \
  --log-group-name "$LOG_GROUP" \
  --log-stream-name "$LOG_STREAM" \
  --query 'events[*].message' \
  --output text

# Verify DynamoDB records
echo "DynamoDB scan results:"
aws dynamodb scan \
  --table-name "$DYNAMO_TABLE" \
  --query 'Items[*].{OrderID:order_id.S,Customer:customer.S,Total:total.N}' \
  --output table
```

**Expected DynamoDB output:**
```
-----------------------------------------------------------------
|                           Scan                               |
+------------------+-------------------+-----------------------+
|  Customer        |  OrderID          |  Total                |
+------------------+-------------------+-----------------------+
|  Alice Johnson   |  ORD-001          |  89.97                |
|  Bob Smith       |  ORD-002          |  24.99                |
|  Carol White     |  ORD-003          |  149.98               |
|  Dave Brown      |  ORD-004          |  149.95               |
|  Eve Davis       |  ORD-005          |  299.0                |
+------------------+-------------------+-----------------------+
```

---

## Cleanup

```bash
# Remove S3 objects
aws s3 rm s3://$BUCKET_NAME/uploads/ --recursive

# Delete Lambda
aws lambda delete-function --function-name "$LAMBDA_FUNCTION"

# Delete DynamoDB table
aws dynamodb delete-table --table-name "$DYNAMO_TABLE"

# Delete SNS topic
aws sns delete-topic --topic-arn "$SNS_TOPIC_ARN"

# Delete IAM role
aws iam delete-role-policy --role-name S3HowToLambdaRole --policy-name S3HowToLambdaPolicy
aws iam detach-role-policy --role-name S3HowToLambdaRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam delete-role --role-name S3HowToLambdaRole

# Remove S3 notification config
aws s3api put-bucket-notification-configuration \
  --bucket "$BUCKET_NAME" \
  --notification-configuration '{}'

echo "Cleanup complete"
```

---

➡️ **Next:** [Lab 07 — Production Patterns](../07-production-patterns/README.md)
