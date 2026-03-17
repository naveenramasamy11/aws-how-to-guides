# 🟢 Lab 02 — Launch Your First RDS Instance

**Difficulty:** 🟢 Beginner
**Time:** ~35 minutes
**Goal:** Create a MySQL 8.0 RDS instance via CLI, wait for it to become available, connect with the MySQL client, create a table, insert rows, and then clean up.

---

## 🎯 Goal

Go from zero to a running, queryable MySQL database on RDS — entirely from the terminal.

---

## 📖 Concepts

### DB Subnet Group

RDS requires a **DB Subnet Group** — a collection of subnets in at least two AZs within your VPC. RDS places instances within these subnets.

### Security Group

The RDS instance needs an **inbound rule** on port 3306 (MySQL) from your IP or from the application's security group.

### Public Accessibility

Setting `--publicly-accessible` assigns the instance a public DNS endpoint. This is fine for lab purposes; production databases should live in private subnets.

### Connection Flow

```
Your Terminal
     │
     │  TCP 3306
     ▼
Security Group (allow 3306 from your IP)
     │
     ▼
RDS Endpoint  (mydb.xxxxxxxxx.us-east-1.rds.amazonaws.com)
     │
     ▼
MySQL 8.0 Engine
```

---

## 🏗️ Architecture

```
Internet
   │
   │ TCP 3306
   ▼
┌─────────────────────────────────────────────┐
│                  VPC (default)               │
│                                              │
│  ┌────────────────┐   ┌──────────────────┐  │
│  │  SG: rds-lab-sg │   │  Subnet Group    │  │
│  │  Inbound 3306   │   │  (subnet-a, b)   │  │
│  └──────┬─────────┘   └──────────────────┘  │
│         │                                    │
│         ▼                                    │
│  ┌──────────────────────────────────────┐   │
│  │  RDS Instance: my-rds-demo           │   │
│  │  MySQL 8.0 / db.t3.micro / gp3 20GB │   │
│  └──────────────────────────────────────┘   │
└─────────────────────────────────────────────┘
```

---

## 🔬 PoC Steps

### Step 1 — Gather environment variables

```bash
export AWS_REGION=us-east-1
export DB_ID=my-rds-demo
export DB_USER=admin
export DB_PASS="MySecurePass123!"   # change this

# Default VPC
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)
echo "VPC: $VPC_ID"

# Two subnets for the DB Subnet Group
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0:2].SubnetId' \
  --output text | tr '\t' ' ')
echo "Subnets: $SUBNET_IDS"
```

**Expected output:**
```
VPC: vpc-0abc1234def56789
Subnets: subnet-0aa111bbb222 subnet-0cc333ddd444
```

### Step 2 — Create a DB Subnet Group

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name rds-lab-subnet-group \
  --db-subnet-group-description "Lab subnet group" \
  --subnet-ids $SUBNET_IDS \
  --tags Key=Project,Value=rds-howto \
  --query 'DBSubnetGroup.DBSubnetGroupName' \
  --output text
```

**Expected output:**
```
rds-lab-subnet-group
```

### Step 3 — Create a Security Group for RDS

```bash
# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name rds-lab-sg \
  --description "RDS lab security group" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=rds-howto}]" \
  --query 'GroupId' --output text)
echo "Security Group: $SG_ID"

# Get your current public IP
MY_IP=$(curl -s https://checkip.amazonaws.com)/32
echo "Your IP CIDR: $MY_IP"

# Allow MySQL inbound from your IP
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 3306 \
  --cidr $MY_IP
```

**Expected output:**
```
Security Group: sg-0abc123def456789
Your IP CIDR: 203.0.113.42/32
```

### Step 4 — Launch the RDS instance

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
  --no-multi-az \
  --publicly-accessible \
  --db-subnet-group-name rds-lab-subnet-group \
  --vpc-security-group-ids $SG_ID \
  --backup-retention-period 1 \
  --tags Key=Project,Value=rds-howto \
  --query 'DBInstance.[DBInstanceIdentifier,DBInstanceStatus]' \
  --output table
```

**Expected output:**
```
------------------------------
|    CreateDBInstance        |
+----------------+-----------+
|  my-rds-demo   | creating  |
+----------------+-----------+
```

### Step 5 — Wait for the instance to become available

```bash
echo "Waiting for RDS instance to become available (5-10 min)..."
aws rds wait db-instance-available \
  --db-instance-identifier $DB_ID
echo "Instance is available!"
```

**Expected output (after ~7 minutes):**
```
Instance is available!
```

### Step 6 — Get the endpoint

```bash
DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier $DB_ID \
  --query 'DBInstances[0].Endpoint.Address' \
  --output text)
echo "Endpoint: $DB_ENDPOINT"
```

**Expected output:**
```
Endpoint: my-rds-demo.cxyz123abc.us-east-1.rds.amazonaws.com
```

### Step 7 — Connect with MySQL client

```bash
# Install mysql client if needed
# Ubuntu/Debian: sudo apt-get install -y mysql-client
# macOS: brew install mysql-client

mysql -h $DB_ENDPOINT -u $DB_USER -p"$DB_PASS" \
  --connect-timeout 10 \
  -e "SELECT VERSION(), NOW();"
```

**Expected output:**
```
+-----------+---------------------+
| VERSION() | NOW()               |
+-----------+---------------------+
| 8.0.36    | 2026-03-17 08:00:12 |
+-----------+---------------------+
```

### Step 8 — Create a database, table, and insert rows

```bash
mysql -h $DB_ENDPOINT -u $DB_USER -p"$DB_PASS" <<'SQL'
CREATE DATABASE IF NOT EXISTS shopdb;
USE shopdb;

CREATE TABLE IF NOT EXISTS products (
  id INT AUTO_INCREMENT PRIMARY KEY,
  name VARCHAR(100) NOT NULL,
  category VARCHAR(50),
  price DECIMAL(10,2),
  created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO products (name, category, price) VALUES
  ('Laptop Pro 15', 'Electronics', 1299.99),
  ('Wireless Mouse', 'Accessories',   29.99),
  ('USB-C Hub 7-port', 'Accessories',  49.99),
  ('Mechanical Keyboard', 'Accessories', 89.99),
  ('4K Monitor 27"', 'Electronics', 399.99);

SELECT id, name, category, price FROM products ORDER BY price DESC;
SQL
```

**Expected output:**
```
+----+---------------------+-------------+---------+
| id | name                | category    | price   |
+----+---------------------+-------------+---------+
|  1 | Laptop Pro 15       | Electronics | 1299.99 |
|  5 | 4K Monitor 27"      | Electronics |  399.99 |
|  4 | Mechanical Keyboard | Accessories |   89.99 |
|  3 | USB-C Hub 7-port    | Accessories |   49.99 |
|  2 | Wireless Mouse      | Accessories |   29.99 |
+----+---------------------+-------------+---------+
```

### Step 9 — Query with Python boto3 + PyMySQL

```python
#!/usr/bin/env python3
"""Connect to RDS MySQL and run a query using PyMySQL."""
import pymysql
import os

DB_ENDPOINT = os.environ["DB_ENDPOINT"]
DB_USER     = os.environ["DB_USER"]
DB_PASS     = os.environ["DB_PASS"]

connection = pymysql.connect(
    host=DB_ENDPOINT,
    user=DB_USER,
    password=DB_PASS,
    database="shopdb",
    connect_timeout=10,
    ssl={"ssl": {}},   # enforce TLS
)

with connection.cursor() as cursor:
    cursor.execute(
        "SELECT category, COUNT(*) AS count, ROUND(SUM(price), 2) AS total "
        "FROM products GROUP BY category ORDER BY total DESC"
    )
    print(f"{'Category':<15} {'Count':>5} {'Total':>10}")
    print("-" * 35)
    for row in cursor.fetchall():
        print(f"{row[0]:<15} {row[1]:>5} {row[2]:>10.2f}")

connection.close()
```

Run:
```bash
export DB_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier my-rds-demo \
  --query 'DBInstances[0].Endpoint.Address' --output text)
export DB_USER=admin
export DB_PASS="MySecurePass123!"
python3 rds_query.py
```

**Expected output:**
```
Category        Count      Total
-----------------------------------
Electronics         2    1699.98
Accessories         3     169.97
```

---

## 🧹 Cleanup

```bash
# Delete RDS instance (skip final snapshot for lab)
aws rds delete-db-instance \
  --db-instance-identifier $DB_ID \
  --skip-final-snapshot

# Wait for deletion
aws rds wait db-instance-deleted \
  --db-instance-identifier $DB_ID
echo "Instance deleted"

# Delete DB subnet group
aws rds delete-db-subnet-group \
  --db-subnet-group-name rds-lab-subnet-group

# Delete security group
aws ec2 delete-security-group --group-id $SG_ID

echo "Cleanup complete"
```

---

## ✅ What You Learned

- How to create a DB subnet group and security group for RDS
- How to launch a MySQL RDS instance via CLI
- How to connect with the MySQL client and run DDL/DML
- How to query RDS from Python using PyMySQL with TLS

---

> ➡️ Next: [Lab 03 — Backups, Snapshots & Parameter Groups](../03-backups-snapshots/README.md)
