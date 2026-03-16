# Lab 03 — Storage Classes & Lifecycle Rules 🟡

## Goal

Reduce S3 storage costs by automatically transitioning objects through storage classes using lifecycle rules — and verify the transitions with real configurations and cost comparisons.

---

## Concepts

### Storage Class Cost Comparison (us-east-1, approximate)

| Storage Class | Storage $/GB-mo | GET $/1K | Min Duration | Retrieval Fee |
|---------------|-----------------|----------|--------------|---------------|
| Standard | $0.023 | $0.0004 | None | None |
| Intelligent-Tiering | $0.023 (frequent) / $0.0125 (infrequent) | $0.0004 | None | None |
| Standard-IA | $0.0125 | $0.001 | 30 days | $0.01/GB |
| One Zone-IA | $0.01 | $0.001 | 30 days | $0.01/GB |
| Glacier Instant | $0.004 | $0.02 | 90 days | $0.03/GB |
| Glacier Flexible | $0.0036 | $0.0004 | 90 days | $0.01–$0.03/GB |
| Glacier Deep Archive | $0.00099 | $0.0004 | 180 days | $0.02/GB |

### Intelligent-Tiering Access Tiers

```
Days of No Access   Tier                  Storage Cost
─────────────────────────────────────────────────────
0–30 days        →  Frequent Access      $0.023/GB-mo
30–90 days       →  Infrequent Access    $0.0125/GB-mo
90–180 days      →  Archive Instant      $0.004/GB-mo
> 180 days       →  Deep Archive Access  $0.00099/GB-mo
```

Monitoring fee: $0.0025 per 1,000 objects. Free for objects < 128 KB.

### Lifecycle Rule Actions

| Action | Description |
|--------|-------------|
| `Transition` | Move objects to a cheaper storage class after N days |
| `Expiration` | Delete objects (or delete markers) after N days |
| `AbortIncompleteMultipartUpload` | Clean up stalled multipart uploads |
| `NoncurrentVersionTransition` | Transition older versions (versioned buckets) |
| `NoncurrentVersionExpiration` | Delete older versions after N days |

### Lifecycle Rule Filters

- **Prefix**: Apply rule only to keys starting with `logs/`
- **Tags**: Apply rule only to objects with tag `Type=Temporary`
- **Size**: Apply rule only to objects larger/smaller than N bytes
- **AND/OR combinations**: Multiple filter criteria

---

## Architecture Diagram

```
Object Created (Day 0)
         │
         ▼  Standard
  ───────────────────  Day 30
         │
         ▼  Standard-IA  (30-day minimum satisfied)
  ───────────────────  Day 90
         │
         ▼  Glacier Instant Retrieval
  ───────────────────  Day 180
         │
         ▼  Glacier Deep Archive
  ───────────────────  Day 365
         │
         ▼  DELETED (Expiration)

 ┌─────────────────────────────────┐
 │   Lifecycle Rule (JSON policy)  │
 │   Filter: prefix="logs/"        │
 │   Transitions: [30d→IA, 90d→GIR,│
 │                 180d→GDA]        │
 │   Expiration: 365 days           │
 └─────────────────────────────────┘
```

---

## PoC Steps

### Step 1 — Set variables

```bash
export AWS_REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-howto-${ACCOUNT_ID}-${AWS_REGION}"
echo "Bucket: ${BUCKET_NAME}"
```

---

### Step 2 — Upload objects to target with lifecycle

```bash
# Create sample log files (simulating daily logs)
for day in $(seq 1 5); do
  cat > /tmp/app_${day}.log << EOF
2024-0${day}-01 ERROR: Connection timeout after 30s
2024-0${day}-01 INFO:  Processed 1024 requests
2024-0${day}-01 WARN:  Memory usage at 78%
EOF
  aws s3api put-object \
    --bucket "$BUCKET_NAME" \
    --key "logs/app/2024-0${day}/app.log" \
    --body /tmp/app_${day}.log \
    --storage-class STANDARD \
    --tagging "Type=AppLog&Lab=03&Day=${day}"
done

echo "Uploaded 5 sample log files"

# Verify they're all STANDARD
aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --prefix "logs/" \
  --query 'Contents[*].{Key:Key,StorageClass:StorageClass,Size:Size}' \
  --output table
```

---

### Step 3 — Create a lifecycle rule for log archival

```bash
cat > /tmp/lifecycle-logs.json << 'EOF'
{
  "Rules": [
    {
      "ID": "log-archival-rule",
      "Status": "Enabled",
      "Filter": {
        "And": {
          "Prefix": "logs/",
          "Tags": [
            {"Key": "Type", "Value": "AppLog"}
          ]
        }
      },
      "Transitions": [
        {
          "Days": 30,
          "StorageClass": "STANDARD_IA"
        },
        {
          "Days": 90,
          "StorageClass": "GLACIER_IR"
        },
        {
          "Days": 180,
          "StorageClass": "DEEP_ARCHIVE"
        }
      ],
      "Expiration": {
        "Days": 365
      },
      "AbortIncompleteMultipartUploads": {
        "DaysAfterInitiation": 7
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket "$BUCKET_NAME" \
  --lifecycle-configuration file:///tmp/lifecycle-logs.json

echo "Lifecycle rule applied"
```

---

### Step 4 — Verify the lifecycle rule was applied

```bash
aws s3api get-bucket-lifecycle-configuration \
  --bucket "$BUCKET_NAME" \
  --query 'Rules[*].{ID:ID,Status:Status,Transitions:Transitions,Expiration:Expiration}' \
  --output json
```

**Expected output:**
```json
[
    {
        "ID": "log-archival-rule",
        "Status": "Enabled",
        "Transitions": [
            {"Days": 30, "StorageClass": "STANDARD_IA"},
            {"Days": 90, "StorageClass": "GLACIER_IR"},
            {"Days": 180, "StorageClass": "DEEP_ARCHIVE"}
        ],
        "Expiration": {"Days": 365}
    }
]
```

---

### Step 5 — Configure Intelligent-Tiering for dynamic workloads

```bash
# Upload objects for Intelligent-Tiering (suitable for unknown access patterns)
for i in $(seq 1 3); do
  echo "Dataset ${i}: $(head -c 200 /dev/urandom | base64 | head -c 200)" > /tmp/dataset_${i}.dat
  aws s3api put-object \
    --bucket "$BUCKET_NAME" \
    --key "data/datasets/dataset_${i}.dat" \
    --body /tmp/dataset_${i}.dat \
    --storage-class INTELLIGENT_TIERING \
    --tagging "Type=Dataset&Lab=03"
done

# Enable Intelligent-Tiering archive tiers at the bucket level
cat > /tmp/it-config.json << 'EOF'
{
  "Id": "EntireS3BucketIT",
  "Status": "Enabled",
  "Tierings": [
    {
      "Days": 90,
      "AccessTier": "ARCHIVE_ACCESS"
    },
    {
      "Days": 180,
      "AccessTier": "DEEP_ARCHIVE_ACCESS"
    }
  ]
}
EOF

aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket "$BUCKET_NAME" \
  --id "EntireS3BucketIT" \
  --intelligent-tiering-configuration file:///tmp/it-config.json

# Verify IT config
aws s3api get-bucket-intelligent-tiering-configuration \
  --bucket "$BUCKET_NAME" \
  --id "EntireS3BucketIT" \
  --query 'IntelligentTieringConfiguration.{Status:Status,Tierings:Tierings}' \
  --output table
```

---

### Step 6 — Add a second lifecycle rule for temp files

```bash
# Create a temporary file with tag
echo "Temporary scratch data — auto-expire in 7 days" > /tmp/temp.dat
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "temp/scratch/2024-03-16/temp.dat" \
  --body /tmp/temp.dat \
  --tagging "Type=Temp&Lab=03"

# Update lifecycle configuration with a second rule
cat > /tmp/lifecycle-full.json << 'EOF'
{
  "Rules": [
    {
      "ID": "log-archival-rule",
      "Status": "Enabled",
      "Filter": {
        "And": {
          "Prefix": "logs/",
          "Tags": [
            {"Key": "Type", "Value": "AppLog"}
          ]
        }
      },
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"},
        {"Days": 180, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 365},
      "AbortIncompleteMultipartUploads": {"DaysAfterInitiation": 7}
    },
    {
      "ID": "temp-file-expiry",
      "Status": "Enabled",
      "Filter": {
        "Tag": {"Key": "Type", "Value": "Temp"}
      },
      "Expiration": {
        "Days": 7
      }
    }
  ]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket "$BUCKET_NAME" \
  --lifecycle-configuration file:///tmp/lifecycle-full.json

# List all rules
aws s3api get-bucket-lifecycle-configuration \
  --bucket "$BUCKET_NAME" \
  --query 'Rules[*].{ID:ID,Status:Status}' \
  --output table
```

**Expected output:**
```
--------------------------------------------
|    GetBucketLifecycleConfiguration       |
+------------------------+-----------------+
|  ID                    |  Status         |
+------------------------+-----------------+
|  log-archival-rule     |  Enabled        |
|  temp-file-expiry      |  Enabled        |
+------------------------+-----------------+
```

---

### Step 7 — Python: cost estimation based on storage class

```python
#!/usr/bin/env python3
# cost_estimate.py
"""
Estimate monthly storage cost breakdown for objects in the bucket.
"""
import boto3, os
from collections import defaultdict

s3 = boto3.client("s3", region_name="us-east-1")
bucket = os.environ["BUCKET_NAME"]

# Storage class pricing ($/GB-month, us-east-1, approximate)
PRICING = {
    "STANDARD":            0.023,
    "INTELLIGENT_TIERING": 0.023,   # worst case (frequent tier)
    "STANDARD_IA":         0.0125,
    "ONEZONE_IA":          0.01,
    "GLACIER_IR":          0.004,
    "GLACIER":             0.0036,
    "DEEP_ARCHIVE":        0.00099,
}

# Gather all objects
paginator = s3.get_paginator("list_objects_v2")
class_totals = defaultdict(int)   # bytes per storage class

for page in paginator.paginate(Bucket=bucket):
    for obj in page.get("Contents", []):
        class_totals[obj["StorageClass"]] += obj["Size"]

print(f"\n{'Storage Class':<25} {'Total Size (MB)':>15} {'Monthly Cost ($)':>17}")
print("-" * 60)
total_cost = 0.0
for sc, total_bytes in sorted(class_totals.items()):
    gb = total_bytes / (1024 ** 3)
    cost = gb * PRICING.get(sc, 0.023)
    total_cost += cost
    print(f"{sc:<25} {total_bytes / (1024**2):>14.2f} MB   ${cost:>14.6f}")

print("-" * 60)
print(f"{'TOTAL ESTIMATED COST':<25} {'':>15} ${total_cost:>14.6f}/month")
```

```bash
python3 cost_estimate.py
```

**Expected output:**
```
Storage Class             Total Size (MB)   Monthly Cost ($)
------------------------------------------------------------
INTELLIGENT_TIERING               0.59 MB         $0.000013
STANDARD                          0.05 MB         $0.000001
------------------------------------------------------------
TOTAL ESTIMATED COST                              $0.000014/month
```

---

## Cleanup

```bash
# Delete objects created in this lab
aws s3 rm s3://$BUCKET_NAME/logs/ --recursive
aws s3 rm s3://$BUCKET_NAME/data/datasets/ --recursive
aws s3 rm s3://$BUCKET_NAME/temp/ --recursive

# Remove lifecycle configuration
aws s3api delete-bucket-lifecycle --bucket "$BUCKET_NAME"

# Remove Intelligent-Tiering configuration
aws s3api delete-bucket-intelligent-tiering-configuration \
  --bucket "$BUCKET_NAME" \
  --id "EntireS3BucketIT"

echo "Cleanup complete"
```

---

➡️ **Next:** [Lab 04 — Security & Encryption](../04-security-encryption/README.md)
