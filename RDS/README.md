# 🗄️ Amazon RDS — How-To Guide

Amazon Relational Database Service (RDS) makes it easy to set up, operate, and scale a relational database in the cloud. This module walks you from first launch to production-hardened deployments.

---

## 📚 Learning Path

| # | Lab | Difficulty | What You'll Build |
|---|-----|------------|-------------------|
| 01 | [RDS Overview](./01-overview/README.md) | 🟢 Beginner | Understand RDS concepts, engine options, pricing |
| 02 | [Launch Your First RDS Instance](./02-first-instance/README.md) | 🟢 Beginner | MySQL RDS instance via CLI, connect & query |
| 03 | [Backups, Snapshots & Parameter Groups](./03-backups-snapshots/README.md) | 🟡 Intermediate | Automated backups, manual snapshots, custom params |
| 04 | [Read Replicas & RDS Proxy](./04-read-replicas-proxy/README.md) | 🟡 Intermediate | Scale reads, connection pooling with RDS Proxy |
| 05 | [IAM Auth, Encryption & VPC Security](./05-iam-security/README.md) | 🔴 Advanced | Least-privilege IAM DB auth, TLS, KMS encryption |
| 06 | [Lambda → RDS Integration Pipeline](./06-lambda-integration/README.md) | 🔴 Advanced | Serverless app writing to RDS via RDS Proxy |
| 07 | [Production Patterns](./07-production-patterns/README.md) | 🔴 Advanced | CloudWatch alarms, cost tips, maintenance, failover |

---

## 🧠 Core Concepts Quick-Reference

| Term | Meaning |
|------|---------|
| **DB Instance** | The compute/storage unit running your database engine |
| **Multi-AZ** | Synchronous standby replica for automatic failover |
| **Read Replica** | Asynchronous replica for read scale-out (can be cross-region) |
| **RDS Proxy** | Managed connection pooler sitting between app and RDS |
| **Parameter Group** | Collection of engine configuration settings |
| **Option Group** | Additional database features (e.g., TDE for Oracle) |
| **Subnet Group** | Set of subnets across AZs in which RDS can place instances |
| **Automated Backup** | Daily snapshot + transaction logs; enables point-in-time restore |
| **Snapshot** | Manual user-initiated backup; persists beyond instance deletion |
| **Enhanced Monitoring** | OS-level metrics at up to 1-second granularity |
| **Performance Insights** | Query-level performance analysis dashboard |

---

## 🏗️ Architecture Overview

```
                         ┌─────────────────────────────────────────┐
                         │               VPC                        │
  ┌──────────┐           │  ┌─────────────┐    ┌─────────────────┐ │
  │  App /   │  IAM Auth │  │  RDS Proxy  │───▶│  RDS Primary    │ │
  │ Lambda   │──────────▶│  │ (us-east-1a)│    │  (us-east-1a)   │ │
  └──────────┘           │  └─────────────┘    └───────┬─────────┘ │
                         │                             │ Sync       │
  ┌──────────┐           │  ┌──────────────────┐  ┌───▼─────────┐  │
  │CloudWatch│◀──metrics─│  │  Read Replica    │  │ Standby     │  │
  │ Alarms   │           │  │  (us-east-1b)    │  │ (us-east-1b)│  │
  └──────────┘           │  └──────────────────┘  └─────────────┘  │
                         └─────────────────────────────────────────┘
                                         │
                              ┌──────────▼──────────┐
                              │   S3 (snapshots /   │
                              │   automated backups) │
                              └─────────────────────┘
```

---

## ⚙️ One-Time Setup

```bash
# Install AWS CLI v2 (if not present)
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configure credentials
aws configure
# AWS Access Key ID: <your-key>
# AWS Secret Access Key: <your-secret>
# Default region name: us-east-1
# Default output format: table

# Set a convenience variable
export AWS_REGION=us-east-1
export DB_ID=my-rds-demo
```

---

## 🔧 CLI Cheat Sheet

```bash
# List all RDS instances
aws rds describe-db-instances \
  --query 'DBInstances[*].[DBInstanceIdentifier,DBInstanceStatus,Engine,DBInstanceClass]' \
  --output table

# Create a MySQL instance (free-tier eligible)
aws rds create-db-instance \
  --db-instance-identifier my-rds-demo \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0 \
  --master-username admin \
  --master-user-password MySecurePass123! \
  --allocated-storage 20 \
  --no-multi-az \
  --publicly-accessible \
  --tags Key=Project,Value=rds-howto

# Describe instance status
aws rds describe-db-instances \
  --db-instance-identifier my-rds-demo \
  --query 'DBInstances[0].[DBInstanceStatus,Endpoint.Address]' \
  --output table

# Create a snapshot
aws rds create-db-snapshot \
  --db-instance-identifier my-rds-demo \
  --db-snapshot-identifier my-rds-demo-snap-$(date +%F) \
  --tags Key=Project,Value=rds-howto

# Delete instance (no final snapshot for lab)
aws rds delete-db-instance \
  --db-instance-identifier my-rds-demo \
  --skip-final-snapshot
```

---

## 📦 Prerequisites

- AWS account with permissions: `rds:*`, `ec2:*`, `iam:PassRole`, `kms:*`
- AWS CLI v2 configured
- MySQL client: `sudo apt install mysql-client` or `brew install mysql-client`
- Python 3.8+ with `boto3` and `PyMySQL`: `pip install boto3 pymysql`

---

> ➡️ Start here: [Lab 01 — RDS Overview](./01-overview/README.md)
