# Lab 04 — Security & Encryption 🟡

## Goal

Lock down an S3 bucket using bucket policies, Block Public Access, server-side encryption (SSE-S3, SSE-KMS, SSE-C), and enforce HTTPS-only access — following AWS security best practices.

---

## Concepts

### Encryption Options

| Method | Key Management | Use Case | Extra Cost |
|--------|---------------|----------|------------|
| **SSE-S3** (AES-256) | AWS manages keys entirely | Default; lowest friction | None |
| **SSE-KMS** | AWS KMS CMK; you control key policy | Audit trail, cross-account, key rotation | KMS API calls ($0.03/10K) |
| **SSE-C** | Customer provides key per request | Maximum key control | None |
| **Client-Side** | Encrypt before upload | Regulated data needing end-to-end | None |

### Block Public Access (BPA)

All four settings should be ON for private buckets:

| Setting | Blocks |
|---------|--------|
| `BlockPublicAcls` | New public ACLs |
| `IgnorePublicAcls` | Existing public ACLs |
| `BlockPublicPolicy` | New public bucket policies |
| `RestrictPublicBuckets` | Public bucket policy access |

### Bucket Policy vs ACL vs IAM

| Mechanism | Scope | Recommended |
|-----------|-------|-------------|
| Bucket Policy | Bucket/prefix/object level; supports cross-account | Yes — primary mechanism |
| IAM Policy | Identity-level; attached to users/roles | Yes — for same-account access |
| ACL | Legacy; object-level | No — disable for new buckets |

### Common Bucket Policy Conditions

| Condition | Key | Use |
|-----------|-----|-----|
| Force HTTPS | `aws:SecureTransport` | Deny HTTP requests |
| Source VPC | `aws:SourceVpc` | Restrict to VPC access only |
| Source IP | `aws:SourceIp` | Restrict to specific CIDRs |
| MFA required | `aws:MultiFactorAuthPresent` | MFA-delete protection |

---

## Architecture Diagram

```
        Client Request
              │
              ▼
  ┌─────────────────────┐
  │  S3 Bucket Policy   │──── Deny if !SecureTransport
  │  Block Public Access│──── Block all public access
  └─────────────────────┘
              │
              ▼
  ┌─────────────────────────────────────────────────┐
  │  S3 Object                                      │
  │                                                 │
  │  ┌────────────────────────────────────────────┐ │
  │  │ SSE-KMS Encryption                         │ │
  │  │                                            │ │
  │  │ Data Key ◄──── AWS KMS CMK                 │ │
  │  │ (Envelope Encryption)                      │ │
  │  └────────────────────────────────────────────┘ │
  └─────────────────────────────────────────────────┘
              │
              ▼
         CloudTrail
    (KMS API audit log)
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

### Step 2 — Enable Block Public Access on the bucket

```bash
aws s3api put-public-access-block \
  --bucket "$BUCKET_NAME" \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# Verify
aws s3api get-public-access-block \
  --bucket "$BUCKET_NAME" \
  --output table
```

**Expected output:**
```
-------------------------------------------------------------
|                   GetPublicAccessBlock                    |
+---------------------------+-------------------------------+
|  BlockPublicAcls          |  True                         |
|  BlockPublicPolicy        |  True                         |
|  IgnorePublicAcls         |  True                         |
|  RestrictPublicBuckets    |  True                         |
+---------------------------+-------------------------------+
```

---

### Step 3 — Apply a bucket policy (deny HTTP, enforce TLS)

```bash
cat > /tmp/bucket-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonHTTPS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::${BUCKET_NAME}",
        "arn:aws:s3:::${BUCKET_NAME}/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket "$BUCKET_NAME" \
  --policy file:///tmp/bucket-policy.json

# Verify
aws s3api get-bucket-policy \
  --bucket "$BUCKET_NAME" \
  --query Policy \
  --output text | python3 -m json.tool
```

---

### Step 4 — Enable default SSE-S3 encryption on the bucket

```bash
# Set SSE-S3 (AES-256) as the bucket default
aws s3api put-bucket-encryption \
  --bucket "$BUCKET_NAME" \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "AES256"
      },
      "BucketKeyEnabled": false
    }]
  }'

# Verify
aws s3api get-bucket-encryption \
  --bucket "$BUCKET_NAME" \
  --query 'ServerSideEncryptionConfiguration.Rules[0]' \
  --output table

# Upload a file — it will be encrypted automatically
echo "Encrypted by SSE-S3" > /tmp/sse_s3_test.txt
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "security/sse-s3-test.txt" \
  --body /tmp/sse_s3_test.txt \
  --tags "Lab=04&Encryption=SSE-S3"

# Verify encryption in object metadata
aws s3api head-object \
  --bucket "$BUCKET_NAME" \
  --key "security/sse-s3-test.txt" \
  --query '{Encryption:ServerSideEncryption,ContentType:ContentType}' \
  --output table
```

**Expected output:**
```
-----------------------------------------------
|             HeadObject                      |
+---------------+-----------------------------+
|  ContentType  |  binary/octet-stream        |
|  Encryption   |  AES256                     |
+---------------+-----------------------------+
```

---

### Step 5 — Create a KMS CMK and switch to SSE-KMS

```bash
# Create a customer-managed KMS key for S3
KMS_KEY_ID=$(aws kms create-key \
  --description "S3 HowTo Lab KMS Key" \
  --key-usage ENCRYPT_DECRYPT \
  --origin AWS_KMS \
  --tags TagKey=Project,TagValue=S3HowTo TagKey=Lab,TagValue=04 \
  --query KeyMetadata.KeyId \
  --output text)
echo "KMS Key ID: $KMS_KEY_ID"

# Create a friendly alias
aws kms create-alias \
  --alias-name "alias/s3-howto-lab" \
  --target-key-id "$KMS_KEY_ID"

# Update bucket default encryption to SSE-KMS
aws s3api put-bucket-encryption \
  --bucket "$BUCKET_NAME" \
  --server-side-encryption-configuration "{
    \"Rules\": [{
      \"ApplyServerSideEncryptionByDefault\": {
        \"SSEAlgorithm\": \"aws:kms\",
        \"KMSMasterKeyID\": \"${KMS_KEY_ID}\"
      },
      \"BucketKeyEnabled\": true
    }]
  }"

# Upload a file — it will be encrypted with SSE-KMS
echo "Encrypted by SSE-KMS with CMK" > /tmp/sse_kms_test.txt
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "security/sse-kms-test.txt" \
  --body /tmp/sse_kms_test.txt \
  --tags "Lab=04&Encryption=SSE-KMS"

# Verify SSE-KMS encryption
aws s3api head-object \
  --bucket "$BUCKET_NAME" \
  --key "security/sse-kms-test.txt" \
  --query '{Encryption:ServerSideEncryption,KMSKeyId:SSEKMSKeyId}' \
  --output table
```

**Expected output:**
```
------------------------------------------------------------
|                       HeadObject                        |
+-------------+--------------------------------------------+
|  Encryption |  aws:kms                                  |
|  KMSKeyId   |  arn:aws:kms:us-east-1:123...             |
+-------------+--------------------------------------------+
```

---

### Step 6 — SSE-C (customer-provided key)

```bash
# Generate a 32-byte AES-256 key and its MD5
CUSTOMER_KEY=$(openssl rand -base64 32)
CUSTOMER_KEY_MD5=$(echo -n "$CUSTOMER_KEY" | base64 -d | openssl md5 -binary | base64)
echo "Customer Key: $CUSTOMER_KEY"
echo "Customer Key MD5: $CUSTOMER_KEY_MD5"

# Upload with SSE-C
echo "Protected with SSE-C customer key" > /tmp/sse_c_test.txt
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "security/sse-c-test.txt" \
  --body /tmp/sse_c_test.txt \
  --sse-customer-algorithm AES256 \
  --sse-customer-key "$CUSTOMER_KEY" \
  --sse-customer-key-md5 "$CUSTOMER_KEY_MD5" \
  --query '{ETag:ETag,Encryption:ServerSideEncryption}' \
  --output table

# To retrieve SSE-C object, you MUST supply the key on every GET
aws s3api get-object \
  --bucket "$BUCKET_NAME" \
  --key "security/sse-c-test.txt" \
  --sse-customer-algorithm AES256 \
  --sse-customer-key "$CUSTOMER_KEY" \
  --sse-customer-key-md5 "$CUSTOMER_KEY_MD5" \
  /tmp/sse_c_downloaded.txt

cat /tmp/sse_c_downloaded.txt
```

**Expected output:**
```
Protected with SSE-C customer key
```

---

### Step 7 — Enforce encrypted uploads via bucket policy

```bash
cat > /tmp/enforce-encryption-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "DenyNonHTTPS",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::${BUCKET_NAME}",
        "arn:aws:s3:::${BUCKET_NAME}/*"
      ],
      "Condition": {
        "Bool": {"aws:SecureTransport": "false"}
      }
    },
    {
      "Sid": "DenyUnencryptedObjectUploads",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/*",
      "Condition": {
        "StringNotEquals": {
          "s3:x-amz-server-side-encryption": ["AES256", "aws:kms"]
        }
      }
    }
  ]
}
EOF

aws s3api put-bucket-policy \
  --bucket "$BUCKET_NAME" \
  --policy file:///tmp/enforce-encryption-policy.json

echo "Bucket policy updated — unencrypted uploads now blocked"

# Test: attempt unencrypted upload (should fail with 403)
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "security/unencrypted-test.txt" \
  --body /tmp/sse_s3_test.txt \
  --no-sign-request 2>&1 || echo "Expected: Access denied for unencrypted upload"
```

---

## Cleanup

```bash
# Remove objects
aws s3 rm s3://$BUCKET_NAME/security/ --recursive

# Disable the enforce-encryption policy (restore minimal policy)
cat > /tmp/minimal-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyNonHTTPS",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::${BUCKET_NAME}",
      "arn:aws:s3:::${BUCKET_NAME}/*"
    ],
    "Condition": {"Bool": {"aws:SecureTransport": "false"}}
  }]
}
EOF
aws s3api put-bucket-policy --bucket "$BUCKET_NAME" --policy file:///tmp/minimal-policy.json

# Schedule KMS key for deletion (7-day waiting period)
aws kms schedule-key-deletion \
  --key-id "$KMS_KEY_ID" \
  --pending-window-in-days 7

echo "Cleanup complete"
```

---

➡️ **Next:** [Lab 05 — Advanced Features](../05-advanced-features/README.md)
