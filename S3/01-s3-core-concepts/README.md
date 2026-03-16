# Lab 01 — S3 Core Concepts 🟢

## Goal

Understand the fundamental building blocks of Amazon S3: buckets, objects, keys, regions, storage classes, and the consistency model — then validate them with hands-on CLI commands.

---

## Concepts

### S3 Object Model

| Component | Description | Example |
|-----------|-------------|---------|
| **Bucket** | Top-level namespace container; globally unique name | `my-app-data-123456789` |
| **Object** | The actual file/data stored in a bucket | `report.pdf` |
| **Key** | Full path to an object (prefix + filename) | `reports/2024/q1/report.pdf` |
| **Value** | The binary content of the object (up to 5 TB) | file bytes |
| **Metadata** | Key-value pairs attached to an object | `Content-Type: application/pdf` |
| **Version ID** | Unique ID assigned when versioning is enabled | `3HL4kqtJlcpXrOFjWmS9PH4w` |
| **ETag** | MD5 checksum of object content | `"d41d8cd98f00b204e9800998ecf8427e"` |

### Key vs Folder

S3 has no real directory structure. The `/` in a key is just a character — the AWS console presents prefixes as "folders" for UX convenience.

```
Key: "data/2024/jan/sales.csv"
      ─────────────┬───────────
                   │
              prefix = "data/2024/jan/"   ← treated as folder in console
              filename = "sales.csv"
```

### Consistency Model (Post-Dec 2020)

| Operation | Consistency |
|-----------|-------------|
| PUT new object | Strong read-after-write |
| PUT overwrite | Strong consistency |
| DELETE | Strong consistency |
| LIST | Strong consistency |

### S3 Storage Classes Overview

| Storage Class | Use Case | Availability | Min Duration | Retrieval |
|---------------|----------|--------------|--------------|-----------|
| S3 Standard | Frequent access | 99.99% | None | Immediate |
| S3 Intelligent-Tiering | Unknown/changing access | 99.9% | None | Immediate |
| S3 Standard-IA | Infrequent access | 99.9% | 30 days | Immediate |
| S3 One Zone-IA | Infrequent, non-critical | 99.5% | 30 days | Immediate |
| S3 Glacier Instant | Archive, ms retrieval | 99.9% | 90 days | Milliseconds |
| S3 Glacier Flexible | Archive, min-hr retrieval | 99.99% | 90 days | 1–12 hours |
| S3 Glacier Deep Archive | Rarely accessed archive | 99.99% | 180 days | Up to 48 hours |

### Pricing Dimensions

| Dimension | What you pay for |
|-----------|-----------------|
| Storage | GB-month stored per storage class |
| Requests | PUT, COPY, POST, LIST (per 1,000) |
| Retrieval | GET, SELECT (per 1,000 + per GB for IA/Glacier) |
| Data Transfer OUT | Per GB transferred to internet or other regions |
| Replication | Per GB replicated |

---

## Architecture Diagram

```
  Client
    │
    ▼  HTTPS
┌──────────────────────────────────┐
│  S3 Endpoint (s3.amazonaws.com)  │
└──────────────────────────────────┘
          │
          ▼
┌──────────────────────────────────────────────────────┐
│  Bucket: my-app-data-123456789  (us-east-1)          │
│                                                      │
│  Object key: "data/2024/jan/sales.csv"               │
│  ┌──────────────────────────────────────────────┐    │
│  │ Metadata: Content-Type, Last-Modified, ETag  │    │
│  │ Storage Class: STANDARD                      │    │
│  │ Value: <binary content, 1.2 MB>              │    │
│  └──────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────┘
```

---

## PoC Steps

### Step 1 — Set environment variables

```bash
export AWS_REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-howto-${ACCOUNT_ID}-${AWS_REGION}"
echo "Working bucket: ${BUCKET_NAME}"
```

**Expected output:**
```
Working bucket: s3-howto-123456789012-us-east-1
```

---

### Step 2 — Create an S3 bucket

```bash
# us-east-1 does NOT use --create-bucket-configuration; all other regions do
aws s3api create-bucket \
  --bucket "$BUCKET_NAME" \
  --region "$AWS_REGION" \
  --tags Key=Project,Value=S3HowTo Key=Environment,Value=Lab \
  --query Location \
  --output text
```

**Expected output:**
```
/s3-howto-123456789012-us-east-1
```

---

### Step 3 — Inspect bucket metadata

```bash
# Verify region binding
aws s3api get-bucket-location \
  --bucket "$BUCKET_NAME" \
  --query LocationConstraint \
  --output text

# View all bucket attributes
aws s3api get-bucket-tagging \
  --bucket "$BUCKET_NAME" \
  --output table
```

**Expected output:**
```
None          ← us-east-1 returns "None" (null) for LocationConstraint

-----------------------------------------------------------------------
|                          GetBucketTagging                           |
+------------------+--------------------------------------------------+
|  Key             |  Value                                           |
+------------------+--------------------------------------------------+
|  Project         |  S3HowTo                                         |
|  Environment     |  Lab                                             |
+------------------+--------------------------------------------------+
```

---

### Step 4 — Upload objects with different storage classes

```bash
# Create sample files
echo "This is a frequently accessed file" > /tmp/standard.txt
echo "This is an infrequently accessed file" > /tmp/infrequent.txt
echo "This is an archival file" > /tmp/archive.txt

# Upload with different storage classes
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "concepts/standard.txt" \
  --body /tmp/standard.txt \
  --storage-class STANDARD \
  --tagging "StorageClass=Standard&Lab=01"

aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "concepts/infrequent.txt" \
  --body /tmp/infrequent.txt \
  --storage-class STANDARD_IA \
  --tagging "StorageClass=StandardIA&Lab=01"

aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "concepts/archive.txt" \
  --body /tmp/archive.txt \
  --storage-class INTELLIGENT_TIERING \
  --tagging "StorageClass=IntelligentTiering&Lab=01"
```

**Expected output for each:**
```json
{
    "ETag": "\"a0f1490a20d0211c997b44bc357e1972\""
}
```

---

### Step 5 — List objects and inspect metadata

```bash
# List all objects
aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --prefix "concepts/" \
  --query 'Contents[*].{Key:Key,Size:Size,StorageClass:StorageClass}' \
  --output table

# Inspect head of a single object (all metadata)
aws s3api head-object \
  --bucket "$BUCKET_NAME" \
  --key "concepts/standard.txt" \
  --output table
```

**Expected output:**
```
---------------------------------------------------------------------------
|                          ListObjectsV2                                  |
+-------------------------------+-------+----------------------------------+
|  Key                          | Size  | StorageClass                     |
+-------------------------------+-------+----------------------------------+
|  concepts/archive.txt         |  23   | INTELLIGENT_TIERING              |
|  concepts/infrequent.txt      |  37   | STANDARD_IA                      |
|  concepts/standard.txt        |  35   | STANDARD                         |
+-------------------------------+-------+----------------------------------+
```

---

### Step 6 — Demonstrate key (prefix) behaviour

```bash
# Upload objects that simulate a "folder" structure
for month in jan feb mar; do
  echo "Sales data for $month 2024" > /tmp/sales_${month}.csv
  aws s3api put-object \
    --bucket "$BUCKET_NAME" \
    --key "data/2024/${month}/sales.csv" \
    --body /tmp/sales_${month}.csv \
    --tags Key=Lab,Value=01
done

# List using a prefix (simulates folder navigation)
aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --prefix "data/2024/" \
  --delimiter "/" \
  --query '{Folders:CommonPrefixes[*].Prefix, Files:Contents[*].Key}' \
  --output table
```

**Expected output:**
```
{
    "Folders": [
        "data/2024/feb/",
        "data/2024/jan/",
        "data/2024/mar/"
    ],
    "Files": null
}
```

---

### Step 7 — Verify strong consistency

```bash
# Write an object then immediately read it back — strong consistency guarantees this works
echo "Consistency test $(date +%s)" > /tmp/consistency.txt

aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "concepts/consistency.txt" \
  --body /tmp/consistency.txt

# Immediately GET — should always succeed with the latest value
aws s3api get-object \
  --bucket "$BUCKET_NAME" \
  --key "concepts/consistency.txt" \
  /tmp/consistency_downloaded.txt

cat /tmp/consistency_downloaded.txt
```

**Expected output:**
```
Consistency test 1711234567
```

---

## Cleanup

```bash
# Delete all objects under the concepts/ and data/ prefixes
aws s3 rm s3://$BUCKET_NAME/concepts/ --recursive
aws s3 rm s3://$BUCKET_NAME/data/ --recursive

# Confirm the bucket is now empty
aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --query 'KeyCount' \
  --output text
# Expected: 0

# NOTE: Do NOT delete the bucket itself — it is reused in all subsequent labs
```

---

➡️ **Next:** [Lab 02 — Bucket & Object Operations](../02-bucket-operations/README.md)
