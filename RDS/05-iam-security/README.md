# 🔴 Lab 05 — IAM Auth, Encryption & VPC Security

**Difficulty:** 🔴 Advanced
**Time:** ~50 minutes
**Goal:** Harden an RDS deployment with IAM database authentication, KMS encryption at rest, enforced TLS in transit, and private-subnet VPC security with no public internet access.

---

## 🎯 Goal

Build a production-grade secure RDS setup where: all data is encrypted at rest with a customer-managed KMS key, all connections use TLS, database users authenticate via IAM tokens (no passwords), and the instance is in private subnets inaccessible from the internet.

---

## 📖 Concepts

### IAM Database Authentication

Instead of a static password, an IAM principal uses `aws rds generate-db-auth-token` to get a **short-lived token (15-minute TTL)** that is used as the MySQL password. The engine validates the token signature using the RDS public cert.

```
IAM Principal
    │
    ▼  generate-db-auth-token (signed HTTPS request)
AWS RDS Service  ───returns 15-min signed token──▶  Application
    │
    ▼  mysql -h endpoint -p <token>
RDS Engine validates signature
    │
    ▼  Connection granted
```

Requirements:
- DB instance must have `--enable-iam-database-authentication`
- MySQL user created with `IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS'`
- IAM policy with `rds-db:connect` for the specific DB resource ARN

### KMS Encryption at Rest

- All data files, backups, snapshots, logs, and read replicas are encrypted
- Uses **AES-256**
- Key can be AWS-managed (`aws/rds`) or customer-managed (CMK)
- **Encryption is set at creation time — you cannot encrypt an existing unencrypted instance** (must snapshot → copy encrypted → restore)
- KMS CMK rotation is supported (annual automatic or manual)

### TLS in Transit

- RDS serves TLS certificates signed by AWS CA
- Download the global bundle: `https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem`
- Enforce TLS by setting parameter `require_secure_transport=ON`

### VPC Security Architecture

```
Internet Gateway
      │ (blocked for RDS)
      ▼
Public Subnets (EC2 bastion, NAT Gateway)
      │
      │ TCP 3306 via private routing
      ▼
Private Subnets (RDS instances)
      │
      ▼
VPC Endpoint for Secrets Manager (optional, no internet for secret retrieval)
```

---

## 🏗️ Architecture

```
┌───────────────────────────────────────────────────────────────┐
│                         VPC                                    │
│                                                                │
│  ┌──────────────────┐         ┌────────────────────────────┐ │
│  │  Public Subnet    │         │   Private Subnet           │ │
│  │                   │         │                            │ │
│  │  ┌────────────┐  │ 3306     │  ┌──────────────────────┐ │ │
│  │  │ EC2 Bastion │──┼─────────▶│  │  RDS (encrypted,     │ │ │
│  │  │  (IAM role) │  │         │  │  no public IP,       │ │ │
│  │  └────────────┘  │         │  │  IAM auth required)  │ │ │
│  └──────────────────┘         │  └──────────────────────┘ │ │
│                                │                            │ │
│  ┌──────────────────┐         │  ┌──────────────────────┐ │ │
│  │   KMS CMK         │         │  │  VPC Endpoint        │ │ │
│  │   (aws/rds or CMK)│◀────────┤  │  secretsmanager      │ │ │
│  └──────────────────┘         │  └──────────────────────┘ │ │
│                                └────────────────────────────┘ │
└───────────────────────────────────────────────────────────────┘
```

---

## 🔬 PoC Steps

### Step 1 — Create a customer-managed KMS key for RDS

```bash
export AWS_REGION=us-east-1
export DB_ID=my-rds-secure
export DB_USER=admin
export DB_PASS="MySecurePass123!"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

KMS_KEY_ID=$(aws kms create-key \
  --description "RDS lab customer-managed key" \
  --key-usage ENCRYPT_DECRYPT \
  --tags TagKey=Project,TagValue=rds-howto \
  --query 'KeyMetadata.KeyId' --output text)
echo "KMS Key ID: $KMS_KEY_ID"

# Create a friendly alias
aws kms create-alias \
  --alias-name alias/rds-lab-key \
  --target-key-id $KMS_KEY_ID
```

**Expected output:**
```
KMS Key ID: 1a2b3c4d-5e6f-7890-abcd-ef1234567890
```

### Step 2 — Create private subnets for RDS (if not already present)

```bash
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)

# Get the list of all AZs
AZ_LIST=$(aws ec2 describe-availability-zones \
  --query 'AvailabilityZones[0:2].ZoneName' --output text | tr '\t' ' ')
AZ_A=$(echo $AZ_LIST | awk '{print $1}')
AZ_B=$(echo $AZ_LIST | awk '{print $2}')

PRIVATE_SUBNET_A=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 172.31.100.0/24 \
  --availability-zone $AZ_A \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=rds-private-a},{Key=Project,Value=rds-howto}]" \
  --query 'Subnet.SubnetId' --output text)

PRIVATE_SUBNET_B=$(aws ec2 create-subnet \
  --vpc-id $VPC_ID \
  --cidr-block 172.31.101.0/24 \
  --availability-zone $AZ_B \
  --tag-specifications "ResourceType=subnet,Tags=[{Key=Name,Value=rds-private-b},{Key=Project,Value=rds-howto}]" \
  --query 'Subnet.SubnetId' --output text)

echo "Private Subnet A: $PRIVATE_SUBNET_A"
echo "Private Subnet B: $PRIVATE_SUBNET_B"
```

### Step 3 — Create DB Subnet Group with private subnets

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name rds-private-subnet-group \
  --db-subnet-group-description "Private subnets for secure RDS lab" \
  --subnet-ids $PRIVATE_SUBNET_A $PRIVATE_SUBNET_B \
  --tags Key=Project,Value=rds-howto \
  --query 'DBSubnetGroup.DBSubnetGroupName' --output text
```

### Step 4 — Create security groups: RDS + EC2 bastion

```bash
# Security group for RDS — allow inbound 3306 only from bastion SG
RDS_SG=$(aws ec2 create-security-group \
  --group-name rds-secure-sg \
  --description "RDS secure - no internet" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=rds-howto}]" \
  --query 'GroupId' --output text)

# Security group for bastion EC2 host
BASTION_SG=$(aws ec2 create-security-group \
  --group-name rds-bastion-sg \
  --description "Bastion for RDS access" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=rds-howto}]" \
  --query 'GroupId' --output text)

# Allow RDS to accept connections from bastion SG only
aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp --port 3306 \
  --source-group $BASTION_SG

# Allow SSH to bastion from your IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
aws ec2 authorize-security-group-ingress \
  --group-id $BASTION_SG --protocol tcp --port 22 --cidr $MY_IP

echo "RDS SG: $RDS_SG | Bastion SG: $BASTION_SG"
```

### Step 5 — Launch encrypted RDS instance with IAM auth

```bash
aws rds create-db-instance \
  --db-instance-identifier $DB_ID \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.36 \
  --master-username $DB_USER \
  --master-user-password "$DB_PASS" \
  --allocated-storage 20 \
  --storage-type gp3 \
  --kms-key-id $KMS_KEY_ID \
  --storage-encrypted \
  --no-publicly-accessible \
  --enable-iam-database-authentication \
  --db-subnet-group-name rds-private-subnet-group \
  --vpc-security-group-ids $RDS_SG \
  --backup-retention-period 1 \
  --tags Key=Project,Value=rds-howto \
  --query 'DBInstance.[DBInstanceIdentifier,StorageEncrypted,IAMDatabaseAuthenticationEnabled]' \
  --output table

aws rds wait db-instance-available --db-instance-identifier $DB_ID
echo "Encrypted instance ready"
```

**Expected output:**
```
----------------------------------------------------------
|                  CreateDBInstance                      |
+-----------------+------------------+------------------+
|  my-rds-secure  |  True            |  True            |
+-----------------+------------------+------------------+
```

### Step 6 — Create an IAM policy for DB auth

```bash
DB_RESOURCE_ID=$(aws rds describe-db-instances \
  --db-instance-identifier $DB_ID \
  --query 'DBInstances[0].DbiResourceId' --output text)
echo "DB Resource ID: $DB_RESOURCE_ID"

cat > /tmp/rds-iam-auth-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["rds-db:connect"],
    "Resource": "arn:aws:rds-db:${AWS_REGION}:${ACCOUNT_ID}:dbuser:${DB_RESOURCE_ID}/iam_user"
  }]
}
EOF

aws iam create-policy \
  --policy-name rds-iam-auth-policy \
  --policy-document file:///tmp/rds-iam-auth-policy.json \
  --query 'Policy.Arn' --output text
```

### Step 7 — Create MySQL IAM user via master login

```bash
DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier $DB_ID \
  --query 'DBInstances[0].Endpoint.Address' --output text)

# Note: from a bastion with network access to private subnet, run:
# Shown here as the SQL you would execute
cat <<'ENDSQL'
-- Connect with master user first (password auth)
-- mysql -h $DB_ENDPOINT -u admin -p"$DB_PASS" --ssl-ca=global-bundle.pem

CREATE USER IF NOT EXISTS 'iam_user'@'%' IDENTIFIED WITH AWSAuthenticationPlugin AS 'RDS';
GRANT SELECT, INSERT, UPDATE, DELETE ON *.* TO 'iam_user'@'%';
FLUSH PRIVILEGES;
ENDSQL
echo "SQL above creates the IAM-authenticated MySQL user"
```

### Step 8 — Generate IAM auth token and connect

```python
#!/usr/bin/env python3
"""Generate an IAM DB auth token and use it to connect to RDS."""
import boto3
import pymysql
import os

region      = "us-east-1"
db_endpoint = os.environ["DB_ENDPOINT"]
db_user     = "iam_user"
db_port     = 3306

# Generate 15-minute auth token
client = boto3.client("rds", region_name=region)
token = client.generate_db_auth_token(
    DBHostname=db_endpoint,
    Port=db_port,
    DBUsername=db_user,
    Region=region,
)
print(f"Token (first 60 chars): {token[:60]}...")

# Download RDS CA bundle (run once):
# curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem

connection = pymysql.connect(
    host=db_endpoint,
    user=db_user,
    password=token,
    port=db_port,
    ssl_ca="global-bundle.pem",
    ssl_verify_cert=True,
    connect_timeout=10,
)
with connection.cursor() as cur:
    cur.execute("SELECT CURRENT_USER(), @@ssl_cipher;")
    row = cur.fetchone()
    print(f"Connected as: {row[0]}")
    print(f"TLS cipher:   {row[1]}")
connection.close()
```

Run:
```bash
curl -o global-bundle.pem https://truststore.pki.rds.amazonaws.com/global/global-bundle.pem
export DB_ENDPOINT=<your-rds-endpoint>
python3 rds_iam_auth.py
```

**Expected output:**
```
Token (first 60 chars): my-rds-secure.xxx.us-east-1.rds.amazonaws.com:3306/?Action=c...
Connected as: iam_user@172.31.100.10
TLS cipher:   TLS_AES_256_GCM_SHA384
```

### Step 9 — Enforce TLS via parameter group

```bash
aws rds create-db-parameter-group \
  --db-parameter-group-name rds-tls-enforce-pg \
  --db-parameter-group-family mysql8.0 \
  --description "Enforce TLS for all MySQL connections" \
  --tags Key=Project,Value=rds-howto

aws rds modify-db-parameter-group \
  --db-parameter-group-name rds-tls-enforce-pg \
  --parameters "ParameterName=require_secure_transport,ParameterValue=ON,ApplyMethod=immediate"

aws rds modify-db-instance \
  --db-instance-identifier $DB_ID \
  --db-parameter-group-name rds-tls-enforce-pg \
  --apply-immediately
```

### Step 10 — Verify encryption status

```bash
aws rds describe-db-instances \
  --db-instance-identifier $DB_ID \
  --query 'DBInstances[0].{Encrypted:StorageEncrypted,KmsKeyId:KmsKeyId,IAMAuth:IAMDatabaseAuthenticationEnabled,PubliclyAccessible:PubliclyAccessible}' \
  --output table
```

**Expected output:**
```
-----------------------------------------------------------------------------------------------
|                              DescribeDBInstances                                            |
+---------------------------------------+----------+----------+-----------------------------+
|  Encrypted                            | True     |          |                             |
|  IAMAuth                              | True     |          |                             |
|  KmsKeyId                             | arn:aws:kms:us-east-1:123456789012:key/1a2b3c4d...  |
|  PubliclyAccessible                   | False    |          |                             |
+---------------------------------------+----------+----------+-----------------------------+
```

---

## 🧹 Cleanup

```bash
aws rds delete-db-instance --db-instance-identifier $DB_ID --skip-final-snapshot
aws rds wait db-instance-deleted --db-instance-identifier $DB_ID

aws rds delete-db-subnet-group --db-subnet-group-name rds-private-subnet-group
aws ec2 delete-security-group --group-id $RDS_SG
aws ec2 delete-security-group --group-id $BASTION_SG
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_A
aws ec2 delete-subnet --subnet-id $PRIVATE_SUBNET_B

aws rds delete-db-parameter-group --db-parameter-group-name rds-tls-enforce-pg

# Schedule KMS key for deletion (7-day minimum)
aws kms schedule-key-deletion --key-id $KMS_KEY_ID --pending-window-in-days 7
aws kms delete-alias --alias-name alias/rds-lab-key

IAM_POLICY_ARN=$(aws iam list-policies --query "Policies[?PolicyName=='rds-iam-auth-policy'].Arn" --output text)
aws iam delete-policy --policy-arn $IAM_POLICY_ARN

echo "Cleanup complete"
```

---

## ✅ What You Learned

- How to create a KMS CMK and use it to encrypt an RDS instance at rest
- How IAM database authentication works and how to set it up end-to-end
- How to enforce TLS via parameter group (`require_secure_transport`)
- How to place RDS in private subnets with no internet exposure

---

> ➡️ Next: [Lab 06 — Lambda → RDS Integration Pipeline](../06-lambda-integration/README.md)
