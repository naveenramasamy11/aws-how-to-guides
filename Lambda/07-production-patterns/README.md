# Lab 07 — Production Patterns: Monitoring, Cost, Retries & Error Handling

🔴 **Difficulty:** Advanced | ⏱️ **Time:** ~75 min

---

## 🎯 Goal

Harden a Lambda function for production use: build a CloudWatch dashboard with key metrics, configure alarms for errors and throttles, implement structured JSON logging, add idempotent retry logic, and right-size memory with Power Tuning. Walk away with a production readiness checklist.

---

## 🧠 Concepts

### Key Lambda Metrics to Monitor

| Metric | Namespace | What it signals |
|--------|-----------|-----------------|
| `Errors` | AWS/Lambda | Function-level exceptions (client errors, unhandled exceptions) |
| `Throttles` | AWS/Lambda | Requests rejected due to concurrency limits |
| `Duration` | AWS/Lambda | Execution time — watch p99 for tail latency |
| `ConcurrentExecutions` | AWS/Lambda | Parallel executions — compare to reserved limit |
| `IteratorAge` | AWS/Lambda | SQS/Kinesis/DDB stream lag — time from write to Lambda processing |
| `InitDuration` | AWS/Lambda/logs | Cold start overhead — visible in REPORT log lines |
| `Invocations` | AWS/Lambda | Total calls — use for cost and throughput trending |
| `PostRuntimeExtensionsDuration` | AWS/Lambda | Time spent in extensions after handler returns |

### Structured Logging Pattern

CloudWatch Logs Insights queries work best with JSON-structured logs. A single JSON object per log line enables powerful filtering and aggregation.

```python
# ❌ Unstructured — hard to query
logger.info("Processed order ORD-001 for Alice, amount=99.99, status=OK")

# ✅ Structured JSON — queryable in Logs Insights
logger.info(json.dumps({
    "event": "order_processed",
    "orderId": "ORD-001",
    "customer": "Alice",
    "amount": 99.99,
    "durationMs": 45,
    "status": "success"
}))
```

### Idempotency Pattern

Lambda async triggers retry on failure. If your function writes to DynamoDB on each invocation, duplicate invocations cause duplicate writes unless you implement idempotency.

```
Approach 1 — Idempotency key in DynamoDB (condition expression):
  PutItem with ConditionExpression: attribute_not_exists(orderId)
  → Second invocation with same orderId raises ConditionalCheckFailedException → idempotent ✅

Approach 2 — AWS Lambda Powertools Idempotency decorator:
  @idempotent(persistence_store=DynamoDBPersistenceLayer(table_name="..."))
  def handler(event, context): ...
  → Automatic deduplication with TTL-based expiry ✅
```

### Cost Optimization Levers

| Lever | Typical Saving |
|-------|---------------|
| Right-size memory (Power Tuning) | 20–50% |
| Reduce average duration (optimize code, connections) | Proportional to time saved |
| Use ARM/Graviton2 (`arm64`) | 20% cheaper per GB-s vs x86_64 |
| Avoid provisioned concurrency on low-traffic functions | Remove idle cost |
| Reduce unnecessary invocations (dedup at source) | Proportional to invocation count |

---

## 🏗️ Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        Production Lambda Setup                            │
│                                                                           │
│  ┌─────────────────────────────────────────────────────────────────┐    │
│  │            Lambda Function (arm64, right-sized memory)           │    │
│  │                                                                  │    │
│  │  Structured JSON logging → CloudWatch Logs                       │    │
│  │  Idempotency → DynamoDB condition expressions                    │    │
│  │  Error handling → try/except + fallback + DLQ                   │    │
│  │  X-Ray tracing → distributed trace map                           │    │
│  └───────────────────────────────┬──────────────────────────────────┘    │
│                                  │                                         │
│       ┌──────────────────────────┼──────────────────────────────┐        │
│       │                          │                               │        │
│  ┌────▼──────┐         ┌─────────▼──────────┐        ┌─────────▼──────┐ │
│  │CloudWatch │         │  CloudWatch Alarms   │        │  X-Ray         │ │
│  │Dashboard  │         │  Error rate > 1%     │        │  Service Map   │ │
│  │- Errors   │         │  Throttles > 0       │        │  & Trace View  │ │
│  │- Duration │         │  Duration p99 > 5s   │        └────────────────┘ │
│  │- Throttles│         │  → SNS → PagerDuty   │                          │
│  └───────────┘         └────────────────────┘                           │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## 🔢 PoC Steps

### Step 1 — Setup

```bash
export AWS_REGION="us-east-1"
export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export FUNCTION_NAME="production-lambda-demo"
export DASHBOARD_NAME="lambda-production-dashboard"
export ALARM_SNS_TOPIC="lambda-production-alerts"

echo "Account: ${AWS_ACCOUNT_ID}"
```

---

### Step 2 — Create the IAM execution role

```bash
cat > /tmp/prod-trust.json << 'EOF'
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
  --role-name prod-lambda-role \
  --assume-role-policy-document file:///tmp/prod-trust.json \
  --tags Key=Project,Value=LambdaLabs Key=Lab,Value=07 \
  --query 'Role.Arn' --output text

ROLE_ARN="arn:aws:iam::${AWS_ACCOUNT_ID}:role/prod-lambda-role"

aws iam attach-role-policy --role-name prod-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Add X-Ray tracing permission
aws iam attach-role-policy --role-name prod-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess

sleep 10
echo "IAM role ready: ${ROLE_ARN}"
```

---

### Step 3 — Write a production-hardened function

```bash
mkdir -p /tmp/prod-function

cat > /tmp/prod-function/lambda_function.py << 'PYEOF'
"""
Production Lambda function demonstrating:
- Structured JSON logging
- Idempotency via DynamoDB conditional writes
- Retry logic with exponential backoff
- Input validation
- Error classification (retriable vs. permanent)
- X-Ray tracing via environment variable AWS_XRAY_SDK_ENABLED
"""
import os
import json
import time
import logging
import uuid
from datetime import datetime, timezone
from typing import Any

logger = logging.getLogger()
logger.setLevel(os.environ.get("LOG_LEVEL", "INFO"))

# ── Structured logging helper ─────────────────────────────────────────────
def log(level: str, event: str, **kwargs: Any) -> None:
    record = {
        "timestamp": datetime.now(timezone.utc).isoformat(),
        "level": level,
        "event": event,
        "service": "production-lambda-demo",
        **kwargs
    }
    getattr(logger, level.lower())(json.dumps(record))


# ── Retry with exponential backoff ───────────────────────────────────────
class RetryableError(Exception):
    """Transient error — safe to retry."""

class PermanentError(Exception):
    """Permanent error — retrying won't help."""


def with_retry(func, max_attempts: int = 3, base_delay: float = 0.5):
    """Execute func with exponential backoff on RetryableError."""
    for attempt in range(1, max_attempts + 1):
        try:
            return func()
        except RetryableError as e:
            if attempt == max_attempts:
                log("error", "max_retries_exceeded", error=str(e), attempts=attempt)
                raise
            delay = base_delay * (2 ** (attempt - 1))
            log("warning", "retrying", error=str(e), attempt=attempt, next_delay_s=delay)
            time.sleep(delay)
        except PermanentError:
            raise


# ── Input validation ─────────────────────────────────────────────────────
def validate_input(event: dict) -> dict:
    required = ["orderId", "amount", "customer"]
    missing = [f for f in required if not event.get(f)]
    if missing:
        raise PermanentError(f"Missing required fields: {missing}")

    try:
        amount = float(event["amount"])
    except (ValueError, TypeError):
        raise PermanentError(f"Invalid amount: {event['amount']!r}")

    if amount <= 0:
        raise PermanentError(f"Amount must be positive, got {amount}")

    return {
        "orderId": str(event["orderId"]).strip(),
        "amount": amount,
        "customer": str(event["customer"]).strip()
    }


# ── Simulated business logic ─────────────────────────────────────────────
_call_count = 0  # Simulates transient upstream failure for PoC demo

def process_order(order: dict) -> dict:
    global _call_count
    _call_count += 1

    # Simulate a transient error on first call within each cold start (PoC only)
    if _call_count == 1 and os.environ.get("SIMULATE_TRANSIENT_ERROR") == "true":
        raise RetryableError("Upstream service temporarily unavailable (simulated)")

    return {
        "orderId": order["orderId"],
        "customer": order["customer"],
        "amount": order["amount"],
        "tax": round(order["amount"] * 0.1, 2),
        "total": round(order["amount"] * 1.1, 2),
        "processedAt": datetime.now(timezone.utc).isoformat(),
        "idempotencyKey": str(uuid.uuid4())
    }


# ── Main handler ─────────────────────────────────────────────────────────
def handler(event, context):
    start_time = time.perf_counter()
    request_id = context.aws_request_id

    log("info", "invocation_start",
        requestId=request_id,
        remainingMs=context.get_remaining_time_in_millis(),
        functionVersion=context.function_version)

    try:
        # Validate
        order = validate_input(event)
        log("info", "input_validated", orderId=order["orderId"])

        # Process (with retry)
        result = with_retry(lambda: process_order(order))
        log("info", "order_processed",
            orderId=result["orderId"],
            total=result["total"])

        duration_ms = round((time.perf_counter() - start_time) * 1000, 2)
        log("info", "invocation_success",
            requestId=request_id,
            durationMs=duration_ms,
            orderId=result["orderId"])

        return {"statusCode": 200, "result": result}

    except PermanentError as e:
        duration_ms = round((time.perf_counter() - start_time) * 1000, 2)
        log("error", "permanent_error",
            requestId=request_id,
            error=str(e),
            durationMs=duration_ms)
        # Return 400 for API Gateway integration; do NOT raise (no retry needed)
        return {"statusCode": 400, "error": str(e)}

    except RetryableError as e:
        duration_ms = round((time.perf_counter() - start_time) * 1000, 2)
        log("error", "retryable_error_exhausted",
            requestId=request_id,
            error=str(e),
            durationMs=duration_ms)
        # Raise to trigger Lambda's built-in async retry + DLQ
        raise

    except Exception as e:
        duration_ms = round((time.perf_counter() - start_time) * 1000, 2)
        log("error", "unexpected_error",
            requestId=request_id,
            error=str(e),
            errorType=type(e).__name__,
            durationMs=duration_ms)
        raise
PYEOF

cd /tmp/prod-function && zip -r function.zip lambda_function.py
```

---

### Step 4 — Deploy on arm64 (Graviton2) for 20% cost savings

```bash
aws lambda create-function \
  --function-name "${FUNCTION_NAME}" \
  --runtime python3.12 \
  --architectures arm64 \
  --role "${ROLE_ARN}" \
  --handler lambda_function.handler \
  --zip-file fileb:///tmp/prod-function/function.zip \
  --timeout 30 \
  --memory-size 256 \
  --tracing-config Mode=Active \
  --environment "Variables={LOG_LEVEL=INFO,SIMULATE_TRANSIENT_ERROR=false}" \
  --description "Lab 07: production-hardened Lambda on arm64 with X-Ray" \
  --tags Project=LambdaLabs,Lab=07 \
  --query '{FunctionName:FunctionName,Architectures:Architectures,TracingConfig:TracingConfig}' \
  --output json

aws lambda wait function-active --function-name "${FUNCTION_NAME}"
echo "arm64 function deployed ✅"
```

---

### Step 5 — Run load tests and capture metrics

```bash
# Run 20 invocations to generate meaningful CloudWatch metrics
echo "Running 20 test invocations..."

for i in $(seq 1 20); do
  PAYLOAD="{\"orderId\": \"ORD-$(printf '%04d' $i)\", \"amount\": $(echo "scale=2; $RANDOM / 100" | bc), \"customer\": \"TestCustomer${i}\"}"
  aws lambda invoke \
    --function-name "${FUNCTION_NAME}" \
    --payload "${PAYLOAD}" \
    --cli-binary-format raw-in-base64-out \
    /tmp/response-${i}.json \
    --query 'StatusCode' --output text &
done

wait
echo "20 invocations complete ✅"

# Inject 3 error invocations (missing required fields)
for i in 21 22 23; do
  aws lambda invoke \
    --function-name "${FUNCTION_NAME}" \
    --payload '{"orderId": "BAD-001"}' \
    --cli-binary-format raw-in-base64-out \
    /tmp/error-response-${i}.json \
    --query 'StatusCode' --output text &
done

wait
echo "Error invocations done ✅"
echo "Waiting 60s for CloudWatch to aggregate metrics..."
sleep 60
```

---

### Step 6 — Query logs with CloudWatch Logs Insights

```bash
# Get the log group start time (5 minutes ago)
START_TIME=$(date -d '5 minutes ago' +%s 2>/dev/null || date -v-5M +%s)
END_TIME=$(date +%s)

# Query structured logs for successful orders
aws logs start-query \
  --log-group-name "/aws/lambda/${FUNCTION_NAME}" \
  --start-time "${START_TIME}" \
  --end-time "${END_TIME}" \
  --query-string '
    fields @timestamp, event, orderId, durationMs, total
    | filter event = "order_processed"
    | sort durationMs desc
    | limit 10
  ' \
  --query 'queryId' --output text > /tmp/query-id.txt

QUERY_ID=$(cat /tmp/query-id.txt)
echo "Query ID: ${QUERY_ID}"

sleep 10

# Get results
aws logs get-query-results \
  --query-id "${QUERY_ID}" \
  --query '{Status:status,Results:results}' \
  --output json | python3 -m json.tool | head -50

# Query for errors by type
aws logs start-query \
  --log-group-name "/aws/lambda/${FUNCTION_NAME}" \
  --start-time "${START_TIME}" \
  --end-time "${END_TIME}" \
  --query-string '
    fields @timestamp, event, error, requestId
    | filter level = "error"
    | stats count(*) as errorCount by event
  ' \
  --query 'queryId' --output text > /tmp/error-query-id.txt

sleep 10
aws logs get-query-results \
  --query-id "$(cat /tmp/error-query-id.txt)" \
  --output json | python3 -m json.tool
```

---

### Step 7 — Create CloudWatch Dashboard

```bash
DASHBOARD_BODY=$(cat << DASH
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 8, "height": 6,
      "properties": {
        "title": "Invocations & Errors",
        "metrics": [
          ["AWS/Lambda", "Invocations", "FunctionName", "${FUNCTION_NAME}", {"color": "#1f77b4", "label": "Invocations"}],
          ["AWS/Lambda", "Errors", "FunctionName", "${FUNCTION_NAME}", {"color": "#d62728", "label": "Errors"}],
          ["AWS/Lambda", "Throttles", "FunctionName", "${FUNCTION_NAME}", {"color": "#ff7f0e", "label": "Throttles"}]
        ],
        "stat": "Sum",
        "period": 60,
        "view": "timeSeries",
        "region": "${AWS_REGION}"
      }
    },
    {
      "type": "metric",
      "x": 8, "y": 0, "width": 8, "height": 6,
      "properties": {
        "title": "Duration (p50 / p95 / p99)",
        "metrics": [
          ["AWS/Lambda", "Duration", "FunctionName", "${FUNCTION_NAME}", {"stat": "p50", "label": "p50"}],
          ["AWS/Lambda", "Duration", "FunctionName", "${FUNCTION_NAME}", {"stat": "p95", "label": "p95"}],
          ["AWS/Lambda", "Duration", "FunctionName", "${FUNCTION_NAME}", {"stat": "p99", "label": "p99"}]
        ],
        "period": 60,
        "view": "timeSeries",
        "region": "${AWS_REGION}"
      }
    },
    {
      "type": "metric",
      "x": 16, "y": 0, "width": 8, "height": 6,
      "properties": {
        "title": "Concurrent Executions",
        "metrics": [
          ["AWS/Lambda", "ConcurrentExecutions", "FunctionName", "${FUNCTION_NAME}", {"stat": "Maximum", "label": "Max Concurrency"}]
        ],
        "period": 60,
        "view": "timeSeries",
        "region": "${AWS_REGION}"
      }
    },
    {
      "type": "log",
      "x": 0, "y": 6, "width": 24, "height": 6,
      "properties": {
        "title": "Recent Errors",
        "query": "SOURCE '/aws/lambda/${FUNCTION_NAME}' | fields @timestamp, event, error, requestId | filter level = 'error' | sort @timestamp desc | limit 20",
        "region": "${AWS_REGION}",
        "view": "table"
      }
    }
  ]
}
DASH
)

aws cloudwatch put-dashboard \
  --dashboard-name "${DASHBOARD_NAME}" \
  --dashboard-body "${DASHBOARD_BODY}" \
  --query 'DashboardValidationMessages' \
  --output text

echo "Dashboard created: https://console.aws.amazon.com/cloudwatch/home?region=${AWS_REGION}#dashboards:name=${DASHBOARD_NAME}"
```

---

### Step 8 — Create CloudWatch Alarms

```bash
# Create SNS topic for alarms
ALARM_TOPIC_ARN=$(aws sns create-topic \
  --name "${ALARM_SNS_TOPIC}" \
  --tags Key=Project,Value=LambdaLabs \
  --query 'TopicArn' --output text)

echo "Alarm SNS topic: ${ALARM_TOPIC_ARN}"

# Alarm 1: Error rate > 5% over 5-minute window
aws cloudwatch put-metric-alarm \
  --alarm-name "${FUNCTION_NAME}-HighErrorRate" \
  --alarm-description "Lambda error rate exceeds 5% — investigate immediately" \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value="${FUNCTION_NAME}" \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 5 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions "${ALARM_TOPIC_ARN}" \
  --ok-actions "${ALARM_TOPIC_ARN}" \
  --tags Key=Project,Value=LambdaLabs

# Alarm 2: Any throttle events
aws cloudwatch put-metric-alarm \
  --alarm-name "${FUNCTION_NAME}-Throttles" \
  --alarm-description "Lambda throttles detected — increase reserved concurrency or request limit increase" \
  --namespace AWS/Lambda \
  --metric-name Throttles \
  --dimensions Name=FunctionName,Value="${FUNCTION_NAME}" \
  --statistic Sum \
  --period 60 \
  --evaluation-periods 1 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions "${ALARM_TOPIC_ARN}"

# Alarm 3: p99 duration > 10 seconds (80% of timeout)
aws cloudwatch put-metric-alarm \
  --alarm-name "${FUNCTION_NAME}-HighDuration-p99" \
  --alarm-description "Lambda p99 duration approaching timeout limit — investigate slow code paths" \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value="${FUNCTION_NAME}" \
  --extended-statistic p99 \
  --period 300 \
  --evaluation-periods 2 \
  --threshold 10000 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --treat-missing-data notBreaching \
  --alarm-actions "${ALARM_TOPIC_ARN}"

echo "Alarms created ✅"
aws cloudwatch describe-alarms \
  --alarm-name-prefix "${FUNCTION_NAME}" \
  --query 'MetricAlarms[*].[AlarmName,StateValue,Threshold]' \
  --output table
```

---

### Step 9 — View X-Ray trace

```bash
# Get traces from the last 5 minutes
END_TIME=$(date +%s)
START_TIME=$((END_TIME - 300))

TRACE_SUMMARIES=$(aws xray get-trace-summaries \
  --start-time "${START_TIME}" \
  --end-time "${END_TIME}" \
  --query 'TraceSummaries[*].[Id,Duration,HasError]' \
  --output table 2>/dev/null)

echo "Recent X-Ray traces:"
echo "${TRACE_SUMMARIES}"

# Get service graph
aws xray get-service-graph \
  --start-time "${START_TIME}" \
  --end-time "${END_TIME}" \
  --query 'Services[*].[Name,Type,State]' \
  --output table 2>/dev/null || echo "Service graph available in X-Ray console"
```

---

### Step 10 — Production Readiness Checklist

```bash
echo "=== Production Readiness Checklist for ${FUNCTION_NAME} ==="
echo ""

# Check X-Ray tracing
TRACING=$(aws lambda get-function-configuration \
  --function-name "${FUNCTION_NAME}" \
  --query 'TracingConfig.Mode' --output text)
echo "[$([ "$TRACING" = "Active" ] && echo "✅" || echo "❌")] X-Ray tracing: ${TRACING}"

# Check architecture
ARCH=$(aws lambda get-function-configuration \
  --function-name "${FUNCTION_NAME}" \
  --query 'Architectures[0]' --output text)
echo "[$([ "$ARCH" = "arm64" ] && echo "✅" || echo "⚠️")] Architecture: ${ARCH} (arm64 = 20% cheaper)"

# Check timeout (should not be default 3s for anything complex)
TIMEOUT=$(aws lambda get-function-configuration \
  --function-name "${FUNCTION_NAME}" \
  --query 'Timeout' --output text)
echo "[$([ "$TIMEOUT" -gt 3 ] && echo "✅" || echo "⚠️")] Timeout: ${TIMEOUT}s (default 3s is too low for most workloads)"

# Check reserved concurrency
CONCURRENCY=$(aws lambda get-function-concurrency \
  --function-name "${FUNCTION_NAME}" \
  --query 'ReservedConcurrentExecutions' --output text 2>/dev/null || echo "None")
echo "[$([ "$CONCURRENCY" != "None" ] && echo "✅" || echo "⚠️")] Reserved concurrency: ${CONCURRENCY}"

# Check DLQ
DLQ_CONFIG=$(aws lambda get-function-configuration \
  --function-name "${FUNCTION_NAME}" \
  --query 'DeadLetterConfig.TargetArn' --output text 2>/dev/null || echo "None")
echo "[$([ "$DLQ_CONFIG" != "None" ] && [ -n "$DLQ_CONFIG" ] && echo "✅" || echo "⚠️")] DLQ configured: ${DLQ_CONFIG}"

# Check alarms
ALARM_COUNT=$(aws cloudwatch describe-alarms \
  --alarm-name-prefix "${FUNCTION_NAME}" \
  --query 'length(MetricAlarms)' --output text)
echo "[$([ "$ALARM_COUNT" -gt 0 ] && echo "✅" || echo "❌")] CloudWatch alarms: ${ALARM_COUNT}"

echo ""
echo "Items marked ⚠️ should be reviewed before production deployment."
```

---

## 🧹 Cleanup

```bash
# Delete Lambda function
aws lambda delete-function --function-name "${FUNCTION_NAME}"

# Delete CloudWatch dashboard
aws cloudwatch delete-dashboards --dashboard-names "${DASHBOARD_NAME}"

# Delete alarms
aws cloudwatch delete-alarms \
  --alarm-names \
    "${FUNCTION_NAME}-HighErrorRate" \
    "${FUNCTION_NAME}-Throttles" \
    "${FUNCTION_NAME}-HighDuration-p99"

# Delete SNS topics
ALARM_TOPIC_ARN=$(aws sns list-topics \
  --query "Topics[?contains(TopicArn,'${ALARM_SNS_TOPIC}')].TopicArn" \
  --output text)
[ -n "${ALARM_TOPIC_ARN}" ] && aws sns delete-topic --topic-arn "${ALARM_TOPIC_ARN}"

# Delete IAM role
aws iam detach-role-policy --role-name prod-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam detach-role-policy --role-name prod-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AWSXRayDaemonWriteAccess
aws iam delete-role --role-name prod-lambda-role

# Delete log group
aws logs delete-log-group --log-group-name "/aws/lambda/${FUNCTION_NAME}" 2>/dev/null

# Clean temp files
rm -rf /tmp/prod-function /tmp/response-*.json /tmp/error-response-*.json /tmp/query-id.txt /tmp/error-query-id.txt

echo "Cleanup complete ✅"
```

---

## 📋 Production Lambda Checklist (Summary)

| Category | Item | Implementation |
|----------|------|---------------|
| **Observability** | Structured JSON logging | `json.dumps({...})` in every log call |
| **Observability** | X-Ray active tracing | `--tracing-config Mode=Active` |
| **Observability** | CloudWatch dashboard | Errors, Duration p99, Concurrency |
| **Observability** | Alarms with SNS action | Error rate, throttles, p99 duration |
| **Reliability** | Dead Letter Queue | `--dead-letter-config TargetArn=...` |
| **Reliability** | Idempotent handler | Condition expressions or Powertools |
| **Reliability** | Retry with backoff | Exponential backoff on transient errors |
| **Reliability** | Reserved concurrency | Protects downstream + guarantees capacity |
| **Security** | Least-privilege role | Named resource ARNs, no `*` in Actions |
| **Security** | VPC placement | Private subnet for DB/internal APIs |
| **Security** | KMS env var encryption | `--kms-key-arn` on sensitive config |
| **Cost** | arm64 architecture | `--architectures arm64` |
| **Cost** | Right-sized memory | Power Tuning tool |
| **Cost** | Provisioned concurrency scoped to aliases | Only on latency-sensitive paths |
| **Deployment** | Function versions + aliases | PROD alias → immutable version |
| **Deployment** | Canary/linear deployments | AWS CodeDeploy Lambda deployment groups |

---

## ✅ What You Learned

- Structured JSON logging unlocks powerful CloudWatch Logs Insights queries — implement it from day one
- Error classification (retriable vs. permanent) determines whether to raise (trigger retry + DLQ) or return a 4xx
- X-Ray active tracing provides end-to-end distributed traces at ~5% sampling by default with no code changes
- arm64/Graviton2 delivers a 20% cost reduction with zero code changes for most Python and Node workloads
- Production readiness is a checklist, not a feeling — automate the check in your CI/CD pipeline

---

➡️ Back to [Lambda Module Overview](../README.md)
