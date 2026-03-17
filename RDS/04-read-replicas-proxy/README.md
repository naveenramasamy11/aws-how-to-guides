# 🟡 Lab 04 — Read Replicas & RDS Proxy

**Difficulty:** 🟡 Intermediate
**Time:** ~45 minutes
**Goal:** Create a read replica to scale read traffic, then deploy RDS Proxy to pool connections and reduce database load from serverless/high-concurrency applications.

---

## 🎯 Goal

Scale an RDS MySQL deployment horizontally with a read replica and eliminate connection exhaustion problems with RDS Proxy.

---

## 📖 Concepts

### Read Replicas

Read replicas use **asynchronous replication** from the primary. They are readable, but not writable (except Aurora). Use them to:
- Offload reporting / analytics queries
- Serve read-heavy application tiers
- Promote to standalone DB for disaster recovery

| Feature | Read Replica | Multi-AZ Standby |
|---------|-------------|-----------------|
| Replication | Async | Sync |
| Readable | ✅ Yes | ❌ No |
| Automatic failover | ❌ No | ✅ Yes |
| Purpose | Scale reads | High availability |
| Cross-region | ✅ Yes | ❌ No |

### RDS Proxy

Lambda functions and containers can open thousands of simultaneous database connections — far more than RDS can handle. **RDS Proxy** sits between the application and RDS, multiplexing many application connections onto a small pool of real database connections.

| Feature | Without Proxy | With Proxy |
|---------|--------------|------------|
| Connection management | Application | Proxy (pooling) |
| Failover time | ~60 sec | ~30 sec (preserves connections) |
| IAM Auth | Optional | Enforced (recommended) |
| Secrets | In application | AWS Secrets Manager |
| Connection limits | Can exhaust RDS | Pooled, controlled |

### Architecture Flow

```
Lambda (1000 invocations)
    │  1000 connections
    ▼
RDS Proxy (proxy endpoint)
    │  ~10 multiplexed connections
    ▼
RDS Primary  ──async──▶  RDS Read Replica
    (writes)                (reads)
```

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────────────┐
│                          VPC                                      │
│                                                                   │
│  ┌──────────────┐     ┌──────────────────────────────────────┐  │
│  │  Application  │     │         RDS Proxy                    │  │
│  │  (Lambda/ECS) │────▶│  rds-lab-proxy.proxy-xxx.rds.       │  │
│  └──────────────┘     │  amazonaws.com:3306                  │  │
│                        └───────────────┬──────────────────────┘  │
│                                        │ pooled connections       │
│                                        ▼                          │
│                        ┌──────────────────────────────────────┐  │
│                        │  RDS Primary (my-rds-demo)            │  │
│                        │  us-east-1a                           │  │
│                        └───────────────┬──────────────────────┘  │
│                                        │ async replication        │
│                                        ▼                          │
│                        ┌──────────────────────────────────────┐  │
│                        │  Read Replica (my-rds-demo-replica)  │  │
│                        │  us-east-1b                          │  │
│                        └──────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🔬 PoC Steps

### Step 1 — Launch primary instance with backup enabled

```bash
export AWS_REGION=us-east-1
export DB_ID=my-rds-demo
export DB_USER=admin
export DB_PASS="MySecurePass123!"

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0:2].SubnetId' --output text | tr '\t' ' ')

aws rds create-db-subnet-group \
  --db-subnet-group-name rds-lab-subnet-group \
  --db-subnet-group-description "Lab subnet group" \
  --subnet-ids $SUBNET_IDS \
  --tags Key=Project,Value=rds-howto 2>/dev/null || true

SG_ID=$(aws ec2 describe-security-groups \
  --filters "Name=group-name,Values=rds-lab-sg" "Name=vpc-id,Values=$VPC_ID" \
  --query 'SecurityGroups[0].GroupId' --output text 2>/dev/null)

if [ "$SG_ID" = "None" ] || [ -z "$SG_ID" ]; then
  SG_ID=$(aws ec2 create-security-group \
    --group-name rds-lab-sg --description "RDS lab SG" \
    --vpc-id $VPC_ID \
    --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=rds-howto}]" \
    --query 'GroupId' --output text)
  MY_IP=$(curl -s https://checkip.amazonaws.com)/32
  aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID --protocol tcp --port 3306 --cidr $MY_IP
fi

aws rds create-db-instance \
  --db-instance-identifier $DB_ID \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.36 \
  --master-username $DB_USER \
  --master-user-password "$DB_PASS" \
  --allocated-storage 20 \
  --storage-type gp3 \
  --no-multi-az \
  --publicly-accessible \
  --db-subnet-group-name rds-lab-subnet-group \
  --vpc-security-group-ids $SG_ID \
  --backup-retention-period 1 \
  --tags Key=Project,Value=rds-howto 2>/dev/null || echo "Instance may already exist"

aws rds wait db-instance-available --db-instance-identifier $DB_ID
echo "Primary ready"
```

### Step 2 — Create a read replica

```bash
aws rds create-db-instance-read-replica \
  --db-instance-identifier ${DB_ID}-replica \
  --source-db-instance-identifier $DB_ID \
  --db-instance-class db.t3.micro \
  --publicly-accessible \
  --tags Key=Project,Value=rds-howto \
  --query 'DBInstance.[DBInstanceIdentifier,DBInstanceStatus]' \
  --output table
```

**Expected output:**
```
---------------------------------------------
|   CreateDBInstanceReadReplica             |
+-----------------------------+-------------+
|  my-rds-demo-replica        | creating    |
+-----------------------------+-------------+
```

```bash
echo "Waiting for replica (5-10 min)..."
aws rds wait db-instance-available \
  --db-instance-identifier ${DB_ID}-replica
echo "Replica ready"
```

### Step 3 — Get replica endpoint and verify read-only

```bash
REPLICA_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier ${DB_ID}-replica \
  --query 'DBInstances[0].Endpoint.Address' --output text)
echo "Replica: $REPLICA_ENDPOINT"

# Test connectivity (read-only)
mysql -h $REPLICA_ENDPOINT -u $DB_USER -p"$DB_PASS" \
  -e "SHOW VARIABLES LIKE 'read_only';"
```

**Expected output:**
```
+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| read_only     | ON    |
+---------------+-------+
```

### Step 4 — Measure replication lag

```bash
mysql -h $REPLICA_ENDPOINT -u $DB_USER -p"$DB_PASS" \
  -e "SHOW SLAVE STATUS\G" 2>/dev/null | grep Seconds_Behind_Master
```

**Expected output:**
```
 Seconds_Behind_Master: 0
```

### Step 5 — Store DB credentials in Secrets Manager (required for RDS Proxy)

```bash
SECRET_ARN=$(aws secretsmanager create-secret \
  --name rds-lab-db-secret \
  --description "RDS lab MySQL credentials" \
  --secret-string "{\"username\":\"${DB_USER}\",\"password\":\"${DB_PASS}\"}" \
  --tags Key=Project,Value=rds-howto \
  --query 'ARN' --output text)
echo "Secret ARN: $SECRET_ARN"
```

**Expected output:**
```
Secret ARN: arn:aws:secretsmanager:us-east-1:123456789012:secret:rds-lab-db-secret-AbCdEf
```

### Step 6 — Create IAM role for RDS Proxy

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Trust policy
cat > /tmp/rds-proxy-trust.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"Service": "rds.amazonaws.com"},
    "Action": "sts:AssumeRole"
  }]
}
EOF

PROXY_ROLE_ARN=$(aws iam create-role \
  --role-name rds-proxy-role \
  --assume-role-policy-document file:///tmp/rds-proxy-trust.json \
  --tags Key=Project,Value=rds-howto \
  --query 'Role.Arn' --output text)
echo "Proxy Role ARN: $PROXY_ROLE_ARN"

# Allow the role to read the secret
cat > /tmp/rds-proxy-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Action": ["secretsmanager:GetSecretValue"],
    "Resource": "$SECRET_ARN"
  },{
    "Effect": "Allow",
    "Action": ["kms:Decrypt"],
    "Resource": "*",
    "Condition": {"StringEquals": {"kms:ViaService": "secretsmanager.us-east-1.amazonaws.com"}}
  }]
}
EOF

aws iam put-role-policy \
  --role-name rds-proxy-role \
  --policy-name rds-proxy-secrets-policy \
  --policy-document file:///tmp/rds-proxy-policy.json
```

### Step 7 — Get subnet IDs for proxy

```bash
SUBNET_ID_LIST=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0:2].SubnetId' \
  --output json | python3 -c "import sys,json; print(json.dumps(json.load(sys.stdin)))")
echo "Subnets: $SUBNET_ID_LIST"
```

### Step 8 — Create RDS Proxy

```bash
aws rds create-db-proxy \
  --db-proxy-name rds-lab-proxy \
  --engine-family MYSQL \
  --auth "[{\"AuthScheme\":\"SECRETS\",\"SecretArn\":\"${SECRET_ARN}\",\"IAMAuth\":\"DISABLED\"}]" \
  --role-arn $PROXY_ROLE_ARN \
  --vpc-subnet-ids $(aws ec2 describe-subnets \
    --filters "Name=vpc-id,Values=$VPC_ID" \
    --query 'Subnets[0:2].SubnetId' \
    --output text | tr '\t' ' ') \
  --vpc-security-group-ids $SG_ID \
  --require-tls \
  --tags Key=Project,Value=rds-howto \
  --query 'DBProxy.[DBProxyName,Status]' \
  --output table
```

**Expected output:**
```
---------------------------------------
|          CreateDBProxy              |
+----------------+--------------------+
|  rds-lab-proxy | creating           |
+----------------+--------------------+
```

### Step 9 — Register proxy target (point at RDS instance)

```bash
# Wait for proxy to become available (~3-5 minutes)
aws rds wait db-proxy-available \
  --db-proxy-name rds-lab-proxy 2>/dev/null || \
  echo "Wait manually: aws rds describe-db-proxies --db-proxy-name rds-lab-proxy"

TARGET_GROUP_NAME=$(aws rds describe-db-proxy-target-groups \
  --db-proxy-name rds-lab-proxy \
  --query 'TargetGroups[0].TargetGroupName' --output text)

aws rds register-db-proxy-targets \
  --db-proxy-name rds-lab-proxy \
  --target-group-name $TARGET_GROUP_NAME \
  --db-instance-identifiers $DB_ID \
  --query 'DBProxyTargets[*].[Endpoint,TargetHealth.State]' \
  --output table
```

**Expected output:**
```
-------------------------------------------------------------
|              RegisterDBProxyTargets                       |
+---------------------------------------------+------------+
|  my-rds-demo.xxx.us-east-1.rds.amazonaws.com | AVAILABLE |
+---------------------------------------------+------------+
```

### Step 10 — Get proxy endpoint and connect

```bash
PROXY_ENDPOINT=$(aws rds describe-db-proxies \
  --db-proxy-name rds-lab-proxy \
  --query 'DBProxies[0].Endpoint' --output text)
echo "Proxy Endpoint: $PROXY_ENDPOINT"

# Connect via proxy (same as connecting to RDS directly)
mysql -h $PROXY_ENDPOINT -u $DB_USER -p"$DB_PASS" \
  --ssl-mode=REQUIRED \
  -e "SELECT @@hostname, @@version;"
```

**Expected output:**
```
+-------------------------------+-----------+
| @@hostname                    | @@version |
+-------------------------------+-----------+
| ip-172-31-5-123.ec2.internal  | 8.0.36    |
+-------------------------------+-----------+
```

---

## 🧹 Cleanup

```bash
# Deregister proxy targets and delete proxy
aws rds deregister-db-proxy-targets \
  --db-proxy-name rds-lab-proxy \
  --target-group-name default \
  --db-instance-identifiers $DB_ID 2>/dev/null || true

aws rds delete-db-proxy --db-proxy-name rds-lab-proxy 2>/dev/null || true

# Delete secret
aws secretsmanager delete-secret --secret-id rds-lab-db-secret --force-delete-without-recovery

# Delete IAM role
aws iam delete-role-policy --role-name rds-proxy-role --policy-name rds-proxy-secrets-policy
aws iam delete-role --role-name rds-proxy-role

# Delete replica then primary
aws rds delete-db-instance --db-instance-identifier ${DB_ID}-replica --skip-final-snapshot
aws rds wait db-instance-deleted --db-instance-identifier ${DB_ID}-replica
aws rds delete-db-instance --db-instance-identifier $DB_ID --skip-final-snapshot
aws rds wait db-instance-deleted --db-instance-identifier $DB_ID

aws rds delete-db-subnet-group --db-subnet-group-name rds-lab-subnet-group
aws ec2 delete-security-group --group-id $SG_ID
echo "Cleanup complete"
```

---

## ✅ What You Learned

- How to create a read replica and verify it is read-only
- How to measure replication lag with `SHOW SLAVE STATUS`
- How to configure RDS Proxy with Secrets Manager for connection pooling
- How proxy endpoints work as drop-in replacements for direct RDS endpoints

---

> ➡️ Next: [Lab 05 — IAM Auth, Encryption & VPC Security](../05-iam-security/README.md)
