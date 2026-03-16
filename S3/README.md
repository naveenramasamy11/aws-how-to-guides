# Amazon S3 — Simple Storage Service

> **Module difficulty:** 🟢 Beginner → 🔴 Advanced | 7 hands-on PoC labs

Amazon S3 is the backbone of AWS storage — infinitely scalable object storage used for backups, data lakes, static websites, ML training datasets, and event-driven pipelines. Mastering S3 means mastering the foundation of almost every AWS workload.

---

## Learning Path

```
Lab 01 ──► Lab 02 ──► Lab 03 ──► Lab 04 ──► Lab 05 ──► Lab 06 ──► Lab 07
Core        Bucket     Storage    Security   Advanced   Event       Production
Concepts    Ops        Classes    & Enc.     Features   Pipeline    Patterns
🟢          🟢         🟡         🟡         🔴         🔴          🔴
```

---

## Labs Index

| Lab | Title | Difficulty | Key Skills |
|-----|-------|------------|------------|
| [01](./01-s3-core-concepts/README.md) | S3 Core Concepts | 🟢 Beginner | Buckets, objects, keys, regions, consistency |
| [02](./02-bucket-operations/README.md) | Bucket & Object Operations | 🟢 Beginner | CLI CRUD, multipart upload, presigned URLs |
| [03](./03-storage-classes-lifecycle/README.md) | Storage Classes & Lifecycle | 🟡 Intermediate | Intelligent-Tiering, Glacier, lifecycle rules |
| [04](./04-security-encryption/README.md) | Security & Encryption | 🟡 Intermediate | Bucket policies, ACLs, SSE-S3/KMS/C, Block Public Access |
| [05](./05-advanced-features/README.md) | Advanced Features | 🔴 Advanced | Versioning, replication, Object Lock, S3 Select |
| [06](./06-event-driven-pipeline/README.md) | Event-Driven Pipeline | 🔴 Advanced | S3 Events → Lambda → DynamoDB/SNS |
| [07](./07-production-patterns/README.md) | Production Patterns | 🔴 Advanced | CloudWatch metrics, cost optimisation, access logging |

---

## Core Concepts Quick-Reference

| Concept | Description |
|---------|-------------|
| **Bucket** | Global-namespace container; region-bound, unlimited objects |
| **Object** | Key + data + metadata; max 5 TB per object |
| **Key** | Full path string (prefix + filename); no real folders |
| **Consistency** | Strong read-after-write consistency for all operations (since Dec 2020) |
| **Storage Class** | Cost/durability trade-off tier assigned per object |
| **Versioning** | Retain every version of an object; enable once, suspend later |
| **Replication** | CRR (cross-region) or SRR (same-region) object replication |
| **Presigned URL** | Time-limited URL granting temp access to a private object |
| **Transfer Acceleration** | Routes uploads via CloudFront edge nodes for speed |
| **Object Lock** | WORM (write-once-read-many) protection via Compliance/Governance modes |

---

## ASCII Architecture Diagram

```
┌─────────────────────────────────────────────────────────────┐
│                         AWS Account                         │
│                                                             │
│   ┌──────────────────────────────────────────────────────┐  │
│   │                  S3 Bucket (us-east-1)               │  │
│   │                                                      │  │
│   │  ┌────────────┐  ┌────────────┐  ┌────────────────┐  │  │
│   │  │  Standard  │  │    IA /    │  │   Glacier /    │  │  │
│   │  │  Objects   │  │  One-Zone  │  │  Deep Archive  │  │  │
│   │  └────────────┘  └────────────┘  └────────────────┘  │  │
│   │         ▲                ▲                            │  │
│   │         │    Lifecycle   │                            │  │
│   │         └────── Rules ───┘                            │  │
│   │                                                      │  │
│   │  Bucket Policy / ACL / Block-Public-Access           │  │
│   │  Versioning ──► Object Lock (WORM)                   │  │
│   │  Event Notifications ──► Lambda / SNS / SQS          │  │
│   └──────────────────────────────────────────────────────┘  │
│            │                         │                      │
│     IAM Roles                  KMS CMK (SSE-KMS)            │
│     (Least Privilege)          (Envelope Encryption)        │
└─────────────────────────────────────────────────────────────┘
```

---

## Setup — Prerequisites

```bash
# Verify AWS CLI version (v2 required)
aws --version

# Configure credentials (if not using IAM role/instance profile)
aws configure
# AWS Access Key ID: <your-key>
# AWS Secret Access Key: <your-secret>
# Default region: us-east-1
# Default output format: json

# Set a reusable variable for all labs
export AWS_REGION="us-east-1"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-howto-${ACCOUNT_ID}-${AWS_REGION}"
echo "Bucket name: ${BUCKET_NAME}"
```

---

## CLI Cheat Sheet

```bash
# Create bucket
aws s3api create-bucket --bucket $BUCKET_NAME --region $AWS_REGION

# Upload file
aws s3 cp file.txt s3://$BUCKET_NAME/prefix/file.txt

# List objects
aws s3 ls s3://$BUCKET_NAME --recursive --human-readable

# Download file
aws s3 cp s3://$BUCKET_NAME/prefix/file.txt ./downloaded.txt

# Delete object
aws s3 rm s3://$BUCKET_NAME/prefix/file.txt

# Sync directory
aws s3 sync ./local-dir s3://$BUCKET_NAME/remote-dir/

# Presigned URL (expires in 1 hour)
aws s3 presign s3://$BUCKET_NAME/file.txt --expires-in 3600

# Set storage class on upload
aws s3 cp file.txt s3://$BUCKET_NAME/file.txt --storage-class INTELLIGENT_TIERING

# Get bucket size
aws s3 ls s3://$BUCKET_NAME --recursive --human-readable --summarize | tail -2

# Delete bucket (empty first)
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3api delete-bucket --bucket $BUCKET_NAME
```
