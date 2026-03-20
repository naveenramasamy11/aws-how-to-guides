# Lab 06 — Integration Patterns: S3 → Lambda → DynamoDB + SNS Pipeline

🔴 **Difficulty:** Advanced | ⏱️ **Time:** ~75 min

---

## 🎯 Goal

Build a fully event-driven data pipeline: a CSV file uploaded to S3 triggers a Lambda function that validates, enriches, and persists each row to DynamoDB — then publishes a summary notification to SNS. This is the canonical "serverless ETL" pattern used in real-world data pipelines.

---

## 🧠 Concepts

### Event-Driven Pipeline Pattern

```
                 Data Producers
                 (applications, IoT, batch jobs)
                        │
                        │  CSV file upload
                        ▼
                ┌───────────────┐
                │      S3       │
                │  (raw/ prefix)│
                └───────┬───────┘
                        │  ObjectCreated notification
                        ▼
                ┌───────────────┐    ┌───────────────────────────┐
                │               │    │  DynamoDB                 │
                │    Lambda     │───►│  Table: processed-records │
                │  (ETL worker) │    └───────────────────────────┘
                │               │
                └───────┬───────┘    ┌───────────────────────────┐
                        │            │  SNS Topic                │
                        └───────────►│  (pipeline-alerts)        │
                                     └──────────────┬────────────┘
                                                    │
                                         Email / Slack / SQS
```

### Why Lambda for ETL?

| Factor | Lambda ETL | EC2/ECS ETL |
|--------|-----------|-------------|
| Cost model | Pay per execution | Pay per hour running |
| Scaling | Auto-scales per file | Manual scaling |
| File size limit | 6 MB payload, 512 MB+ /tmp | Unlimited |
| Duration limit | 15 minutes max | Unlimited |
| Ideal for | Files < a few hundred MB | Large files, long-running |

For files > 100 MB, stream-process using S3 Select or multipart reads from `/tmp`.

### Dead Letter Queue (DLQ) for Async Triggers

Async invocations (S3, SNS, EventBridge) retry automatically 2 times on failure, then drop the event. A DLQ catches failed events for later inspection or replay.

```
S3 Event
   │
   ├── Attempt 1: fails
   │   (wait 1 min)
   ├── Attempt 2: fails
   │   (wait 2 min)
   └── Attempt 3: fails → DLQ (SQS) ← engineers investigate here
```

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────────────┐
│                           Data Pipeline                                   │
│                                                                            │
│  CSV Upload                                                                │
│  ─────────                                                                 │
│  aws s3 cp orders.csv s3://pipeline-bucket/raw/                           │
│                    │                                                        │
│                    │ S3 ObjectCreated event                                │
│                    ▼                                                        │
│  ┌─────────────────────────────────────────────────────────────────────┐  │
│  │                      Lambda: etl-pipeline-worker                    │  │
│  │                                                                     │  │
│  │  1. Download CSV from S3 to /tmp                                   │  │
│  │  2. Validate schema (required columns, data types)                 │  │
│  │  3. Enrich (add processed_at timestamp, compute totals)            │  │
│  │  4. Batch write valid rows to DynamoDB                             │  │
│  │  5. Collect invalid rows, write error report back to S3            │  │
│  │  6. Publish summary to SNS                                         │  │
│  │                                                                     │  │
│  │  On failure → DLQ (SQS) for replay                                 │  │
│  └─────────────────────────────────────────────────────────────────────┘  │
│           │                        │                                        │
│           ▼                        ▼                                        │
│  ┌────────────────┐    ┌──────────────────────────────────┐               │
│  │   DynamoDB     │    │   SNS Topic → Email               │               │
│  │ processed-recs │    │   "Processed 95/100 rows (5 err)" │               │
│  └────────────────┘    └──────────────────────────────────┘               │
└──────────────────────────────────────────────────────────────────────────┘
```

---

## 🔢 PoC Steps

### Step 1 — Setup

```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export FUNCTION_NAME="etl-pipeline-worker"
export BUCKET_NAME="lambda-pipeline-${AWS_ACCOUNT_ID}"
export TABLE_NAME="processed-records"
export TOPIC_NAME="pipeline-alerts"
export DLQ_NAME="etl-pipeline-dlq"

echo "Account: ${AWS_ACCOUNT_ID}, Region: ${AWS_REGION}"
```

---

### Step 2 — Create supporting resources

```bash
# S3 bucket with versioning and structured prefixes
aws s3 mb s3://${BUCKET_NAME} --region ${AWS_REGION}
aws s3api put-bucket-versioning \
  --bucket "${BUCKET_NAME}" \
  --versioning-configuration Status=Enabled
aws s3api put-bucket-tagging \
  --bucket "${BUCKET_NAME}" \
  --tagging 'TagSet=[{Key=Project,Value=LambdaLabs},{Key=Lab,Value=06}]'

# Create raw/ and errors/ prefixes (empty placeholder objects)
echo "" | aws s3 cp - s3://${BUCKET_NAME}/raw/.keep
echo "" | aws s3 cp - s3://${BUCKET_NAME}/errors/.keep

echo "S3 bucket created ✅"

# DynamoDB table for processed records
aws dynamodb create-table \
  --table-name "${TABLE_NAME}" \
  --attribute-definitions \
    AttributeName=orderId,AttributeType=S \
    AttributeName=processedAt,AttributeType=S \
  --key-schema \
    AttributeName=orderId,KeyType=HASH \
    AttributeName=processedAt,KeyType=RANGE \
  --billing-mode PAY_PER_REQUEST \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=06 \
  --query '{TableName:TableDescription.TableName,Status:TableDescription.TableStatus}' \
  --output table

aws dynamodb wait table-exists --table-name "${TABLE_NAME}"
echo "DynamoDB table created ✅"

# SNS topic for notifications
TOPIC_ARN=$(aws sns create-topic \
  --name "${TOPIC_NAME}" \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=06 \
  --query 'TopicArn' --output text)

echo "SNS topic: ${TOPIC_ARN}"

# Subscribe your email (optional — comment out if not wanted)
# aws sns subscribe \
#   --topic-arn "${TOPIC_ARN}" \
#   --protocol email \
#   --notification-endpoint "your-email@example.com"

# SQS DLQ for failed events
DLQ_URL=$(aws sqs create-queue \
  --queue-name "${DLQ_NAME}" \
  --tags Project=LambdaLabs,Lab=06 \
  --query 'QueueUrl' --output text)

DLQ_ARN=$(aws sqs get-queue-attributes \
  --queue-url "${DLQ_URL}" \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

echo "DLQ: ${DLQ_ARN}"
```

---

### Step 3 — Create the IAM execution role

```bash
cat > /tmp/etl-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "lambda.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

aws iam create-role \
  --role-name etl-pipeline-lambda-role \
  --assume-role-policy-document file:///tmp/etl-trust.json \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=06 \
  --query 'Role.Arn' --output text

ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/etl-pipeline-lambda-role"
TABLE_ARN="arn:aws:dynamodb:${AWS_REGION}:${AWS_ACCOUNT_ID}:table/${TABLE_NAME}"

cat > /tmp/etl-policy.json << POLICY
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "CloudWatchLogs",
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup","logs:CreateLogStream","logs:PutLogEvents"],
      "Resource": "arn:aws:logs:${AWS_REGION}:${AWS_ACCOUNT_ID}:log-group:/aws/lambda/${FUNCTION_NAME}:*"
    },
    {
      "Sid": "S3ReadInput",
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/raw/*"
    },
    {
      "Sid": "S3WriteErrors",
      "Effect": "Allow",
      "Action": ["s3:PutObject"],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/errors/*"
    },
    {
      "Sid": "DynamoDBWrite",
      "Effect": "Allow",
      "Action": ["dynamodb:BatchWriteItem","dynamodb:PutItem"],
      "Resource": "${TABLE_ARN}"
    },
    {
      "Sid": "SNSPublish",
      "Effect": "Allow",
      "Action": ["sns:Publish"],
      "Resource": "${TOPIC_ARN}"
    },
    {
      "Sid": "SQSSendDLQ",
      "Effect": "Allow",
      "Action": ["sqs:SendMessage"],
      "Resource": "${DLQ_ARN}"
    }
  ]
}
POLICY

aws iam put-role-policy \
  --role-name etl-pipeline-lambda-role \
  --policy-name EtlPipelinePolicy \
  --policy-document file:///tmp/etl-policy.json

sleep 10
echo "IAM role configured ✅"
```

---

### Step 4 — Write the ETL function

```bash
mkdir -p /tmp/etl-function

cat > /tmp/etl-function/lambda_function.py << 'PYEOF'
"""
etl-pipeline-worker: S3 CSV → DynamoDB + SNS notification pipeline.
Triggered by S3 ObjectCreated events on the raw/ prefix.
"""
import os
import csv
import json
import time
import logging
import io
import boto3
from datetime import datetime, timezone
from botocore.exceptions import ClientError

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Clients initialized at cold start (reused across warm invocations)
s3_client = boto3.client("s3")
dynamodb = boto3.resource("dynamodb")
sns_client = boto3.client("sns")

TABLE_NAME = os.environ["TABLE_NAME"]
TOPIC_ARN = os.environ["TOPIC_ARN"]
ERROR_PREFIX = os.environ.get("ERROR_PREFIX", "errors/")

table = dynamodb.Table(TABLE_NAME)

REQUIRED_COLUMNS = {"orderId", "customer", "amount", "status"}
VALID_STATUSES = {"PENDING", "CONFIRMED", "SHIPPED", "DELIVERED", "CANCELLED"}


def validate_row(row: dict, line_number: int) -> tuple[bool, str]:
    """Validate a CSV row. Returns (is_valid, error_message)."""
    missing = REQUIRED_COLUMNS - set(row.keys())
    if missing:
        return False, f"Line {line_number}: missing columns {missing}"

    try:
        amount = float(row["amount"])
        if amount < 0:
            return False, f"Line {line_number}: amount must be non-negative (got {amount})"
    except ValueError:
        return False, f"Line {line_number}: amount '{row['amount']}' is not a number"

    if row["status"].upper() not in VALID_STATUSES:
        return False, f"Line {line_number}: invalid status '{row['status']}'"

    if not row["orderId"].strip():
        return False, f"Line {line_number}: orderId cannot be empty"

    return True, ""


def batch_write_to_dynamodb(items: list[dict]) -> int:
    """Write items to DynamoDB in batches of 25. Returns count of written items."""
    written = 0
    batch_size = 25

    for i in range(0, len(items), batch_size):
        batch = items[i:i + batch_size]
        request_items = {
            TABLE_NAME: [
                {"PutRequest": {"Item": item}} for item in batch
            ]
        }

        retries = 0
        while request_items:
            response = dynamodb.batch_write_item(RequestItems=request_items)
            unprocessed = response.get("UnprocessedItems", {})

            if not unprocessed:
                written += len(batch)
                break

            # Exponential backoff for unprocessed items
            retries += 1
            wait_time = min(2 ** retries * 0.1, 2.0)
            time.sleep(wait_time)
            request_items = unprocessed

    return written


def handler(event, context):
    processed_at = datetime.now(timezone.utc).isoformat()
    results = {
        "files_processed": 0,
        "rows_valid": 0,
        "rows_invalid": 0,
        "rows_written": 0,
        "errors": []
    }

    for record in event.get("Records", []):
        bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]
        logger.info("Processing s3://%s/%s", bucket, key)

        try:
            # Download the CSV from S3
            obj = s3_client.get_object(Bucket=bucket, Key=key)
            content = obj["Body"].read().decode("utf-8")

            reader = csv.DictReader(io.StringIO(content))
            valid_items = []
            invalid_rows = []

            for line_number, row in enumerate(reader, start=2):  # line 1 is header
                is_valid, error_msg = validate_row(row, line_number)

                if is_valid:
                    valid_items.append({
                        "orderId": row["orderId"].strip(),
                        "processedAt": processed_at,
                        "customer": row["customer"].strip(),
                        "amount": str(float(row["amount"])),
                        "status": row["status"].upper().strip(),
                        "sourceFile": key,
                        "sourceAccount": context.invoked_function_arn.split(":")[4]
                    })
                    results["rows_valid"] += 1
                else:
                    invalid_rows.append({"line": line_number, "error": error_msg, "data": dict(row)})
                    results["rows_invalid"] += 1
                    logger.warning("Invalid row: %s", error_msg)

            # Write valid rows to DynamoDB
            if valid_items:
                written = batch_write_to_dynamodb(valid_items)
                results["rows_written"] += written
                logger.info("Wrote %d/%d valid rows to DynamoDB", written, len(valid_items))

            # Write error report back to S3
            if invalid_rows:
                error_key = key.replace("raw/", ERROR_PREFIX).replace(".csv", f"_errors_{int(time.time())}.json")
                error_report = {
                    "sourceFile": key,
                    "processedAt": processed_at,
                    "errorCount": len(invalid_rows),
                    "errors": invalid_rows
                }
                s3_client.put_object(
                    Bucket=bucket,
                    Key=error_key,
                    Body=json.dumps(error_report, indent=2),
                    ContentType="application/json"
                )
                logger.info("Error report written to s3://%s/%s", bucket, error_key)

            results["files_processed"] += 1

        except ClientError as e:
            error_msg = f"AWS error processing s3://{bucket}/{key}: {e}"
            logger.error(error_msg)
            results["errors"].append(error_msg)
        except Exception as e:
            error_msg = f"Unexpected error processing s3://{bucket}/{key}: {e}"
            logger.error(error_msg)
            results["errors"].append(error_msg)

    # Publish summary to SNS
    total = results["rows_valid"] + results["rows_invalid"]
    summary = (
        f"ETL Pipeline Summary\n"
        f"Files processed: {results['files_processed']}\n"
        f"Rows processed: {total} "
        f"({results['rows_valid']} valid, {results['rows_invalid']} invalid)\n"
        f"DynamoDB rows written: {results['rows_written']}\n"
        f"Processing errors: {len(results['errors'])}"
    )

    try:
        sns_client.publish(
            TopicArn=TOPIC_ARN,
            Subject=f"ETL Pipeline: {results['rows_written']} records written",
            Message=summary
        )
        logger.info("SNS notification sent")
    except ClientError as e:
        logger.error("Failed to publish SNS: %s", e)

    logger.info("Pipeline complete: %s", json.dumps(results))
    return results
PYEOF

cd /tmp/etl-function && zip -r function.zip lambda_function.py
```

---

### Step 5 — Deploy the function

```bash
aws lambda create-function \
  --function-name "${FUNCTION_NAME}" \
  --runtime python3.12 \
  --role "${ROLE_ARN}" \
  --handler lambda_function.handler \
  --zip-file fileb:///tmp/etl-function/function.zip \
  --timeout 300 \
  --memory-size 512 \
  --ephemeral-storage '{"Size": 1024}' \
  --environment "Variables={TABLE_NAME=${TABLE_NAME},TOPIC_ARN=${TOPIC_ARN},ERROR_PREFIX=errors/}" \
  --dead-letter-config "TargetArn=${DLQ_ARN}" \
  --description "Lab 06: S3 CSV ETL to DynamoDB + SNS" \
  --tags Project=LambdaLabs,Lab=06 \
  --query '{FunctionName:FunctionName,MemorySize:MemorySize,Timeout:Timeout}' \
  --output table

aws lambda wait function-active --function-name "${FUNCTION_NAME}"

FUNCTION_ARN=$(aws lambda get-function --function-name "${FUNCTION_NAME}" \
  --query 'Configuration.FunctionArn' --output text)

echo "Function deployed: ${FUNCTION_ARN}"
```

---

### Step 6 — Configure S3 trigger

```bash
# Grant S3 permission to invoke Lambda
aws lambda add-permission \
  --function-name "${FUNCTION_NAME}" \
  --statement-id allow-s3-etl-trigger \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn "arn:aws:s3:::${BUCKET_NAME}" \
  --source-account "${AWS_ACCOUNT_ID}" \
  --query 'Statement' --output text

# Configure S3 bucket notification (only raw/ prefix .csv files)
cat > /tmp/s3-notification.json << NOTIF
{
  "LambdaFunctionConfigurations": [{
    "LambdaFunctionArn": "${FUNCTION_ARN}",
    "Events": ["s3:ObjectCreated:*"],
    "Filter": {
      "Key": {
        "FilterRules": [
          {"Name": "prefix", "Value": "raw/"},
          {"Name": "suffix", "Value": ".csv"}
        ]
      }
    }
  }]
}
NOTIF

aws s3api put-bucket-notification-configuration \
  --bucket "${BUCKET_NAME}" \
  --notification-configuration file:///tmp/s3-notification.json

echo "S3 trigger configured ✅"
```

---

### Step 7 — Upload test CSV files and observe the pipeline

```bash
# Create a test CSV with valid and invalid rows
cat > /tmp/orders-test.csv << 'CSV'
orderId,customer,amount,status
ORD-001,Alice Smith,149.99,PENDING
ORD-002,Bob Jones,75.50,CONFIRMED
ORD-003,Carol White,299.00,SHIPPED
ORD-004,Dave Brown,-50.00,PENDING
ORD-005,Eve Davis,invalid_amount,DELIVERED
ORD-006,Frank Wilson,1200.00,UNKNOWN_STATUS
ORD-007,Grace Lee,89.95,DELIVERED
ORD-008,Henry Taylor,445.00,PENDING
CSV

# Upload to S3 to trigger the pipeline
aws s3 cp /tmp/orders-test.csv s3://${BUCKET_NAME}/raw/orders-test.csv
echo "CSV uploaded — pipeline triggered"

# Wait for Lambda to execute
sleep 10

# View Lambda logs
echo "=== Lambda Execution Logs ==="
aws logs tail "/aws/lambda/${FUNCTION_NAME}" --since 1m

# Check DynamoDB — valid rows should be there
echo ""
echo "=== DynamoDB Records ==="
aws dynamodb scan \
  --table-name "${TABLE_NAME}" \
  --query 'Items[*].{orderId:orderId.S,customer:customer.S,amount:amount.S,status:status.S}' \
  --output table

# Check error report in S3
echo ""
echo "=== Error Reports in S3 ==="
aws s3 ls s3://${BUCKET_NAME}/errors/ --recursive
ERROR_KEY=$(aws s3 ls s3://${BUCKET_NAME}/errors/ --recursive | grep "_errors_" | head -1 | awk '{print $4}')
if [ -n "${ERROR_KEY}" ]; then
  aws s3 cp s3://${BUCKET_NAME}/${ERROR_KEY} - | python3 -m json.tool
fi
```

**Expected DynamoDB records:**
```
------------------------------------------------------
|                       Scan                        |
+----------+---------------+--------+----------+
| orderId  | customer      | amount | status   |
+----------+---------------+--------+----------+
| ORD-001  | Alice Smith   | 149.99 | PENDING  |
| ORD-002  | Bob Jones     | 75.5   | CONFIRMED|
| ORD-003  | Carol White   | 299.0  | SHIPPED  |
| ORD-007  | Grace Lee     | 89.95  | DELIVERED|
| ORD-008  | Henry Taylor  | 445.0  | PENDING  |
+----------+---------------+--------+----------+
```

---

### Step 8 — Check the DLQ (simulating a failure)

```bash
# Test what happens when DLQ catches a failure
# Invoke directly with a malformed S3 event to trigger an error path
aws lambda invoke \
  --function-name "${FUNCTION_NAME}" \
  --payload '{
    "Records": [{
      "s3": {
        "bucket": {"name": "nonexistent-bucket-xyz"},
        "object": {"key": "raw/nonexistent.csv"}
      }
    }]
  }' \
  --cli-binary-format raw-in-base64-out \
  /tmp/dlq-test-response.json

python3 -m json.tool /tmp/dlq-test-response.json

# Check DLQ message count
aws sqs get-queue-attributes \
  --queue-url "${DLQ_URL}" \
  --attribute-names ApproximateNumberOfMessages \
  --query 'Attributes' --output table
```

---

## 🧹 Cleanup

```bash
# Delete Lambda
aws lambda delete-function --function-name "${FUNCTION_NAME}"

# Delete DynamoDB table
aws dynamodb delete-table --table-name "${TABLE_NAME}"

# Delete SNS topic
aws sns delete-topic --topic-arn "${TOPIC_ARN}"

# Delete SQS DLQ
aws sqs delete-queue --queue-url "${DLQ_URL}"

# Empty and delete S3 bucket
aws s3 rm s3://${BUCKET_NAME} --recursive
aws s3 rb s3://${BUCKET_NAME}

# Delete IAM role
aws iam delete-role-policy --role-name etl-pipeline-lambda-role --policy-name EtlPipelinePolicy
aws iam delete-role --role-name etl-pipeline-lambda-role

# Delete log group
aws logs delete-log-group --log-group-name "/aws/lambda/${FUNCTION_NAME}" 2>/dev/null

echo "Cleanup complete ✅"
```

---

## ✅ What You Learned

- S3 event notifications deliver per-object triggers — filter by prefix and suffix to scope triggers precisely
- DynamoDB BatchWriteItem with exponential backoff handles throttling in production workloads
- Writing error reports back to S3 creates an auditable, replayable error trail
- Dead Letter Queues (DLQ) catch events that exhaust Lambda's built-in retry mechanism
- The `ephemeral-storage` setting (up to 10 GB) is key for processing large files in `/tmp`
- Source account condition in the S3 permission prevents confused deputy attacks

---

➡️ Next: [Lab 07 — Production Patterns](../07-production-patterns/README.md)
