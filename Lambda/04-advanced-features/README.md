# Lab 04 — Advanced Features: Layers, Env Vars, Concurrency & SnapStart

🟡 **Difficulty:** Intermediate | ⏱️ **Time:** ~60 min

---

## 🎯 Goal

Master four critical Lambda features that separate hobby functions from production-grade deployments: **Layers** (shared dependencies), **Environment Variables** (config management), **Concurrency controls** (reserved vs. provisioned), and **SnapStart** (Java cold-start elimination). Each section includes a working PoC.

---

## 🧠 Concepts

### 1. Lambda Layers

A Layer is a `.zip` archive containing libraries, a custom runtime, or data files. It is mounted at `/opt` inside the execution environment.

```
Without Layers                     With Layers
─────────────────                  ─────────────────────────────────
function.zip (50 MB)               function.zip (2 MB — code only)
  ├── lambda_function.py                └── lambda_function.py
  ├── boto3/ (10 MB)
  ├── pandas/ (20 MB)              Layer: pandas-numpy-layer (50 MB)
  └── numpy/ (20 MB)                 mounted at /opt during execution
```

Benefits:
- Multiple functions share one layer → smaller deployment packages
- Layer updates propagate independently of function code
- Up to 5 layers per function; total unzipped size ≤ 250 MB
- Layer versions are immutable and can be made public

### 2. Environment Variables

Lambda encrypts environment variables at rest with the Lambda service key (or your own KMS CMK). They appear as OS environment variables inside the handler.

| Use case | Approach |
|----------|----------|
| Non-sensitive config | Environment variables directly |
| Sensitive secrets | Secrets Manager reference + fetch in code |
| Feature flags | SSM Parameter Store + fetch at cold start |

### 3. Concurrency Controls

```
Reserved Concurrency             Provisioned Concurrency
─────────────────────            ──────────────────────────────
Hard cap on parallel executions  Pre-initialized environments

Use for:                         Use for:
• Throttling noisy functions     • Latency-sensitive APIs
• Protecting downstream DBs      • Eliminating cold starts
• Cost control                   • Consistent p99 response times

Cost: None extra                 Cost: billed per GB-second allocated
```

### 4. SnapStart (Java)

SnapStart takes a snapshot of a fully initialized Java execution environment after the Init phase and caches it in Amazon S3. Subsequent cold starts restore from the snapshot instead of re-running JVM startup + application initialization.

```
Normal Java cold start:     SnapStart cold start:
─────────────────────────   ──────────────────────
JVM startup (1–3 s)          Restore snapshot (100 ms)
Spring/Quarkus init           No init code re-runs
Your module init      ────► Handler runs immediately
Handler runs
```

---

## 🏗️ Architecture

```
                     ┌─────────────────────────────────────────┐
                     │          Lambda Layer Store               │
                     │  pandas-layer-v1 (arn:aws:lambda:...)    │
                     └──────────────────┬──────────────────────┘
                                        │ mounted at /opt
                                        ▼
┌───────────┐    invoke    ┌────────────────────────────────────┐
│  Caller   │ ──────────► │         Lambda Function             │
└───────────┘              │  Env vars: TABLE_NAME, LOG_LEVEL   │
                           │  Reserved concurrency: 100          │
                           │  Provisioned concurrency: 10        │
                           │  Code: analytics_function.py        │
                           │  Layer: /opt/python/pandas          │
                           └────────────────────────────────────┘
```

---

## 🔢 PoC Steps

### Step 1 — Environment setup

```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export FUNCTION_NAME="advanced-lambda-demo"
export LAYER_NAME="pandas-analytics-layer"

ROLE_ARN=$(aws iam get-role --role-name lambda-basic-execution-role \
  --query 'Role.Arn' --output text 2>/dev/null || echo "NEED_TO_CREATE")

if [ "${ROLE_ARN}" = "NEED_TO_CREATE" ]; then
  cat > /tmp/trust.json << 'EOF'
{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"lambda.amazonaws.com"},"Action":"sts:AssumeRole"}]}
EOF
  ROLE_ARN=$(aws iam create-role --role-name lambda-basic-execution-role \
    --assume-role-policy-document file:///tmp/trust.json \
    --tags Key=Project,Value=LambdaLabs \
    --query 'Role.Arn' --output text)
  aws iam attach-role-policy --role-name lambda-basic-execution-role \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
  sleep 10
fi

echo "Role ARN: ${ROLE_ARN}"
```

---

### Step 2 — Build and publish a Lambda Layer (pandas + numpy)

```bash
# Build the layer package locally using a Docker-like approach with pip
mkdir -p /tmp/layer-build/python
pip3 install pandas numpy --target /tmp/layer-build/python \
  --platform manylinux2014_x86_64 \
  --implementation cp \
  --python-version 3.12 \
  --only-binary=:all: \
  --quiet 2>/dev/null || \
pip3 install pandas numpy --target /tmp/layer-build/python --quiet

# Create the zip
cd /tmp/layer-build
zip -r pandas-layer.zip python/ -q

echo "Layer zip size: $(du -sh pandas-layer.zip | cut -f1)"

# Publish the layer version
LAYER_ARN=$(aws lambda publish-layer-version \
  --layer-name "${LAYER_NAME}" \
  --description "pandas + numpy for Python 3.12" \
  --zip-file fileb://pandas-layer.zip \
  --compatible-runtimes python3.12 \
  --compatible-architectures x86_64 \
  --query 'LayerVersionArn' \
  --output text)

echo "Published layer: ${LAYER_ARN}"

# Verify the layer
aws lambda get-layer-version-by-arn \
  --arn "${LAYER_ARN}" \
  --query '{LayerName:LayerName,Version:Version,CompatibleRuntimes:CompatibleRuntimes}' \
  --output table
```

**Expected output:**
```
-----------------------------------------------
|         GetLayerVersionByArn                |
+--------------------+--------+---------------+
|  LayerName         |Version | CompatibleRuntimes |
+--------------------+--------+---------------+
|  pandas-analytics  |  1     | python3.12    |
+--------------------+--------+---------------+
```

---

### Step 3 — Write the function that uses the layer

```bash
mkdir -p /tmp/advanced-function

cat > /tmp/advanced-function/lambda_function.py << 'PYEOF'
import os
import json
import logging
import pandas as pd
import numpy as np

# Read config from environment variables (set at cold start)
LOG_LEVEL = os.environ.get("LOG_LEVEL", "INFO")
APP_ENV = os.environ.get("APP_ENV", "dev")
MAX_ROWS = int(os.environ.get("MAX_ROWS", "100"))

logger = logging.getLogger()
logger.setLevel(getattr(logging, LOG_LEVEL, logging.INFO))
logger.info("Cold start — env=%s, max_rows=%d, pandas=%s", APP_ENV, MAX_ROWS, pd.__version__)


def handler(event, context):
    """
    Analyze a dataset passed in the event using pandas.
    Event shape: {"data": [[1, 2, 3], [4, 5, 6], ...], "columns": ["a", "b", "c"]}
    """
    data = event.get("data", [[1, 2, 3], [4, 5, 6], [7, 8, 9], [10, 11, 12]])
    columns = event.get("columns", ["value_a", "value_b", "value_c"])

    df = pd.DataFrame(data[:MAX_ROWS], columns=columns)

    stats = {
        "shape": list(df.shape),
        "columns": list(df.columns),
        "describe": df.describe().to_dict(),
        "correlation": df.corr().to_dict(),
        "sum": df.sum().to_dict(),
        "environment": APP_ENV,
        "pandasVersion": pd.__version__,
        "numpyVersion": np.__version__,
        "requestId": context.aws_request_id
    }

    logger.info("Processed %d rows with env=%s", len(df), APP_ENV)
    return stats
PYEOF

cd /tmp/advanced-function && zip -r function.zip lambda_function.py
```

---

### Step 4 — Deploy with Layer and Environment Variables

```bash
aws lambda create-function \
  --function-name "${FUNCTION_NAME}" \
  --runtime python3.12 \
  --role "${ROLE_ARN}" \
  --handler lambda_function.handler \
  --zip-file fileb:///tmp/advanced-function/function.zip \
  --timeout 60 \
  --memory-size 512 \
  --layers "${LAYER_ARN}" \
  --environment "Variables={LOG_LEVEL=DEBUG,APP_ENV=dev,MAX_ROWS=50}" \
  --description "Lab 04: demonstrates layers, env vars, concurrency" \
  --tags Project=LambdaLabs,Lab=04 \
  --query '{FunctionName:FunctionName,Layers:Layers,Runtime:Runtime}' \
  --output json

aws lambda wait function-active --function-name "${FUNCTION_NAME}"
echo "Function deployed ✅"
```

---

### Step 5 — Test the function

```bash
cat > /tmp/test-payload.json << 'EOF'
{
  "data": [
    [10, 20, 30],
    [15, 25, 35],
    [20, 30, 40],
    [25, 35, 45],
    [30, 40, 50]
  ],
  "columns": ["revenue", "cost", "profit"]
}
EOF

aws lambda invoke \
  --function-name "${FUNCTION_NAME}" \
  --payload file:///tmp/test-payload.json \
  --cli-binary-format raw-in-base64-out \
  /tmp/analytics-response.json

python3 -c "
import json
with open('/tmp/analytics-response.json') as f:
    r = json.load(f)
print('Shape:', r['shape'])
print('Environment:', r['environment'])
print('Pandas:', r['pandasVersion'])
print('Sum:', r['sum'])
"
```

**Expected output:**
```
Shape: [5, 3]
Environment: dev
Pandas: 2.x.x
Sum: {'revenue': 100, 'cost': 150, 'profit': 200}
```

---

### Step 6 — Update environment variables without redeploying code

```bash
# Update env vars — promotes to prod config
aws lambda update-function-configuration \
  --function-name "${FUNCTION_NAME}" \
  --environment "Variables={LOG_LEVEL=WARNING,APP_ENV=prod,MAX_ROWS=1000}" \
  --query '{FunctionName:FunctionName,Environment:Environment}' \
  --output json

aws lambda wait function-updated --function-name "${FUNCTION_NAME}"

# Invoke again — same code, different behavior
aws lambda invoke \
  --function-name "${FUNCTION_NAME}" \
  --payload file:///tmp/test-payload.json \
  --cli-binary-format raw-in-base64-out \
  /tmp/prod-response.json

python3 -c "
import json
with open('/tmp/prod-response.json') as f:
    r = json.load(f)
print('Environment now:', r['environment'])
print('Max rows config now: 1000 (from env var)')
"
```

---

### Step 7 — Encrypt env vars with a KMS CMK

```bash
# Create a KMS key for Lambda env var encryption
KEY_ARN=$(aws kms create-key \
  --description "Lambda env var encryption key - Lab 04" \
  --tags TagKey=Project,TagValue=LambdaLabs \
  --query 'KeyMetadata.KeyId' --output text)

KEY_ID=$(aws kms describe-key --key-id "${KEY_ARN}" --query 'KeyMetadata.KeyId' --output text)
echo "KMS Key ID: ${KEY_ID}"

# Create an alias for readability
aws kms create-alias \
  --alias-name "alias/lambda-lab04" \
  --target-key-id "${KEY_ID}"

# Grant Lambda role permission to use the key
aws kms create-grant \
  --key-id "${KEY_ID}" \
  --grantee-principal "${ROLE_ARN}" \
  --operations Decrypt GenerateDataKey \
  --name "lambda-lab04-decrypt-grant"

# Update function to use KMS encryption for env vars
aws lambda update-function-configuration \
  --function-name "${FUNCTION_NAME}" \
  --kms-key-arn "${KEY_ARN}" \
  --environment "Variables={LOG_LEVEL=WARNING,APP_ENV=prod,MAX_ROWS=1000,SECRET_KEY=my-super-secret}" \
  --query '{KMSKeyArn:KMSKeyArn}' --output table

echo "Env vars now encrypted with CMK ✅"
```

---

### Step 8 — Set Reserved Concurrency

```bash
# Reserve 20 concurrent executions for this function
# This ALSO limits it — it cannot use more than 20 simultaneously
aws lambda put-function-concurrency \
  --function-name "${FUNCTION_NAME}" \
  --reserved-concurrent-executions 20 \
  --query '{ReservedConcurrentExecutions:ReservedConcurrentExecutions}' \
  --output table

echo "Reserved concurrency set to 20 ✅"

# Verify
aws lambda get-function-concurrency \
  --function-name "${FUNCTION_NAME}" \
  --query 'ReservedConcurrentExecutions' \
  --output text

# To THROTTLE a function to zero (block all invocations — useful for emergency stop):
# aws lambda put-function-concurrency \
#   --function-name "${FUNCTION_NAME}" \
#   --reserved-concurrent-executions 0
```

---

### Step 9 — Set Provisioned Concurrency (eliminates cold starts)

```bash
# Publish a version first — provisioned concurrency works on versions/aliases only
VERSION=$(aws lambda publish-version \
  --function-name "${FUNCTION_NAME}" \
  --description "v1 - prod ready" \
  --query 'Version' --output text)

echo "Published version: ${VERSION}"

# Create a PROD alias pointing to this version
aws lambda create-alias \
  --function-name "${FUNCTION_NAME}" \
  --name PROD \
  --function-version "${VERSION}" \
  --description "Production alias" \
  --query '{Name:Name,FunctionVersion:FunctionVersion}' \
  --output table

# Set provisioned concurrency on the PROD alias (keeps 5 environments pre-warmed)
aws lambda put-provisioned-concurrency-config \
  --function-name "${FUNCTION_NAME}" \
  --qualifier PROD \
  --provisioned-concurrent-executions 5 \
  --query '{Status:Status,AllocatedProvisionedConcurrentExecutions:AllocatedProvisionedConcurrentExecutions}' \
  --output table

echo "Waiting for provisioned concurrency to be ready..."
aws lambda wait function-active-v2 \
  --function-name "${FUNCTION_NAME}:PROD" 2>/dev/null || sleep 30

# Check status
aws lambda get-provisioned-concurrency-config \
  --function-name "${FUNCTION_NAME}" \
  --qualifier PROD \
  --query '{Status:Status,Allocated:AllocatedProvisionedConcurrentExecutions,Requested:RequestedProvisionedConcurrentExecutions}' \
  --output table
```

**Expected output:**
```
-----------------------------------------------
|    GetProvisionedConcurrencyConfig          |
+-----------+---------------------+------------+
| Status    | Allocated           | Requested  |
+-----------+---------------------+------------+
| READY     | 5                   | 5          |
+-----------+---------------------+------------+
```

```bash
# Invoke via PROD alias — guaranteed no cold start
aws lambda invoke \
  --function-name "${FUNCTION_NAME}:PROD" \
  --payload file:///tmp/test-payload.json \
  --cli-binary-format raw-in-base64-out \
  /tmp/prod-alias-response.json && echo "Warm invocation succeeded ✅"
```

---

### Step 10 — List all layers in your account

```bash
aws lambda list-layers \
  --compatible-runtime python3.12 \
  --query 'Layers[*].[LayerName,LatestMatchingVersion.LayerVersionArn,LatestMatchingVersion.Version]' \
  --output table
```

---

## 🧹 Cleanup

```bash
# Remove provisioned concurrency
aws lambda delete-provisioned-concurrency-config \
  --function-name "${FUNCTION_NAME}" --qualifier PROD

# Remove alias
aws lambda delete-alias --function-name "${FUNCTION_NAME}" --name PROD

# Remove reserved concurrency
aws lambda delete-function-concurrency --function-name "${FUNCTION_NAME}"

# Delete function (all versions)
aws lambda delete-function --function-name "${FUNCTION_NAME}"

# Delete the layer version
LAYER_VERSION=$(echo "${LAYER_ARN}" | awk -F: '{print $NF}')
aws lambda delete-layer-version --layer-name "${LAYER_NAME}" --version-number "${LAYER_VERSION}"

# Delete KMS key (schedule for 7-day deletion)
aws kms schedule-key-deletion --key-id "${KEY_ID}" --pending-window-in-days 7
aws kms delete-alias --alias-name "alias/lambda-lab04"

# Delete log group
aws logs delete-log-group --log-group-name "/aws/lambda/${FUNCTION_NAME}" 2>/dev/null

echo "Cleanup complete ✅"
```

---

## ✅ What You Learned

- Layers decouple large dependencies from function code — publish once, use in many functions
- Environment variables are the simplest config mechanism; KMS CMK encryption adds a security layer
- Reserved concurrency is a **dual-purpose control**: it guarantees capacity AND limits blast radius
- Provisioned concurrency eliminates cold starts but incurs cost — apply it to latency-sensitive aliases only
- SnapStart (Java) is free and delivers similar cold-start improvements as provisioned concurrency for JVM runtimes

---

➡️ Next: [Lab 05 — IAM & Security](../05-iam-security/README.md)
