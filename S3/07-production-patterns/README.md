# Lab 07 — Production Patterns 🔴

## Goal

Harden your S3 deployment for production: enable server access logging, configure CloudWatch metrics and alarms, implement cost optimisation through S3 Storage Lens, set up error handling / retry patterns, and build a bucket health-check script.

---

## Concepts

### CloudWatch S3 Metrics

| Metric | Namespace | Dimensions | Use |
|--------|-----------|------------|-----|
| `BucketSizeBytes` | `AWS/S3` | `BucketName`, `StorageType` | Storage growth tracking |
| `NumberOfObjects` | `AWS/S3` | `BucketName`, `StorageType` | Object count monitoring |
| `AllRequests` | `AWS/S3` (request metrics) | `BucketName`, `FilterId` | Request volume & latency |
| `4xxErrors` | `AWS/S3` (request metrics) | `BucketName`, `FilterId` | Auth/policy issues |
| `5xxErrors` | `AWS/S3` (request metrics) | `BucketName`, `FilterId` | S3 service errors |
| `FirstByteLatency` | `AWS/S3` (request metrics) | `BucketName`, `FilterId` | Time-to-first-byte |
| `TotalRequestLatency` | `AWS/S3` (request metrics) | `BucketName`, `FilterId` | End-to-end latency |

**Note:** `BucketSizeBytes` and `NumberOfObjects` are updated daily. Request metrics require a bucket-level metrics configuration to be enabled.

### Access Logging vs CloudTrail

| Feature | S3 Server Access Logs | CloudTrail S3 Data Events |
|---------|----------------------|--------------------------|
| Cost | Free (storage only) | $0.10/100K events |
| Latency | Best-effort delivery | Near real-time |
| Detail | HTTP-level request log | API-level audit trail |
| Query | S3 Select / Athena | CloudTrail Lake / Athena |
| Use | Traffic analysis | Security / compliance audit |

### Retry Pattern — Exponential Backoff

For S3 operations that can fail with `503 SlowDown` or transient 5xx errors, use exponential backoff with jitter:

```
Attempt 1: wait 0 ms
Attempt 2: wait 100 ms + jitter
Attempt 3: wait 200 ms + jitter
Attempt 4: wait 400 ms + jitter
Attempt 5: wait 800 ms + jitter
```

The AWS SDK for Python (boto3) does this automatically — configure `max_attempts` and `mode="adaptive"`.

### Cost Optimisation Quick Wins

| Action | Typical Savings |
|--------|----------------|
| Enable lifecycle transitions | 30–90% on old data |
| Enable Intelligent-Tiering | 10–40% on variable-access data |
| Delete incomplete multipart uploads | Varies (often forgotten) |
| Enable S3 Storage Lens | Identifies inefficiencies |
| Use S3 Batch Operations for bulk operations | Reduces request costs |

---

## Architecture Diagram

```
┌──────────────────────────────────────────────────────────────┐
│  Production S3 Bucket                                        │
│                                                              │
│  ┌─────────────────┐  ┌──────────────────┐                   │
│  │  Server Access  │  │  Request Metrics │                   │
│  │  Logs ──────────┼──►  CloudWatch      │                   │
│  └─────────────────┘  │  Namespace:      │                   │
│                        │  AWS/S3          │                   │
│  ┌─────────────────┐  └──────┬───────────┘                   │
│  │  CloudTrail     │         │                               │
│  │  Data Events    │         ▼                               │
│  │  (API audit)    │  ┌──────────────────┐                   │
│  └─────────────────┘  │  CloudWatch      │                   │
│                        │  Alarms          │                   │
│  ┌─────────────────┐  │  4xxErrors > 100 │                   │
│  │  S3 Storage     │  │  5xxErrors > 10  │                   │
│  │  Lens (org-     │  │  BucketSize >    │                   │
│  │  wide insights) │  │  50 GB           │                   │
│  └─────────────────┘  └──────────────────┘                   │
└──────────────────────────────────────────────────────────────┘
```

---

## PoC Steps

### Step 1 — Set variables

```bash
export AWS_REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-howto-${ACCOUNT_ID}-${AWS_REGION}"
export LOGS_BUCKET="${BUCKET_NAME}-access-logs"
echo "Main bucket: $BUCKET_NAME | Logs bucket: $LOGS_BUCKET"
```

---

### Step 2 — Create a dedicated logging bucket and enable server access logging

```bash
# Create logging bucket
aws s3api create-bucket \
  --bucket "$LOGS_BUCKET" \
  --region "$AWS_REGION" \
  --tags Key=Project,Value=S3HowTo Key=Purpose,Value=AccessLogs

# Set bucket ownership controls (required for log delivery)
aws s3api put-bucket-ownership-controls \
  --bucket "$LOGS_BUCKET" \
  --ownership-controls 'Rules=[{ObjectOwnership=BucketOwnerPreferred}]'

# Grant S3 log delivery group write access to the logging bucket
aws s3api put-bucket-acl \
  --bucket "$LOGS_BUCKET" \
  --grant-write 'URI="http://acs.amazonaws.com/groups/s3/LogDelivery"' \
  --grant-read-acp 'URI="http://acs.amazonaws.com/groups/s3/LogDelivery"'

# Enable server access logging on the main bucket
aws s3api put-bucket-logging \
  --bucket "$BUCKET_NAME" \
  --bucket-logging-status "{
    \"LoggingEnabled\": {
      \"TargetBucket\": \"${LOGS_BUCKET}\",
      \"TargetPrefix\": \"s3-access-logs/${BUCKET_NAME}/\"
    }
  }"

# Verify
aws s3api get-bucket-logging \
  --bucket "$BUCKET_NAME" \
  --output table
```

**Expected output:**
```
-----------------------------------------------------------
|                    GetBucketLogging                     |
+------------------------+--------------------------------+
|  TargetBucket          |  s3-howto-...-access-logs      |
|  TargetPrefix          |  s3-access-logs/s3-howto-.../  |
+------------------------+--------------------------------+
```

---

### Step 3 — Enable CloudWatch request metrics

```bash
# Enable request metrics for all objects in the bucket
aws s3api put-bucket-metrics-configuration \
  --bucket "$BUCKET_NAME" \
  --id "AllObjects" \
  --metrics-configuration '{"Id":"AllObjects","Filter":{"Prefix":""}}'

# Verify
aws s3api get-bucket-metrics-configuration \
  --bucket "$BUCKET_NAME" \
  --id "AllObjects" \
  --output table
```

---

### Step 4 — Create CloudWatch alarms for S3 error rates

```bash
# First, retrieve or set up an SNS topic ARN for alarms
# (Reuse or create a new one)
ALARM_SNS_ARN=$(aws sns create-topic \
  --name "S3HowToAlarms" \
  --tags Key=Project,Value=S3HowTo \
  --query TopicArn \
  --output text)
echo "Alarm SNS ARN: $ALARM_SNS_ARN"

# Alarm: 4xx error rate > 100 in 5 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name "S3HowTo-4xxErrors-High" \
  --alarm-description "S3 4xx errors exceed 100 in 5 minutes — check bucket policy / IAM" \
  --namespace "AWS/S3" \
  --metric-name "4xxErrors" \
  --dimensions \
    Name=BucketName,Value=$BUCKET_NAME \
    Name=FilterId,Value=AllObjects \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 100 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions "$ALARM_SNS_ARN" \
  --ok-actions "$ALARM_SNS_ARN" \
  --tags Key=Project,Value=S3HowTo

# Alarm: 5xx error rate > 10 in 5 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name "S3HowTo-5xxErrors-High" \
  --alarm-description "S3 5xx errors exceed 10 in 5 minutes — potential S3 service issue" \
  --namespace "AWS/S3" \
  --metric-name "5xxErrors" \
  --dimensions \
    Name=BucketName,Value=$BUCKET_NAME \
    Name=FilterId,Value=AllObjects \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 10 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions "$ALARM_SNS_ARN" \
  --tags Key=Project,Value=S3HowTo

# List all alarms for this project
aws cloudwatch describe-alarms \
  --alarm-name-prefix "S3HowTo" \
  --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Metric:MetricName}' \
  --output table
```

---

### Step 5 — Production-hardened Python client with retries

```python
#!/usr/bin/env python3
# s3_production_client.py
"""
Production-grade S3 client with:
- Adaptive retry (exponential backoff + jitter)
- Checksum validation on upload
- Structured error handling
- Per-operation logging
"""
import boto3
import hashlib
import base64
import logging
import os
from botocore.config import Config
from botocore.exceptions import ClientError

# Configure structured logging
logging.basicConfig(
    level=logging.INFO,
    format="%(asctime)s %(levelname)s %(name)s %(message)s",
)
log = logging.getLogger("s3_prod_client")

# Production boto3 config: adaptive retry mode, 5 max attempts
PROD_CONFIG = Config(
    retries={"max_attempts": 5, "mode": "adaptive"},
    max_pool_connections=50,
)

s3 = boto3.client("s3", region_name="us-east-1", config=PROD_CONFIG)
BUCKET = os.environ["BUCKET_NAME"]


def upload_with_checksum(local_path: str, s3_key: str) -> dict:
    """Upload a file to S3 with MD5 checksum verification."""
    with open(local_path, "rb") as f:
        data = f.read()

    md5 = base64.b64encode(hashlib.md5(data).digest()).decode()
    log.info("Uploading %s to s3://%s/%s (MD5: %s)", local_path, BUCKET, s3_key, md5)

    try:
        resp = s3.put_object(
            Bucket=BUCKET,
            Key=s3_key,
            Body=data,
            ContentMD5=md5,
            Tagging="Source=ProdClient&Verified=true",
        )
        log.info("Upload succeeded. ETag: %s", resp["ETag"])
        return {"status": "success", "etag": resp["ETag"], "key": s3_key}
    except ClientError as e:
        code = e.response["Error"]["Code"]
        msg = e.response["Error"]["Message"]
        log.error("Upload failed: %s — %s", code, msg)
        raise


def safe_get_object(s3_key: str) -> bytes:
    """Download an object with graceful 404 handling."""
    try:
        resp = s3.get_object(Bucket=BUCKET, Key=s3_key)
        data = resp["Body"].read()
        log.info("Downloaded %s (%d bytes)", s3_key, len(data))
        return data
    except ClientError as e:
        if e.response["Error"]["Code"] == "NoSuchKey":
            log.warning("Object not found: %s", s3_key)
            return None
        raise


def get_bucket_stats() -> dict:
    """Return total object count and size for the bucket."""
    paginator = s3.get_paginator("list_objects_v2")
    count, total_bytes = 0, 0
    for page in paginator.paginate(Bucket=BUCKET):
        for obj in page.get("Contents", []):
            count += 1
            total_bytes += obj["Size"]
    stats = {"object_count": count, "total_bytes": total_bytes,
             "total_mb": round(total_bytes / (1024 ** 2), 2)}
    log.info("Bucket stats: %s", stats)
    return stats


def abort_stalled_multipart_uploads(days_old: int = 7) -> int:
    """Cancel any multipart uploads older than `days_old` days."""
    from datetime import datetime, timezone, timedelta
    paginator = s3.get_paginator("list_multipart_uploads")
    cutoff = datetime.now(timezone.utc) - timedelta(days=days_old)
    aborted = 0
    try:
        for page in paginator.paginate(Bucket=BUCKET):
            for upload in page.get("Uploads", []):
                if upload["Initiated"] < cutoff:
                    s3.abort_multipart_upload(
                        Bucket=BUCKET,
                        Key=upload["Key"],
                        UploadId=upload["UploadId"],
                    )
                    log.info("Aborted stalled upload: %s (id=%s)", upload["Key"], upload["UploadId"])
                    aborted += 1
    except ClientError as e:
        if e.response["Error"]["Code"] != "NoSuchUpload":
            raise
    log.info("Aborted %d stalled multipart uploads", aborted)
    return aborted


if __name__ == "__main__":
    # Demo: upload, retrieve, stats, cleanup check
    import tempfile

    with tempfile.NamedTemporaryFile(mode="w", suffix=".txt", delete=False) as f:
        f.write("Production test file — checksum validated upload\n")
        tmp = f.name

    result = upload_with_checksum(tmp, "production/test-checksum.txt")
    print("Upload result:", result)

    content = safe_get_object("production/test-checksum.txt")
    print("Retrieved:", content.decode())

    missing = safe_get_object("production/does-not-exist.txt")
    print("Missing file result:", missing)

    stats = get_bucket_stats()
    print("Bucket stats:", stats)

    aborted = abort_stalled_multipart_uploads(days_old=1)
    print(f"Cleaned up {aborted} stalled uploads")
```

```bash
python3 s3_production_client.py
```

**Expected output:**
```
Upload result: {'status': 'success', 'etag': '"abc123..."', 'key': 'production/test-checksum.txt'}
Retrieved: Production test file — checksum validated upload
Missing file result: None
Bucket stats: {'object_count': 3, 'total_bytes': 204, 'total_mb': 0.0}
Cleaned up 0 stalled uploads
```

---

### Step 6 — Bucket health-check script

```python
#!/usr/bin/env python3
# s3_health_check.py
"""
Run a comprehensive health check on an S3 bucket and print a report.
Checks: versioning, encryption, public access, logging, lifecycle rules, bucket policy.
"""
import boto3
import json
import os

s3 = boto3.client("s3", region_name="us-east-1")
BUCKET = os.environ["BUCKET_NAME"]

checks = {}

# 1. Versioning
try:
    v = s3.get_bucket_versioning(Bucket=BUCKET)
    checks["versioning"] = {"status": v.get("Status", "Disabled"), "ok": v.get("Status") == "Enabled"}
except Exception as e:
    checks["versioning"] = {"error": str(e), "ok": False}

# 2. Encryption
try:
    enc = s3.get_bucket_encryption(Bucket=BUCKET)
    algo = enc["ServerSideEncryptionConfiguration"]["Rules"][0]["ApplyServerSideEncryptionByDefault"]["SSEAlgorithm"]
    checks["encryption"] = {"algorithm": algo, "ok": True}
except Exception:
    checks["encryption"] = {"algorithm": "none", "ok": False}

# 3. Block Public Access
try:
    bpa = s3.get_public_access_block(Bucket=BUCKET)["PublicAccessBlockConfiguration"]
    all_blocked = all(bpa.values())
    checks["block_public_access"] = {"config": bpa, "ok": all_blocked}
except Exception as e:
    checks["block_public_access"] = {"error": str(e), "ok": False}

# 4. Logging
try:
    log_cfg = s3.get_bucket_logging(Bucket=BUCKET).get("LoggingEnabled")
    checks["server_access_logging"] = {
        "enabled": bool(log_cfg),
        "target": log_cfg.get("TargetBucket") if log_cfg else None,
        "ok": bool(log_cfg),
    }
except Exception as e:
    checks["server_access_logging"] = {"error": str(e), "ok": False}

# 5. Lifecycle rules
try:
    lc = s3.get_bucket_lifecycle_configuration(Bucket=BUCKET)
    rules = [r["ID"] for r in lc.get("Rules", []) if r["Status"] == "Enabled"]
    checks["lifecycle"] = {"enabled_rules": rules, "ok": len(rules) > 0}
except s3.exceptions.from_code("NoSuchLifecycleConfiguration"):
    checks["lifecycle"] = {"enabled_rules": [], "ok": False}
except Exception as e:
    checks["lifecycle"] = {"enabled_rules": [], "ok": False}

# 6. Bucket policy (check for HTTPS-only deny)
try:
    policy = json.loads(s3.get_bucket_policy(Bucket=BUCKET)["Policy"])
    has_https_deny = any(
        s.get("Effect") == "Deny" and
        s.get("Condition", {}).get("Bool", {}).get("aws:SecureTransport") == "false"
        for s in policy.get("Statement", [])
    )
    checks["bucket_policy"] = {"has_https_deny": has_https_deny, "ok": has_https_deny}
except Exception:
    checks["bucket_policy"] = {"has_https_deny": False, "ok": False}

# Print report
print(f"\nS3 Bucket Health Check: {BUCKET}")
print("=" * 60)
for check, result in checks.items():
    icon = "OK" if result.get("ok") else "FAIL"
    detail = {k: v for k, v in result.items() if k not in ("ok", "error")}
    err = result.get("error", "")
    print(f"  [{icon:4}] {check:<25} {detail} {err}")

score = sum(1 for r in checks.values() if r.get("ok"))
print("=" * 60)
print(f"  Score: {score}/{len(checks)} checks passed")
print()
```

```bash
python3 s3_health_check.py
```

**Expected output:**
```
S3 Bucket Health Check: s3-howto-123456789012-us-east-1
============================================================
  [OK  ] versioning               {'status': 'Enabled'}
  [OK  ] encryption               {'algorithm': 'aws:kms'}
  [OK  ] block_public_access      {'config': {'BlockPublicAcls': True, ...}, ...}
  [OK  ] server_access_logging    {'enabled': True, 'target': 's3-howto-...-access-logs'}
  [FAIL] lifecycle                {'enabled_rules': []}
  [OK  ] bucket_policy            {'has_https_deny': True}
============================================================
  Score: 5/6 checks passed
```

---

### Step 7 — Cost optimisation report using boto3

```python
#!/usr/bin/env python3
# s3_cost_report.py
"""
Generate a cost optimisation report identifying:
1. Objects in Standard that haven't been accessed in 30+ days
2. Stalled incomplete multipart uploads
3. Orphaned delete markers in versioned buckets
"""
import boto3, os
from datetime import datetime, timezone, timedelta
from collections import defaultdict

s3 = boto3.client("s3", region_name="us-east-1")
BUCKET = os.environ["BUCKET_NAME"]
NOW = datetime.now(timezone.utc)

print(f"\nS3 Cost Optimisation Report: {BUCKET}")
print(f"Generated: {NOW.isoformat()}\n")

# 1. Objects in STANDARD storage class (candidates for tiering)
paginator = s3.get_paginator("list_objects_v2")
standard_objects = []
for page in paginator.paginate(Bucket=BUCKET):
    for obj in page.get("Contents", []):
        age_days = (NOW - obj["LastModified"]).days
        if obj["StorageClass"] == "STANDARD" and age_days > 30:
            standard_objects.append({
                "key": obj["Key"],
                "size_kb": round(obj["Size"] / 1024, 1),
                "age_days": age_days,
            })

print(f"[1] STANDARD objects older than 30 days (consider STANDARD_IA or lifecycle rule):")
if standard_objects:
    for o in standard_objects[:5]:
        print(f"    {o['key']:<50} {o['size_kb']:>8} KB   {o['age_days']} days old")
    if len(standard_objects) > 5:
        print(f"    ... and {len(standard_objects) - 5} more")
else:
    print("    None found — good!")

# 2. Stalled multipart uploads
stalled = []
try:
    for page in s3.get_paginator("list_multipart_uploads").paginate(Bucket=BUCKET):
        for upload in page.get("Uploads", []):
            age_hours = (NOW - upload["Initiated"]).total_seconds() / 3600
            stalled.append({"key": upload["Key"], "age_hours": round(age_hours, 1), "upload_id": upload["UploadId"]})
except Exception:
    pass

print(f"\n[2] Stalled multipart uploads (wasting money):")
if stalled:
    for u in stalled:
        print(f"    {u['key']:<50} {u['age_hours']} hours old")
    # Estimate wasted cost (min 5 MB parts at $0.023/GB)
    print(f"    Recommendation: run abort_stalled_multipart_uploads()")
else:
    print("    None found — clean!")

# 3. Delete markers (orphaned versions)
try:
    versions = s3.list_object_versions(Bucket=BUCKET)
    markers = versions.get("DeleteMarkers", [])
    print(f"\n[3] Delete markers (versioning overhead): {len(markers)} found")
    if markers:
        print("    Recommendation: add NoncurrentVersionExpiration lifecycle rule")
    else:
        print("    None found — clean!")
except Exception:
    print("\n[3] Delete markers: versioning not enabled or no markers")

print("\nReport complete.\n")
```

```bash
python3 s3_cost_report.py
```

---

## Full Cleanup — All Labs

```bash
# Remove all remaining objects
aws s3 rm s3://$BUCKET_NAME --recursive

# Remove logging config
aws s3api put-bucket-logging \
  --bucket "$BUCKET_NAME" \
  --bucket-logging-status '{}'

# Delete logging bucket
aws s3 rm s3://$LOGS_BUCKET --recursive
aws s3api delete-bucket --bucket "$LOGS_BUCKET"

# Delete the main bucket (must be empty)
aws s3api delete-bucket --bucket "$BUCKET_NAME"

# Delete CloudWatch alarms
aws cloudwatch delete-alarms \
  --alarm-names "S3HowTo-4xxErrors-High" "S3HowTo-5xxErrors-High"

# Delete SNS alarm topic
aws sns delete-topic --topic-arn "$ALARM_SNS_ARN"

echo "All S3 HowTo lab resources deleted."
```

---

## Summary — What You've Learned

| Lab | Key Takeaways |
|-----|--------------|
| 01 | S3 object model, key/prefix, storage classes, strong consistency |
| 02 | CRUD operations, multipart uploads, presigned URLs |
| 03 | Lifecycle rules, Intelligent-Tiering, cost modelling |
| 04 | BPA, bucket policy, SSE-S3/KMS/C, enforced encryption |
| 05 | Versioning, CRR, Object Lock, S3 Select |
| 06 | Event-driven pipeline: S3 → Lambda → DynamoDB → SNS |
| 07 | Production hardening: logging, metrics, alarms, health checks, cost reporting |
