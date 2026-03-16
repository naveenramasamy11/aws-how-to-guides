# Lab 02 — Bucket & Object Operations 🟢

## Goal

Master the full CRUD lifecycle for S3 objects using both the high-level `aws s3` CLI and the low-level `aws s3api`. Covers multipart uploads for large files and presigned URLs for temporary access delegation.

---

## Concepts

### High-Level (`aws s3`) vs Low-Level (`aws s3api`)

| Command | Abstraction | Best For |
|---------|-------------|----------|
| `aws s3 cp` | High-level | Simple copy/upload/download |
| `aws s3 sync` | High-level | Directory-level synchronisation |
| `aws s3 ls` | High-level | Quick listing |
| `aws s3api put-object` | Low-level | Fine-grained metadata/tagging control |
| `aws s3api create-multipart-upload` | Low-level | Files > 5 GB; parallel part uploads |
| `aws s3api get-object` | Low-level | Range requests, checksum validation |

### Multipart Upload Thresholds

| File Size | Recommendation |
|-----------|---------------|
| < 100 MB | Single PUT |
| 100 MB – 5 GB | Multipart (optional but faster) |
| > 5 GB | Multipart REQUIRED (5 GB single PUT limit) |
| Max object size | 5 TB (via multipart) |

### Presigned URLs

- Signed with your IAM credentials at generation time
- The recipient needs NO AWS credentials to use the URL
- Honour the generating user's permissions at time of use (not generation)
- Default expiry: 3,600 seconds; max: 7 days (604,800 s)

---

## Architecture Diagram

```
  Your Machine / CI Pipeline
        │
        │  aws s3 cp / aws s3api put-object
        ▼
┌───────────────────────────────────────────────────┐
│  S3 Bucket                                        │
│                                                   │
│  ┌─────────────────────────────────────────────┐  │
│  │  Multipart Upload (parallel parts)          │  │
│  │  Part 1 (5 MB) ──┐                          │  │
│  │  Part 2 (5 MB) ──┼──► CompleteMultipart     │  │
│  │  Part 3 (5 MB) ──┘     ──► Assembled Object │  │
│  └─────────────────────────────────────────────┘  │
│                                                   │
│  Presigned URL ◄──── aws s3 presign ─────────┐    │
│       │                                      │    │
│       └──► Anyone with URL can GET/PUT ───────────┘
└───────────────────────────────────────────────────┘
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

### Step 2 — Upload and download with `aws s3 cp`

```bash
# Create a sample file
echo "Hello from S3 Lab 02" > /tmp/hello.txt

# Upload
aws s3 cp /tmp/hello.txt s3://$BUCKET_NAME/lab02/hello.txt \
  --tags "Lab=02&Purpose=Demo"

# Download to a different path
aws s3 cp s3://$BUCKET_NAME/lab02/hello.txt /tmp/hello_downloaded.txt

cat /tmp/hello_downloaded.txt
```

**Expected output:**
```
upload: /tmp/hello.txt to s3://s3-howto-123456789012-us-east-1/lab02/hello.txt
download: s3://s3-howto-123456789012-us-east-1/lab02/hello.txt to /tmp/hello_downloaded.txt
Hello from S3 Lab 02
```

---

### Step 3 — Sync a local directory to S3

```bash
# Create a local directory with several files
mkdir -p /tmp/sync-source
for i in 1 2 3 4 5; do
  echo "File number $i — created at $(date -u +%Y-%m-%dT%H:%M:%SZ)" > /tmp/sync-source/file_${i}.txt
done

# First sync — uploads all 5 files
aws s3 sync /tmp/sync-source s3://$BUCKET_NAME/lab02/sync-demo/ \
  --tags "Lab=02"

# Modify one file and add a new one, then sync again
echo "Updated content" > /tmp/sync-source/file_3.txt
echo "Brand new file" > /tmp/sync-source/file_6.txt

# Second sync — only uploads file_3.txt (changed) and file_6.txt (new)
aws s3 sync /tmp/sync-source s3://$BUCKET_NAME/lab02/sync-demo/

# Verify
aws s3 ls s3://$BUCKET_NAME/lab02/sync-demo/ --output table 2>/dev/null || \
aws s3 ls s3://$BUCKET_NAME/lab02/sync-demo/
```

**Expected output (second sync):**
```
upload: /tmp/sync-source/file_3.txt to s3://.../lab02/sync-demo/file_3.txt
upload: /tmp/sync-source/file_6.txt to s3://.../lab02/sync-demo/file_6.txt
```

---

### Step 4 — Copy and move objects within S3

```bash
# Copy an object to a new key (server-side — no data transfer costs)
aws s3api copy-object \
  --bucket "$BUCKET_NAME" \
  --copy-source "${BUCKET_NAME}/lab02/hello.txt" \
  --key "lab02/hello-copy.txt" \
  --tagging-directive COPY \
  --query 'CopyObjectResult.{ETag:ETag,LastModified:LastModified}' \
  --output table

# Verify both objects exist
aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --prefix "lab02/hello" \
  --query 'Contents[*].{Key:Key,Size:Size}' \
  --output table

# "Move" = copy + delete source (S3 has no native rename)
aws s3api delete-object \
  --bucket "$BUCKET_NAME" \
  --key "lab02/hello.txt" \
  --query 'DeleteMarker' \
  --output text
```

**Expected output:**
```
--------------------------------------------------------------
|                       CopyObject                          |
+--------------------------+---------------------------------+
|  ETag                    |  LastModified                   |
+--------------------------+---------------------------------+
|  "a0f1490a20d0211c997b4" |  2024-03-16T10:00:00.000Z       |
+--------------------------+---------------------------------+
```

---

### Step 5 — Batch delete objects

```bash
# Generate 10 test objects
for i in $(seq 1 10); do
  aws s3api put-object \
    --bucket "$BUCKET_NAME" \
    --key "lab02/batch/obj_${i}.txt" \
    --body <(echo "Batch object $i") \
    --tags "Lab=02&Batch=true" > /dev/null
done

echo "Created 10 batch objects"

# Build delete JSON manifest
DELETE_JSON=$(aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --prefix "lab02/batch/" \
  --query '{Objects: Contents[*].{Key: Key}}' \
  --output json)

echo "Deleting: $(echo $DELETE_JSON | python3 -c 'import sys,json; print(len(json.loads(sys.stdin.read())["Objects"]))'  ) objects"

# Batch delete (max 1000 per request)
aws s3api delete-objects \
  --bucket "$BUCKET_NAME" \
  --delete "$DELETE_JSON" \
  --query 'Deleted[*].Key' \
  --output table
```

**Expected output:**
```
Created 10 batch objects
Deleting: 10 objects
--------------------------------------------
|              DeleteObjects               |
+------------------------------------------+
|  lab02/batch/obj_1.txt                   |
|  lab02/batch/obj_10.txt                  |
...
```

---

### Step 6 — Multipart upload for large files

```bash
# Create a 15 MB test file
dd if=/dev/urandom of=/tmp/large_file.bin bs=1M count=15 2>/dev/null
echo "File size: $(du -sh /tmp/large_file.bin | cut -f1)"

# Initiate multipart upload
UPLOAD_ID=$(aws s3api create-multipart-upload \
  --bucket "$BUCKET_NAME" \
  --key "lab02/large-file.bin" \
  --tagging "Lab=02&Type=Multipart" \
  --query UploadId \
  --output text)
echo "Upload ID: $UPLOAD_ID"

# Upload Part 1 (bytes 0–7,999,999 = 8 MB)
ETAG1=$(aws s3api upload-part \
  --bucket "$BUCKET_NAME" \
  --key "lab02/large-file.bin" \
  --part-number 1 \
  --upload-id "$UPLOAD_ID" \
  --body <(dd if=/tmp/large_file.bin bs=1M count=8 skip=0 2>/dev/null) \
  --query ETag \
  --output text)
echo "Part 1 ETag: $ETAG1"

# Upload Part 2 (remaining bytes ~7 MB)
ETAG2=$(aws s3api upload-part \
  --bucket "$BUCKET_NAME" \
  --key "lab02/large-file.bin" \
  --part-number 2 \
  --upload-id "$UPLOAD_ID" \
  --body <(dd if=/tmp/large_file.bin bs=1M count=7 skip=8 2>/dev/null) \
  --query ETag \
  --output text)
echo "Part 2 ETag: $ETAG2"

# Complete multipart upload
aws s3api complete-multipart-upload \
  --bucket "$BUCKET_NAME" \
  --key "lab02/large-file.bin" \
  --upload-id "$UPLOAD_ID" \
  --multipart-upload "{\"Parts\":[{\"PartNumber\":1,\"ETag\":$ETAG1},{\"PartNumber\":2,\"ETag\":$ETAG2}]}" \
  --query 'Key' \
  --output text
```

**Expected output:**
```
File size: 15M
Upload ID: VXBsb2FkIElEIGZvciA2aWWpbmcncyBteS1tb3ZpZQ
Part 1 ETag: "7e10e82425f173ebe3f7a8a8fd2d0e97"
Part 2 ETag: "83d9c6b62f0e0b3c9f9fa2df3fd27a1a"
lab02/large-file.bin
```

---

### Step 7 — Generate and use a presigned URL

```bash
# Upload a file to share
echo "Confidential report — for authorised recipients only" > /tmp/report.txt
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "lab02/report.txt" \
  --body /tmp/report.txt \
  --tags "Lab=02"

# Generate a presigned GET URL (valid for 10 minutes)
PRESIGNED_URL=$(aws s3 presign \
  s3://$BUCKET_NAME/lab02/report.txt \
  --expires-in 600)

echo "Presigned URL (expires in 10 min):"
echo "$PRESIGNED_URL"

# Use the presigned URL with curl (no AWS credentials needed)
curl -s "$PRESIGNED_URL"
```

**Expected output:**
```
Presigned URL (expires in 10 min):
https://s3-howto-123456789012-us-east-1.s3.amazonaws.com/lab02/report.txt?X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Credential=...&X-Amz-Expires=600...

Confidential report — for authorised recipients only
```

---

### Step 8 — Generate a presigned PUT URL (upload delegation)

```python
#!/usr/bin/env python3
# presigned_put.py
import boto3, os

s3 = boto3.client("s3", region_name="us-east-1")
bucket = os.environ["BUCKET_NAME"]

# Generate a presigned PUT URL — allows anyone to upload to this key
presigned_put = s3.generate_presigned_url(
    "put_object",
    Params={"Bucket": bucket, "Key": "lab02/uploaded-by-external.txt"},
    ExpiresIn=3600,
)
print("PUT URL:", presigned_put)

# Simulate external upload using the presigned URL
import urllib.request
req = urllib.request.Request(presigned_put, data=b"Uploaded via presigned PUT URL", method="PUT")
with urllib.request.urlopen(req) as resp:
    print(f"HTTP Status: {resp.status}")  # Expected: 200

# Verify the object was created
obj = s3.get_object(Bucket=bucket, Key="lab02/uploaded-by-external.txt")
print("Content:", obj["Body"].read().decode())
```

```bash
python3 presigned_put.py
```

**Expected output:**
```
PUT URL: https://s3-howto-...s3.amazonaws.com/lab02/uploaded-by-external.txt?X-Amz-...
HTTP Status: 200
Content: Uploaded via presigned PUT URL
```

---

## Cleanup

```bash
# Delete all lab02 objects
aws s3 rm s3://$BUCKET_NAME/lab02/ --recursive

# Verify
aws s3api list-objects-v2 \
  --bucket "$BUCKET_NAME" \
  --prefix "lab02/" \
  --query 'KeyCount' \
  --output text
# Expected: 0
```

---

➡️ **Next:** [Lab 03 — Storage Classes & Lifecycle](../03-storage-classes-lifecycle/README.md)
