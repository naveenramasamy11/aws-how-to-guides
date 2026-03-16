# Lab 05 — Advanced Features 🔴

## Goal

Master S3's power features: versioning, Cross-Region Replication (CRR), Object Lock (WORM), and S3 Select for in-place SQL query on objects — critical for regulated industries, disaster recovery, and cost-efficient data processing.

---

## Concepts

### Versioning States

| State | Behaviour |
|-------|-----------|
| Disabled (default) | Overwrite replaces object; no recovery |
| Enabled | Every PUT creates a new version; DELETE creates a delete marker |
| Suspended | New objects get `null` version ID; existing versions preserved |

### Object Lock Modes

| Mode | Override / Delete | Retention | Use Case |
|------|-------------------|-----------|----------|
| Governance | IAM users with `s3:BypassGovernanceRetention` CAN delete | Per-object | Internal compliance testing |
| Compliance | NO ONE (not even root) can delete until expiry | Per-object | SEC 17a-4, FINRA, HIPAA archival |

### Replication Prerequisites

- Source bucket versioning must be **enabled**
- Destination bucket versioning must be **enabled**
- IAM role with `s3:ReplicateObject`, `s3:GetObjectVersionForReplication`
- Objects uploaded BEFORE replication was enabled are NOT replicated by default

### S3 Select

- Run SQL `SELECT` directly on CSV, JSON, or Parquet objects stored in S3
- Returns only matching data — reduces data transfer by up to 98%
- Cost: $0.002/GB scanned + $0.0007/GB returned

---

## Architecture Diagram

```
  Source Bucket (us-east-1)           Dest Bucket (us-west-2)
  ┌──────────────────────┐            ┌──────────────────────┐
  │  Versioning: ENABLED │  IAM Role  │  Versioning: ENABLED │
  │                      │──────────► │                      │
  │  obj.csv  v1         │  S3 CRR    │  obj.csv  v1 (copy)  │
  │  obj.csv  v2         │  Rule      │  obj.csv  v2 (copy)  │
  │  obj.csv  DELETE     │            │                      │
  │  (marker)            │            │                      │
  └──────────────────────┘            └──────────────────────┘
          │
          │  Object Lock (WORM)
          ▼
  ┌───────────────────────────────┐
  │  Compliance Mode              │
  │  Retain Until: 2029-01-01     │
  │  Cannot delete until expiry   │
  └───────────────────────────────┘
```

---

## PoC Steps

### Step 1 — Set variables

```bash
export AWS_REGION="us-east-1"
export DEST_REGION="us-west-2"
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export BUCKET_NAME="s3-howto-${ACCOUNT_ID}-${AWS_REGION}"
export REPLICA_BUCKET="s3-howto-replica-${ACCOUNT_ID}-${DEST_REGION}"
echo "Source: $BUCKET_NAME | Replica: $REPLICA_BUCKET"
```

---

### Step 2 — Enable versioning on the source bucket

```bash
aws s3api put-bucket-versioning \
  --bucket "$BUCKET_NAME" \
  --versioning-configuration Status=Enabled

# Verify
aws s3api get-bucket-versioning \
  --bucket "$BUCKET_NAME" \
  --query '{Status:Status,MFADelete:MFADelete}' \
  --output table
```

**Expected output:**
```
-------------------------------------
|       GetBucketVersioning         |
+-----------+-------------------------+
|  MFADelete|  None                   |
|  Status   |  Enabled                |
+-----------+-------------------------+
```

---

### Step 3 — Demonstrate versioning: multiple versions of a file

```bash
# Create version 1
echo "Version 1 — initial content" > /tmp/versioned.txt
V1_ID=$(aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/versioned.txt" \
  --body /tmp/versioned.txt \
  --tags "Lab=05" \
  --query VersionId \
  --output text)
echo "Version 1 ID: $V1_ID"

# Create version 2
echo "Version 2 — updated content with new data" > /tmp/versioned.txt
V2_ID=$(aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/versioned.txt" \
  --body /tmp/versioned.txt \
  --tags "Lab=05" \
  --query VersionId \
  --output text)
echo "Version 2 ID: $V2_ID"

# Create version 3
echo "Version 3 — final production data" > /tmp/versioned.txt
V3_ID=$(aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/versioned.txt" \
  --body /tmp/versioned.txt \
  --tags "Lab=05" \
  --query VersionId \
  --output text)
echo "Version 3 ID: $V3_ID"

# List all versions
aws s3api list-object-versions \
  --bucket "$BUCKET_NAME" \
  --prefix "advanced/versioned.txt" \
  --query 'Versions[*].{VersionId:VersionId,IsLatest:IsLatest,LastModified:LastModified}' \
  --output table

# Retrieve a specific old version
aws s3api get-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/versioned.txt" \
  --version-id "$V1_ID" \
  /tmp/v1_retrieved.txt
echo "Retrieved V1 content:" && cat /tmp/v1_retrieved.txt
```

---

### Step 4 — Delete marker demonstration

```bash
# "Delete" the object (creates a delete marker, does NOT remove versions)
aws s3api delete-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/versioned.txt" \
  --query '{DeleteMarker:DeleteMarker,VersionId:VersionId}' \
  --output table

# Trying to GET now returns 404
aws s3api get-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/versioned.txt" \
  /tmp/should_fail.txt 2>&1 || echo "Expected: 404 Not Found (delete marker active)"

# Restore by deleting the delete marker (using its VersionId)
DELETE_MARKER_ID=$(aws s3api list-object-versions \
  --bucket "$BUCKET_NAME" \
  --prefix "advanced/versioned.txt" \
  --query 'DeleteMarkers[0].VersionId' \
  --output text)
echo "Delete marker ID: $DELETE_MARKER_ID"

aws s3api delete-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/versioned.txt" \
  --version-id "$DELETE_MARKER_ID" \
  --query 'DeleteMarker' \
  --output text

# Object is now accessible again
aws s3api get-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/versioned.txt" \
  /tmp/restored.txt && cat /tmp/restored.txt
```

---

### Step 5 — Set up Cross-Region Replication (CRR)

```bash
# Create the replica bucket in us-west-2
aws s3api create-bucket \
  --bucket "$REPLICA_BUCKET" \
  --region "$DEST_REGION" \
  --create-bucket-configuration LocationConstraint=$DEST_REGION \
  --tags Key=Project,Value=S3HowTo Key=Purpose,Value=CRRReplica

# Enable versioning on replica (required for CRR)
aws s3api put-bucket-versioning \
  --bucket "$REPLICA_BUCKET" \
  --versioning-configuration Status=Enabled \
  --region "$DEST_REGION"

# Create IAM role for S3 replication
cat > /tmp/replication-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "s3.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

REPLICATION_ROLE_ARN=$(aws iam create-role \
  --role-name S3HowToReplicationRole \
  --assume-role-policy-document file:///tmp/replication-trust.json \
  --tags Key=Project,Value=S3HowTo \
  --query Role.Arn \
  --output text)
echo "Replication role ARN: $REPLICATION_ROLE_ARN"

# Create and attach the replication permissions policy
cat > /tmp/replication-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetReplicationConfiguration",
        "s3:ListBucket"
      ],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObjectVersionForReplication",
        "s3:GetObjectVersionAcl",
        "s3:GetObjectVersionTagging"
      ],
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "s3:ReplicateObject",
        "s3:ReplicateDelete",
        "s3:ReplicateTags"
      ],
      "Resource": "arn:aws:s3:::${REPLICA_BUCKET}/*"
    }
  ]
}
EOF

aws iam put-role-policy \
  --role-name S3HowToReplicationRole \
  --policy-name S3HowToReplicationPolicy \
  --policy-document file:///tmp/replication-policy.json

# Apply CRR configuration to source bucket
cat > /tmp/crr-config.json << EOF
{
  "Role": "${REPLICATION_ROLE_ARN}",
  "Rules": [{
    "ID": "replicate-advanced-prefix",
    "Status": "Enabled",
    "Filter": {"Prefix": "advanced/"},
    "Destination": {
      "Bucket": "arn:aws:s3:::${REPLICA_BUCKET}",
      "StorageClass": "STANDARD_IA"
    },
    "DeleteMarkerReplication": {"Status": "Enabled"}
  }]
}
EOF

aws s3api put-bucket-replication \
  --bucket "$BUCKET_NAME" \
  --replication-configuration file:///tmp/crr-config.json

echo "CRR configured — new objects under advanced/ will replicate to $REPLICA_BUCKET"

# Upload a new object to trigger replication
echo "CRR test object — $(date -u)" > /tmp/crr_test.txt
aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/crr-test.txt" \
  --body /tmp/crr_test.txt \
  --tags "Lab=05&ReplicationType=CRR"

# Check replication status (may take a few seconds)
sleep 5
aws s3api head-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/crr-test.txt" \
  --query '{ReplicationStatus:ReplicationStatus,StorageClass:StorageClass}' \
  --output table
```

---

### Step 6 — S3 Select: query CSV data in-place

```bash
# Create a CSV file with employee data
cat > /tmp/employees.csv << 'EOF'
id,name,department,salary,hire_date
1,Alice Johnson,Engineering,120000,2019-03-15
2,Bob Smith,Marketing,85000,2020-07-01
3,Carol White,Engineering,135000,2018-01-20
4,Dave Brown,HR,72000,2021-11-05
5,Eve Davis,Engineering,145000,2017-06-10
6,Frank Miller,Marketing,92000,2019-09-22
7,Grace Lee,Finance,98000,2020-02-14
8,Henry Taylor,Engineering,128000,2018-08-30
9,Iris Chen,Finance,105000,2019-04-17
10,Jack Wilson,HR,68000,2022-01-03
EOF

aws s3api put-object \
  --bucket "$BUCKET_NAME" \
  --key "advanced/employees.csv" \
  --body /tmp/employees.csv \
  --tags "Lab=05&Type=CSV"

# Use S3 Select to query only Engineering employees earning > 125000
aws s3api select-object-content \
  --bucket "$BUCKET_NAME" \
  --key "advanced/employees.csv" \
  --expression "SELECT s.name, s.department, s.salary FROM s3object s WHERE s.department = 'Engineering' AND CAST(s.salary AS FLOAT) > 125000" \
  --expression-type SQL \
  --input-serialization '{"CSV": {"FileHeaderInfo": "USE"}}' \
  --output-serialization '{"CSV": {}}' \
  /tmp/s3select-output.csv

echo "S3 Select query results:"
cat /tmp/s3select-output.csv
```

**Expected output:**
```
S3 Select query results:
Carol White,Engineering,135000
Eve Davis,Engineering,145000
Henry Taylor,Engineering,128000
```

---

## Cleanup

```bash
# Remove all versions and delete markers for advanced/ prefix
aws s3api list-object-versions \
  --bucket "$BUCKET_NAME" \
  --prefix "advanced/" \
  --query '{Objects: Versions[*].{Key:Key,VersionId:VersionId}}' \
  --output json | python3 -c "
import sys, json, subprocess
data = json.load(sys.stdin)
if data.get('Objects'):
    subprocess.run(['aws','s3api','delete-objects','--bucket','$(echo $BUCKET_NAME)','--delete',json.dumps(data)], check=True)
print('Versions deleted')
"

# Delete replica bucket
aws s3 rm s3://$REPLICA_BUCKET --recursive
aws s3api delete-bucket --bucket "$REPLICA_BUCKET" --region "$DEST_REGION"

# Remove CRR config
aws s3api delete-bucket-replication --bucket "$BUCKET_NAME"

# Clean up IAM role
aws iam delete-role-policy --role-name S3HowToReplicationRole --policy-name S3HowToReplicationPolicy
aws iam delete-role --role-name S3HowToReplicationRole

echo "Cleanup complete"
```

---

➡️ **Next:** [Lab 06 — Event-Driven Pipeline](../06-event-driven-pipeline/README.md)
