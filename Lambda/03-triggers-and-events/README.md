# Lab 03 — Triggers & Event Sources: S3, API Gateway, SQS, EventBridge

🟡 **Difficulty:** Intermediate | ⏱️ **Time:** ~45 min

---

## 🎯 Goal

Configure four of the most common Lambda trigger types — S3 events, API Gateway (HTTP), SQS queue consumption, and EventBridge scheduled rules. You will deploy a single multi-purpose function that handles each event source and observe the different event payload shapes.

---

## 🧠 Concepts

### Invocation Models

| Trigger | Model | Error Behaviour | Retry |
|---------|-------|-----------------|-------|
| **API Gateway** | Synchronous | Caller receives error response | No automatic retry |
| **S3** | Asynchronous | Lambda retries 2× then drops | DLQ recommended |
| **SQS** | Poll-based (streaming) | Message returns to queue | Configurable visibility timeout |
| **EventBridge** | Asynchronous | Retry with exponential backoff × 24 hrs | Built-in retry |
| **DynamoDB Streams** | Poll-based (streaming) | Shard blocked until processed | At-least-once |
| **Kinesis** | Poll-based (streaming) | Shard blocked until processed | At-least-once |

### Event Source Mapping vs. Resource-Based Policy

```
POLL-BASED TRIGGERS (SQS, DynamoDB, Kinesis, MQ):
  → Lambda service polls the source on your behalf
  → Requires an "Event Source Mapping" resource
  → Lambda role needs READ permissions on the source

PUSH TRIGGERS (S3, API GW, SNS, EventBridge):
  → Source service invokes Lambda directly
  → Requires a "Resource-Based Policy" on the Lambda function
  → Granting the source service permission to call lambda:InvokeFunction
```

---

## 🏗️ Architecture

```
┌──────────────┐    PutObject     ┌───────────────┐
│      S3      │ ──────────────►  │               │
│   (bucket)   │                  │               │
└──────────────┘                  │  multi-event  │      CloudWatch
                                  │    Lambda     │ ────► Logs
┌──────────────┐  HTTP GET/POST   │   function    │
│  API Gateway │ ──────────────►  │               │
│   (HTTP API) │                  │               │
└──────────────┘                  └───────────────┘
                                        ▲     ▲
┌──────────────┐  Polling (ESM)         │     │
│     SQS      │ ───────────────────────┘     │
│    Queue     │                               │
└──────────────┘                               │
                                               │
┌──────────────┐  Push (cron)                  │
│ EventBridge  │ ──────────────────────────────┘
│ (rule 1/min) │
└──────────────┘
```

---

## 🔢 PoC Steps

### Step 1 — Environment setup

```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export FUNCTION_NAME="multi-event-handler"
export BUCKET_NAME="lambda-trigger-lab-${AWS_ACCOUNT_ID}"
export QUEUE_NAME="lambda-trigger-queue"

echo "Setup complete — Account: ${AWS_ACCOUNT_ID}"
```

---

### Step 2 — Create the IAM execution role

```bash
cat > /tmp/trust-policy.json << 'EOF'
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
  --role-name multi-event-lambda-role \
  --assume-role-policy-document file:///tmp/trust-policy.json \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=03 \
  --query 'Role.Arn' --output text

# Attach managed policies
for policy in AWSLambdaBasicExecutionRole AWSLambdaSQSQueueExecutionRole; do
  aws iam attach-role-policy \
    --role-name multi-event-lambda-role \
    --policy-arn "arn:aws:iam::aws:policy/service-role/${policy}"
  echo "Attached: ${policy}"
done

sleep 10
ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/multi-event-lambda-role"
echo "Role ARN: ${ROLE_ARN}"
```

---

### Step 3 — Write the multi-source handler

```bash
mkdir -p /tmp/multi-event

cat > /tmp/multi-event/lambda_function.py << 'PYEOF'
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)


def detect_source(event):
    """Detect which AWS service triggered this invocation."""
    if "Records" in event:
        first = event["Records"][0]
        if first.get("eventSource") == "aws:sqs":
            return "SQS"
        if "s3" in first:
            return "S3"
        if first.get("eventSource") == "aws:dynamodb":
            return "DynamoDB"
    if "source" in event and event.get("source", "").startswith("aws."):
        return "EventBridge"
    if "requestContext" in event and "http" in event.get("requestContext", {}):
        return "API_GATEWAY_HTTP"
    if "httpMethod" in event:
        return "API_GATEWAY_REST"
    return "DIRECT_INVOKE"


def handle_s3(event):
    results = []
    for record in event.get("Records", []):
        bucket = record["s3"]["bucket"]["name"]
        key = record["s3"]["object"]["key"]
        size = record["s3"]["object"].get("size", 0)
        results.append(f"S3 event: {record['eventName']} | s3://{bucket}/{key} ({size} bytes)")
    return results


def handle_sqs(event):
    results = []
    for record in event.get("Records", []):
        body = record.get("body", "")
        msg_id = record.get("messageId", "")
        results.append(f"SQS message [{msg_id[:8]}...]: {body[:100]}")
    return results


def handle_api_gateway(event):
    method = event.get("requestContext", {}).get("http", {}).get("method") or event.get("httpMethod", "UNKNOWN")
    path = event.get("rawPath") or event.get("path", "/")
    body = event.get("body", "")
    return {
        "statusCode": 200,
        "headers": {"Content-Type": "application/json"},
        "body": json.dumps({
            "message": "Lambda handled your request!",
            "method": method,
            "path": path,
            "echoBody": body
        })
    }


def handle_eventbridge(event):
    return f"EventBridge rule fired: {event.get('detail-type', 'unknown')} from {event.get('source', 'unknown')}"


def handler(event, context):
    source = detect_source(event)
    logger.info("Source: %s | RequestId: %s", source, context.aws_request_id)
    logger.info("Event: %s", json.dumps(event)[:500])

    if source == "S3":
        result = handle_s3(event)
    elif source == "SQS":
        result = handle_sqs(event)
    elif source in ("API_GATEWAY_HTTP", "API_GATEWAY_REST"):
        return handle_api_gateway(event)
    elif source == "EventBridge":
        result = handle_eventbridge(event)
    else:
        result = f"Direct invoke — event keys: {list(event.keys())}"

    logger.info("Result: %s", result)
    return {"source": source, "result": result, "requestId": context.aws_request_id}
PYEOF

cd /tmp/multi-event && zip -r function.zip lambda_function.py
```

---

### Step 4 — Deploy the function

```bash
aws lambda create-function \
  --function-name "${FUNCTION_NAME}" \
  --runtime python3.12 \
  --role "${ROLE_ARN}" \
  --handler lambda_function.handler \
  --zip-file fileb:///tmp/multi-event/function.zip \
  --timeout 30 \
  --memory-size 128 \
  --tags Project=LambdaLabs,Lab=03 \
  --query '{FunctionName:FunctionName,State:State}' \
  --output table

aws lambda wait function-active --function-name "${FUNCTION_NAME}"
FUNCTION_ARN=$(aws lambda get-function --function-name "${FUNCTION_NAME}" --query 'Configuration.FunctionArn' --output text)
echo "Function ARN: ${FUNCTION_ARN}"
```

---

### Step 5 — Trigger 1: S3 event notification

```bash
# Create the S3 bucket
aws s3 mb s3://${BUCKET_NAME} --region ${AWS_REGION}
aws s3api put-bucket-tags \
  --bucket ${BUCKET_NAME} \
  --tagging 'TagSet=[{Key=Project,Value=LambdaLabs},{Key=Lab,Value=03}]'

# Grant S3 permission to invoke Lambda
aws lambda add-permission \
  --function-name "${FUNCTION_NAME}" \
  --statement-id s3-invoke \
  --action lambda:InvokeFunction \
  --principal s3.amazonaws.com \
  --source-arn "arn:aws:s3:::${BUCKET_NAME}" \
  --source-account "${AWS_ACCOUNT_ID}" \
  --query 'Statement' --output text

# Configure S3 to notify Lambda on object creation
cat > /tmp/s3-notification.json << NOTIF
{
  "LambdaFunctionConfigurations": [{
    "LambdaFunctionArn": "${FUNCTION_ARN}",
    "Events": ["s3:ObjectCreated:*"],
    "Filter": {
      "Key": {
        "FilterRules": [{"Name": "suffix", "Value": ".json"}]
      }
    }
  }]
}
NOTIF

aws s3api put-bucket-notification-configuration \
  --bucket "${BUCKET_NAME}" \
  --notification-configuration file:///tmp/s3-notification.json

echo "S3 trigger configured ✅"

# Test: upload a file
echo '{"event": "test", "lab": "03"}' | aws s3 cp - s3://${BUCKET_NAME}/test-event.json

sleep 3
echo "Recent Lambda logs:"
aws logs tail "/aws/lambda/${FUNCTION_NAME}" --since 30s
```

**Expected log output:**
```
Source: S3 | RequestId: ...
Event: {"Records": [{"eventName": "ObjectCreated:Put", "s3": {"bucket": {"name": "lambda-trigger-lab-..."}, "object": {"key": "test-event.json", "size": 31}}}]}
Result: ['S3 event: ObjectCreated:Put | s3://lambda-trigger-lab-.../test-event.json (31 bytes)']
```

---

### Step 6 — Trigger 2: SQS queue

```bash
# Create the SQS queue
QUEUE_URL=$(aws sqs create-queue \
  --queue-name "${QUEUE_NAME}" \
  --attributes VisibilityTimeout=60,MessageRetentionPeriod=86400 \
  --tags Project=LambdaLabs,Lab=03 \
  --query 'QueueUrl' --output text)

QUEUE_ARN=$(aws sqs get-queue-attributes \
  --queue-url "${QUEUE_URL}" \
  --attribute-names QueueArn \
  --query 'Attributes.QueueArn' --output text)

echo "Queue URL: ${QUEUE_URL}"
echo "Queue ARN: ${QUEUE_ARN}"

# Create the Event Source Mapping (Lambda polls SQS)
aws lambda create-event-source-mapping \
  --function-name "${FUNCTION_NAME}" \
  --event-source-arn "${QUEUE_ARN}" \
  --batch-size 5 \
  --maximum-batching-window-in-seconds 5 \
  --query '{UUID:UUID,State:State,BatchSize:BatchSize}' \
  --output table

echo "SQS trigger configured ✅"

# Test: send messages
for i in 1 2 3; do
  aws sqs send-message \
    --queue-url "${QUEUE_URL}" \
    --message-body "{\"messageNumber\": ${i}, \"text\": \"Hello from SQS message ${i}\"}" \
    --query 'MessageId' --output text
done

sleep 15  # Allow Lambda to poll and process
aws logs tail "/aws/lambda/${FUNCTION_NAME}" --since 30s
```

---

### Step 7 — Trigger 3: EventBridge scheduled rule

```bash
# Create an EventBridge rule that fires every minute
aws events put-rule \
  --name "lambda-lab03-heartbeat" \
  --schedule-expression "rate(1 minute)" \
  --description "Lab 03: fires Lambda every minute" \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=03 \
  --query 'RuleArn' --output text

# Grant EventBridge permission to invoke Lambda
aws lambda add-permission \
  --function-name "${FUNCTION_NAME}" \
  --statement-id eventbridge-invoke \
  --action lambda:InvokeFunction \
  --principal events.amazonaws.com \
  --source-arn "arn:aws:events:${AWS_REGION}:${AWS_ACCOUNT_ID}:rule/lambda-lab03-heartbeat" \
  --query 'Statement' --output text

# Set Lambda as the rule target
aws events put-targets \
  --rule lambda-lab03-heartbeat \
  --targets "Id=lambda-target,Arn=${FUNCTION_ARN},Input={\"source\":\"aws.events\",\"detail-type\":\"Scheduled Event\",\"detail\":{}}"

echo "EventBridge trigger configured ✅"
echo "Waiting 70 seconds for the first scheduled invocation..."
sleep 70
aws logs tail "/aws/lambda/${FUNCTION_NAME}" --since 2m
```

---

### Step 8 — Trigger 4: HTTP API Gateway

```bash
# Create an HTTP API
API_ID=$(aws apigatewayv2 create-api \
  --name "lambda-lab03-api" \
  --protocol-type HTTP \
  --tags Project=LambdaLabs,Lab=03 \
  --query 'ApiId' --output text)

echo "API ID: ${API_ID}"

# Create Lambda integration
INTEGRATION_ID=$(aws apigatewayv2 create-integration \
  --api-id "${API_ID}" \
  --integration-type AWS_PROXY \
  --integration-uri "${FUNCTION_ARN}" \
  --payload-format-version 2.0 \
  --query 'IntegrationId' --output text)

# Create a catch-all route
aws apigatewayv2 create-route \
  --api-id "${API_ID}" \
  --route-key "ANY /{proxy+}" \
  --target "integrations/${INTEGRATION_ID}" \
  --query '{RouteId:RouteId,RouteKey:RouteKey}' \
  --output table

# Grant API Gateway permission to invoke Lambda
aws lambda add-permission \
  --function-name "${FUNCTION_NAME}" \
  --statement-id apigw-invoke \
  --action lambda:InvokeFunction \
  --principal apigateway.amazonaws.com \
  --source-arn "arn:aws:execute-api:${AWS_REGION}:${AWS_ACCOUNT_ID}:${API_ID}/*" \
  --query 'Statement' --output text

# Deploy to $default stage
aws apigatewayv2 create-stage \
  --api-id "${API_ID}" \
  --stage-name '$default' \
  --auto-deploy \
  --tags Project=LambdaLabs,Lab=03 \
  --query '{StageName:StageName,DeploymentId:DeploymentId}' \
  --output table

API_ENDPOINT="https://${API_ID}.execute-api.${AWS_REGION}.amazonaws.com"
echo "API Endpoint: ${API_ENDPOINT}"

# Test the HTTP trigger
echo "Testing GET request:"
curl -s -X GET "${API_ENDPOINT}/hello?name=Lambda" | python3 -m json.tool

echo ""
echo "Testing POST request:"
curl -s -X POST "${API_ENDPOINT}/process" \
  -H "Content-Type: application/json" \
  -d '{"data": "test payload"}' | python3 -m json.tool
```

**Expected output:**
```json
{
    "message": "Lambda handled your request!",
    "method": "GET",
    "path": "/hello",
    "echoBody": ""
}
```

---

## 🧹 Cleanup

```bash
# Remove EventBridge rule
aws events remove-targets --rule lambda-lab03-heartbeat --ids lambda-target
aws events delete-rule --name lambda-lab03-heartbeat

# Remove API Gateway
aws apigatewayv2 delete-api --api-id "${API_ID}"

# Remove SQS event source mapping (get UUID first)
ESM_UUID=$(aws lambda list-event-source-mappings \
  --function-name "${FUNCTION_NAME}" \
  --query 'EventSourceMappings[0].UUID' --output text)
aws lambda delete-event-source-mapping --uuid "${ESM_UUID}"

# Delete the SQS queue
aws sqs delete-queue --queue-url "${QUEUE_URL}"

# Empty and delete S3 bucket
aws s3 rm s3://${BUCKET_NAME} --recursive
aws s3 rb s3://${BUCKET_NAME}

# Delete Lambda function
aws lambda delete-function --function-name "${FUNCTION_NAME}"

# Delete log group
aws logs delete-log-group --log-group-name "/aws/lambda/${FUNCTION_NAME}"

# Delete IAM role
aws iam detach-role-policy --role-name multi-event-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam detach-role-policy --role-name multi-event-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaSQSQueueExecutionRole
aws iam delete-role --role-name multi-event-lambda-role

echo "All resources cleaned up ✅"
```

---

## ✅ What You Learned

- S3 and EventBridge use **push** triggers — they invoke Lambda directly via resource-based policies
- SQS uses a **poll-based** Event Source Mapping — Lambda service polls the queue on your behalf
- Each trigger delivers a different event shape; detecting the source enables a unified multi-trigger handler
- API Gateway HTTP API uses payload format version 2.0 and requires `aws:apigateway.amazonaws.com` permission

---

➡️ Next: [Lab 04 — Advanced Features](../04-advanced-features/README.md)
