# Lab 05 — Cross-Account Access with IAM Roles 🔴

## 🎯 Goal
Enable a user or role in **Account A** (the "source" account) to assume an IAM role in **Account B** (the "target" account) and access S3 buckets — without sharing long-term credentials.

---

## 🧠 Concepts

### Why Cross-Account Roles?
Organizations typically have multiple AWS accounts (dev, staging, prod, shared-services, security). Cross-account IAM roles allow:
- **Centralized identity** — manage users in one account, grant access to others
- **Temporary credentials** — no long-term keys in target accounts
- **Audit trail** — CloudTrail records who assumed what role and when

### Trust Policy vs Permission Policy
| | Trust Policy | Permission Policy |
|--|-------------|------------------|
| **Purpose** | WHO can assume the role | WHAT the role can do |
| **Location** | On the role (in Account B) | On the role (in Account B) |
| **Configured by** | Account B admin | Account B admin |
| **Called by** | Account A users/roles | N/A |

### ExternalId
A secret token that Account A must provide when calling `sts:AssumeRole`. Used to prevent the **confused deputy problem** — where a malicious party tricks a trusted third party into assuming a role on their behalf.

### sts:AssumeRole Flow
```
Account A (111111111111)              Account B (222222222222)
       │                                        │
  IAM User/Role                          IAM Role: CrossAccountRole
       │                                        │
       │  sts:AssumeRole ──────────────────────►│
       │  (with ExternalId if required)         │
       │                                        │
       │◄──────────── Temporary Creds ──────────│
       │  (AccessKeyId, SecretKey, Token)       │
       │                                        │
       │  API Calls using temp creds  ─────────►│
       │                              (S3, etc.) │
```

---

## 🏗️ Architecture

```
Account A (Dev)  ───────────────────────────────────────────
  IAM User: ops-engineer
  IAM Policy: AllowAssumeS3ReaderInProd
    └── sts:AssumeRole → arn:aws:iam::AccountB:role/ProdS3Reader

Account B (Prod) ───────────────────────────────────────────
  IAM Role: ProdS3Reader
    ├── Trust Policy: Allow Account A user to assume
    └── Permission Policy: AmazonS3ReadOnlyAccess
```

---

## 🧪 PoC

> ⚠️ **Prerequisites:** You need two AWS accounts (or two separate CLI profiles).
>
> ```bash
> # Setup two profiles
> aws configure --profile account-a  # Source account
> aws configure --profile account-b  # Target account
> ```

### Step 1: Get Account IDs
```bash
ACCOUNT_A=$(aws sts get-caller-identity --profile account-a --query Account --output text)
ACCOUNT_B=$(aws sts get-caller-identity --profile account-b --query Account --output text)

echo "Account A (Source): $ACCOUNT_A"
echo "Account B (Target): $ACCOUNT_B"
```

### Step 2: [Account B] Create the Cross-Account Role
```bash
# This is run in Account B (target account)
# External ID adds extra security (use a UUID in production)
EXTERNAL_ID="xaccount-poc-$(date +%s)"
echo "ExternalId: $EXTERNAL_ID"  # Save this securely!

cat > /tmp/cross-account-trust.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::${ACCOUNT_A}:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "${EXTERNAL_ID}"
        }
      }
    }
  ]
}
EOF

# Create the role in Account B
ROLE_ARN_B=$(aws iam create-role \
  --profile account-b \
  --role-name ProdS3Reader \
  --assume-role-policy-document file:///tmp/cross-account-trust.json \
  --description "Cross-account role: allows Account A to read S3 in Account B" \
  --max-session-duration 3600 \
  --query 'Role.Arn' \
  --output text)

echo "Created cross-account role: $ROLE_ARN_B ✅"

# Attach S3 read permissions
aws iam attach-role-policy \
  --profile account-b \
  --role-name ProdS3Reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

echo "S3 ReadOnly policy attached to ProdS3Reader ✅"
```

### Step 3: [Account B] Optionally scope to specific S3 buckets
```bash
# More restrictive: only allow access to specific bucket
cat > /tmp/specific-bucket-policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowSpecificBucketAccess",
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::prod-data-bucket",
        "arn:aws:s3:::prod-data-bucket/*"
      ]
    },
    {
      "Sid": "AllowListBuckets",
      "Effect": "Allow",
      "Action": "s3:ListAllMyBuckets",
      "Resource": "*"
    }
  ]
}
EOF

ACCOUNT_ID_B=$(aws sts get-caller-identity --profile account-b --query Account --output text)

SPECIFIC_POLICY_ARN=$(aws iam create-policy \
  --profile account-b \
  --policy-name ProdSpecificBucketAccess \
  --policy-document file:///tmp/specific-bucket-policy.json \
  --query 'Policy.Arn' \
  --output text)

echo "Created specific bucket policy: $SPECIFIC_POLICY_ARN ✅"
```

### Step 4: [Account A] Grant permission to assume the role
```bash
# In Account A, grant ops-engineer user permission to assume the cross-account role
cat > /tmp/allow-assume-role.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowAssumeS3ReaderInProd",
      "Effect": "Allow",
      "Action": "sts:AssumeRole",
      "Resource": "${ROLE_ARN_B}",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "${EXTERNAL_ID}"
        }
      }
    }
  ]
}
EOF

# Create policy in Account A
ASSUME_POLICY_ARN=$(aws iam create-policy \
  --profile account-a \
  --policy-name AllowAssumeS3ReaderInProd \
  --policy-document file:///tmp/allow-assume-role.json \
  --query 'Policy.Arn' \
  --output text)

echo "Created assume-role policy in Account A: $ASSUME_POLICY_ARN ✅"

# Create the user in Account A (or attach to existing user)
aws iam create-user --profile account-a --user-name ops-engineer

# Attach the assume-role policy
aws iam attach-user-policy \
  --profile account-a \
  --user-name ops-engineer \
  --policy-arn $ASSUME_POLICY_ARN

echo "Policy attached to ops-engineer ✅"
```

### Step 5: [Account A] Assume the cross-account role
```bash
# Assume the role in Account B and get temporary credentials
TEMP_CREDS=$(aws sts assume-role \
  --profile account-a \
  --role-arn $ROLE_ARN_B \
  --role-session-name "ops-engineer-prod-access" \
  --external-id $EXTERNAL_ID \
  --duration-seconds 3600)

echo "Temporary credentials obtained ✅"
echo "$TEMP_CREDS" | python3 -m json.tool

# Export credentials as environment variables
export AWS_ACCESS_KEY_ID=$(echo $TEMP_CREDS | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['Credentials']['AccessKeyId'])")
export AWS_SECRET_ACCESS_KEY=$(echo $TEMP_CREDS | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['Credentials']['SecretAccessKey'])")
export AWS_SESSION_TOKEN=$(echo $TEMP_CREDS | python3 -c "import sys,json; d=json.load(sys.stdin); print(d['Credentials']['SessionToken'])")

echo "Environment variables set ✅"
```

### Step 6: Verify you're now operating in Account B
```bash
# With the exported temp creds, this should show Account B identity
aws sts get-caller-identity

# Expected output shows Account B's account ID and the role ARN
# {
#   "UserId": "AROA...:ops-engineer-prod-access",
#   "Account": "222222222222",           ← Account B!
#   "Arn": "arn:aws:sts::222222222222:assumed-role/ProdS3Reader/ops-engineer-prod-access"
# }
```

### Step 7: List S3 buckets in Account B
```bash
# Should work — role has S3 read access
aws s3 ls

# Should fail — role only has S3 permissions
aws ec2 describe-instances --region us-east-1
# Error: not authorized to perform ec2:DescribeInstances
```

### Step 8: Use named profile for cross-account (preferred in practice)
```bash
# Add to ~/.aws/config
cat >> ~/.aws/config << EOF

[profile prod-reader]
role_arn = ${ROLE_ARN_B}
external_id = ${EXTERNAL_ID}
source_profile = account-a
role_session_name = cross-account-session
EOF

# Now simply use the profile
aws s3 ls --profile prod-reader
aws sts get-caller-identity --profile prod-reader
```

---

## 🔎 Audit: See Who Assumed the Role
```bash
# In Account B, check CloudTrail for AssumeRole events
aws cloudtrail lookup-events \
  --profile account-b \
  --lookup-attributes AttributeKey=EventName,AttributeValue=AssumeRole \
  --start-time $(date -u -d '1 hour ago' +%Y-%m-%dT%H:%M:%SZ) \
  --query 'Events[*].{Time:EventTime,User:Username,Event:EventName}' \
  --output table
```

---

## ✅ Cleanup

```bash
# Account A cleanup
aws iam detach-user-policy \
  --profile account-a \
  --user-name ops-engineer \
  --policy-arn $ASSUME_POLICY_ARN

aws iam delete-user --profile account-a --user-name ops-engineer
aws iam delete-policy --profile account-a --policy-arn $ASSUME_POLICY_ARN

# Account B cleanup
aws iam detach-role-policy \
  --profile account-b \
  --role-name ProdS3Reader \
  --policy-arn arn:aws:iam::aws:policy/AmazonS3ReadOnlyAccess

aws iam delete-role --profile account-b --role-name ProdS3Reader

# Unset temp creds
unset AWS_ACCESS_KEY_ID AWS_SECRET_ACCESS_KEY AWS_SESSION_TOKEN

echo "All resources cleaned up ✅"
```

---

## 🔒 Security Best Practices
1. **Always use ExternalId** for third-party/partner cross-account access
2. **Scope trust policy** to specific users/roles, not `root` of the entire account
3. **Set max session duration** to the minimum needed (900s to 43200s)
4. **Enable CloudTrail** in Account B to log all AssumeRole calls
5. **Use AWS Organizations SCPs** as an additional guardrail
6. **Name your sessions** (`--role-session-name`) for better auditability

---

## ➡️ Next Lab
[Lab 06 — Permission Boundaries →](../06-permission-boundaries/README.md)
