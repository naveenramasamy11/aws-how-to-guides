# Lab 02 — Deploy Your First Lambda Function (CLI + Python + Boto3)

🟢 **Difficulty:** Beginner | ⏱️ **Time:** ~30 min

---

## 🎯 Goal

Create, deploy, invoke, and delete a Python Lambda function entirely from the command line. By the end you will understand the complete deployment lifecycle — IAM execution role, zip packaging, `create-function`, synchronous invocation, and log retrieval.

---

## 🧠 Concepts

### The Three Things Every Lambda Needs

```
┌─────────────────────────────────────────────────────┐
│                  Lambda Function                     │
│                                                      │
│  1. CODE          — your handler file(s) in a zip   │
│  2. EXECUTION ROLE — IAM role Lambda assumes at      │
│                      runtime (allows CloudWatch,    │
│                      and whatever your code needs)  │
│  3. RUNTIME        — language environment to run in  │
└─────────────────────────────────────────────────────┘
```

### What Happens When You Create a Function

```
aws lambda create-function
          │
          ├── Lambda uploads your zip to an internal S3 bucket
          ├── Registers the function metadata (runtime, handler, role)
          └── Function is ready for invocation
```

---

## 🏗️ Architecture

```
Developer
    │
    │  aws lambda invoke
    ▼
Lambda Service
    │
    │  Passes event JSON
    ▼
┌─────────────────────────────────────┐
│         Execution Environment        │
│                                      │
│  Python 3.12                         │
│  Handler: lambda_function.handler   │
│                                      │
│  ┌──────────────────────────────┐   │
│  │  def handler(event, context):│   │
│  │    return {"statusCode": 200 │   │
│  │            "body": "Hello!"} │   │
│  └──────────────────────────────┘   │
└──────────┬──────────────────────────┘
           │ CloudWatch Logs
           ▼
    /aws/lambda/hello-world-lambda
```

---

## 🔢 PoC Steps

### Step 1 — Set environment variables

```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export FUNCTION_NAME="hello-world-lambda"

echo "Account: ${AWS_ACCOUNT_ID}, Region: ${AWS_REGION}"
```

**Expected output:**
```
Account: 123456789012, Region: us-east-1
```

---

### Step 2 — Create the IAM execution role

Lambda needs a role to assume at runtime. The trust policy allows Lambda to assume the role.

```bash
# Create the trust policy document
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

# Create the execution role
aws iam create-role \
  --role-name lambda-basic-execution-role \
  --assume-role-policy-document file:///tmp/lambda-trust-policy.json \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=02 \
  --query 'Role.[RoleName,Arn]' \
  --output table
```

**Expected output:**
```
------------------------------------------------------------------------------------------
|                                       CreateRole                                       |
+-----------------------------+----------------------------------------------------------+
|  lambda-basic-execution-role|  arn:aws:iam::123456789012:role/lambda-basic-execution-role |
+-----------------------------+----------------------------------------------------------+
```

```bash
# Attach the AWS managed policy for basic Lambda execution (CloudWatch Logs)
aws iam attach-role-policy \
  --role-name lambda-basic-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

echo "Policy attached. Waiting 10 seconds for IAM propagation..."
sleep 10
```

---

### Step 3 — Write the function code

```bash
mkdir -p /tmp/hello-lambda

cat > /tmp/hello-lambda/lambda_function.py << 'PYEOF'
import json
import logging

logger = logging.getLogger()
logger.setLevel(logging.INFO)

def handler(event, context):
    """
    Simple Lambda handler that echoes input and returns a greeting.
    """
    logger.info("Received event: %s", json.dumps(event))

    name = event.get("name", "World")

    response = {
        "statusCode": 200,
        "message": f"Hello, {name}! 👋",
        "requestId": context.aws_request_id,
        "remainingTimeMs": context.get_remaining_time_in_millis()
    }

    logger.info("Returning response: %s", json.dumps(response))
    return response
PYEOF

echo "Function code written:"
cat /tmp/hello-lambda/lambda_function.py
```

---

### Step 4 — Package into a zip file

```bash
cd /tmp/hello-lambda
zip -r function.zip lambda_function.py

echo "Zip contents:"
unzip -l function.zip
```

**Expected output:**
```
Archive:  function.zip
  Length      Date    Time    Name
---------  ---------- -----   ----
      ...  2024-01-01 00:00   lambda_function.py
---------                     -------
      ...                     1 file
```

---

### Step 5 — Create the Lambda function

```bash
ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/lambda-basic-execution-role"

aws lambda create-function \
  --function-name "${FUNCTION_NAME}" \
  --runtime python3.12 \
  --role "${ROLE_ARN}" \
  --handler lambda_function.handler \
  --zip-file fileb:///tmp/hello-lambda/function.zip \
  --timeout 30 \
  --memory-size 128 \
  --description "Lab 02: Hello World Lambda function" \
  --tags Project=LambdaLabs,Lab=02 \
  --query '{FunctionName:FunctionName, Runtime:Runtime, State:State, LastModified:LastModified}' \
  --output table
```

**Expected output:**
```
-----------------------------------------------------------------------------------
|                                  CreateFunction                                 |
+---------------+------------------+----------+-----------------------------------+
| FunctionName  |  LastModified    |  Runtime | State                             |
+---------------+------------------+----------+-----------------------------------+
| hello-world.. | 2024-01-01T..Z  | python3.12 | Pending                         |
+---------------+------------------+----------+-----------------------------------+
```

```bash
# Wait for function to become Active
aws lambda wait function-active --function-name "${FUNCTION_NAME}"
echo "Function is now Active ✅"
```

---

### Step 6 — Invoke the function synchronously

```bash
# Invoke with a JSON payload
aws lambda invoke \
  --function-name "${FUNCTION_NAME}" \
  --payload '{"name": "Lambda Learner"}' \
  --cli-binary-format raw-in-base64-out \
  --query 'StatusCode' \
  /tmp/response.json

echo "HTTP Status Code:"
cat <(echo "StatusCode:")
echo "Response body:"
cat /tmp/response.json | python3 -m json.tool
```

**Expected output:**
```
200
Response body:
{
    "statusCode": 200,
    "message": "Hello, Lambda Learner! 👋",
    "requestId": "abc123-def456-...",
    "remainingTimeMs": 29850
}
```

---

### Step 7 — View CloudWatch logs

```bash
# Get the latest log stream for this function
LOG_STREAM=$(aws logs describe-log-streams \
  --log-group-name "/aws/lambda/${FUNCTION_NAME}" \
  --order-by LastEventTime \
  --descending \
  --limit 1 \
  --query 'logStreams[0].logStreamName' \
  --output text)

echo "Latest log stream: ${LOG_STREAM}"

# Fetch the log events
aws logs get-log-events \
  --log-group-name "/aws/lambda/${FUNCTION_NAME}" \
  --log-stream-name "${LOG_STREAM}" \
  --query 'events[*].message' \
  --output text
```

**Expected output:**
```
START RequestId: abc123... Version: $LATEST
Received event: {"name": "Lambda Learner"}
Returning response: {"statusCode": 200, "message": "Hello, Lambda Learner! 👋", ...}
END RequestId: abc123...
REPORT RequestId: abc123... Duration: 2.45 ms  Billed Duration: 3 ms  Memory Size: 128 MB  Max Memory Used: 37 MB
```

---

### Step 8 — Invoke asynchronously

```bash
# Async invoke returns 202 immediately; Lambda queues the request
aws lambda invoke \
  --function-name "${FUNCTION_NAME}" \
  --invocation-type Event \
  --payload '{"name": "Async Caller"}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/async-response.json

echo "Async invoke status code (should be 202):"
cat /tmp/async-response.json

# The actual response is in CloudWatch Logs — check after a few seconds
sleep 5
aws logs tail "/aws/lambda/${FUNCTION_NAME}" --since 1m
```

---

### Step 9 — Update function code and republish

```bash
# Add a new version of the code
cat > /tmp/hello-lambda/lambda_function.py << 'PYEOF'
import json
import logging
import platform

logger = logging.getLogger()
logger.setLevel(logging.INFO)

# Module-level init (cold start only)
PYTHON_VERSION = platform.python_version()
logger.info("Cold start: Python %s", PYTHON_VERSION)

def handler(event, context):
    name = event.get("name", "World")
    response = {
        "statusCode": 200,
        "message": f"Hello, {name}! Running Python {PYTHON_VERSION}",
        "requestId": context.aws_request_id
    }
    logger.info("Response: %s", json.dumps(response))
    return response
PYEOF

cd /tmp/hello-lambda
zip -r function.zip lambda_function.py

# Update the function code
aws lambda update-function-code \
  --function-name "${FUNCTION_NAME}" \
  --zip-file fileb://function.zip \
  --query '{FunctionName:FunctionName,CodeSha256:CodeSha256,LastModified:LastModified}' \
  --output table

# Wait for update to complete
aws lambda wait function-updated --function-name "${FUNCTION_NAME}"

# Publish a named version
aws lambda publish-version \
  --function-name "${FUNCTION_NAME}" \
  --description "v2 - includes Python version info" \
  --query '{FunctionName:FunctionName,Version:Version,Description:Description}' \
  --output table
```

**Expected output:**
```
-----------------------------------------------------------
|                    PublishVersion                       |
+--------------+-------+---------------------------------+
| FunctionName | Version | Description                  |
+--------------+-------+---------------------------------+
| hello-world-lambda | 1 | v2 - includes Python version info |
+--------------+-------+---------------------------------+
```

```bash
# Invoke the updated function
aws lambda invoke \
  --function-name "${FUNCTION_NAME}" \
  --payload '{"name": "Updated Caller"}' \
  --cli-binary-format raw-in-base64-out \
  /tmp/response-v2.json && python3 -m json.tool /tmp/response-v2.json
```

---

### Step 10 — Deploy using Boto3 (Python SDK)

```python
#!/usr/bin/env python3
"""
deploy_lambda.py — programmatically invoke a Lambda function using Boto3.
Run: python3 deploy_lambda.py
"""
import json
import boto3
import base64

FUNCTION_NAME = "hello-world-lambda"
REGION = "us-east-1"

client = boto3.client("lambda", region_name=REGION)

# Synchronous invocation
response = client.invoke(
    FunctionName=FUNCTION_NAME,
    InvocationType="RequestResponse",
    Payload=json.dumps({"name": "Boto3 Caller"})
)

status_code = response["StatusCode"]
payload = json.loads(response["Payload"].read())

print(f"Status: {status_code}")
print(f"Response: {json.dumps(payload, indent=2)}")

# Get function configuration
config = client.get_function_configuration(FunctionName=FUNCTION_NAME)
print(f"\nFunction config:")
print(f"  Runtime: {config['Runtime']}")
print(f"  Memory: {config['MemorySize']} MB")
print(f"  Timeout: {config['Timeout']} s")
print(f"  Handler: {config['Handler']}")
```

```bash
# Save and run the script
cat > /tmp/deploy_lambda.py << 'SCRIPT'
import json
import boto3

FUNCTION_NAME = "hello-world-lambda"
REGION = "us-east-1"

client = boto3.client("lambda", region_name=REGION)

response = client.invoke(
    FunctionName=FUNCTION_NAME,
    InvocationType="RequestResponse",
    Payload=json.dumps({"name": "Boto3 Caller"})
)

status_code = response["StatusCode"]
payload = json.loads(response["Payload"].read())

print(f"Status: {status_code}")
print(f"Response: {json.dumps(payload, indent=2)}")

config = client.get_function_configuration(FunctionName=FUNCTION_NAME)
print(f"\nRuntime: {config['Runtime']}, Memory: {config['MemorySize']} MB, Timeout: {config['Timeout']} s")
SCRIPT

python3 /tmp/deploy_lambda.py
```

---

## 🧹 Cleanup

```bash
# Delete the Lambda function (all versions and aliases)
aws lambda delete-function --function-name "${FUNCTION_NAME}"
echo "Function deleted ✅"

# Remove the CloudWatch log group
aws logs delete-log-group \
  --log-group-name "/aws/lambda/${FUNCTION_NAME}"
echo "Log group deleted ✅"

# Detach the managed policy from the role
aws iam detach-role-policy \
  --role-name lambda-basic-execution-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Delete the IAM role
aws iam delete-role --role-name lambda-basic-execution-role
echo "IAM role deleted ✅"

# Clean up local files
rm -rf /tmp/hello-lambda /tmp/response*.json /tmp/async-response.json /tmp/deploy_lambda.py /tmp/lambda-trust-policy.json
echo "Local files cleaned up ✅"
```

---

## ✅ What You Learned

- Lambda functions require three things: code, execution role, and a runtime
- The `AWSLambdaBasicExecutionRole` managed policy is the minimum needed for CloudWatch Logs
- Synchronous invocations return the function response directly; async returns 202 immediately
- Module-level code runs once per cold start — a key optimization opportunity
- `publish-version` creates an immutable snapshot of function code + config

---

➡️ Next: [Lab 03 — Triggers & Event Sources](../03-triggers-and-events/README.md)
