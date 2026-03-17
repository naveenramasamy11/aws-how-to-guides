# 🟢 Lab 01 — Amazon RDS Overview

**Difficulty:** 🟢 Beginner
**Time:** ~20 minutes
**Goal:** Understand what RDS is, when to use it, how it's priced, and how its components fit together — before you create a single resource.

---

## 🎯 Goal

Get a solid conceptual foundation of Amazon RDS so every subsequent lab makes sense. You will explore the AWS console and CLI to survey available engines, instance classes, and region availability — without incurring charges.

---

## 📖 Concepts

### What is Amazon RDS?

Amazon RDS is a **managed relational database service** that handles:

- Hardware provisioning and OS patching
- Database engine installation and upgrades
- Automated backups and point-in-time recovery
- Multi-AZ synchronous replication for high availability
- Storage autoscaling

You keep full SQL access — RDS just removes the undifferentiated heavy lifting.

### Supported Engines

| Engine | Best For | Notable Feature |
|--------|----------|-----------------|
| **MySQL 8.0** | Web apps, WordPress | Widely compatible |
| **PostgreSQL 16** | Complex queries, JSON | Full ACID, extensions |
| **MariaDB 10.11** | MySQL-compatible OSS | Aria engine, Galera |
| **Oracle SE2/EE** | Enterprise Oracle workloads | Bring Your Own License |
| **SQL Server** | .NET apps, SSRS/SSIS | Windows Auth support |
| **Aurora MySQL** | High-throughput MySQL | 5× faster than MySQL |
| **Aurora PostgreSQL** | High-throughput Postgres | 3× faster than Postgres |

### Instance Classes

| Class Family | Use Case | Example |
|---|---|---|
| `db.t3/t4g` | Dev/test, low-traffic | `db.t3.micro` (free tier) |
| `db.m5/m6i/m6g` | General production | `db.m6i.large` |
| `db.r5/r6i/r6g` | Memory-intensive | `db.r6i.4xlarge` |
| `db.x2g` | SAP HANA-scale memory | `db.x2g.16xlarge` |

### Storage Types

| Type | IOPS | Best For |
|------|------|---------|
| **gp2** | 3 IOPS/GB (burstable) | Dev, small workloads |
| **gp3** | 3,000 baseline, configurable up to 16,000 | Most production workloads |
| **io1 / io2** | Up to 256,000 | Mission-critical, high IOPS |
| **Magnetic** | Legacy only | Not recommended |

### Multi-AZ vs Read Replica

```
Multi-AZ (HA)                    Read Replica (Scale)
─────────────────                ─────────────────────
Primary ──sync──▶ Standby        Primary ──async──▶ Replica
  │                                │                  │
Automatic failover             Promoted to         Read traffic
(~60 sec RTO)                  standalone if       offloaded
                               needed
```

### Pricing Model

| Component | Pricing Basis |
|-----------|---------------|
| Instance | Per hour (On-Demand) or 1/3-yr Reserved |
| Storage | Per GB-month (gp3 ~$0.115/GB-mo) |
| I/O | Per million requests (gp2 only) |
| Backup | Free up to DB storage size; then per GB-mo |
| Data transfer | Free within region; charged cross-region |
| Multi-AZ | 2× instance cost (standby instance) |

> **Free Tier:** 750 hours/month of `db.t3.micro` (MySQL/PostgreSQL/MariaDB) + 20 GB gp2 storage for 12 months.

### Key Quotas (us-east-1 defaults)

| Quota | Default |
|-------|---------|
| DB instances per region | 40 |
| Max storage per instance | 64 TiB (gp3/io1) |
| Automated backup retention | 0–35 days |
| Read replicas per source | 15 (MySQL/PostgreSQL) |
| Max connections (db.t3.micro MySQL) | ~66 |

---

## 🏗️ Architecture Overview

```
┌──────────────────────────────────────────────────────────┐
│                        AWS Region                         │
│                                                           │
│   ┌─────────────────┐        ┌──────────────────────┐    │
│   │   Availability   │        │    Availability Zone  │    │
│   │    Zone A        │        │         B             │    │
│   │                  │        │                       │    │
│   │  ┌────────────┐  │ Sync   │  ┌─────────────────┐ │    │
│   │  │  Primary   │──┼───────▶│  │    Standby      │ │    │
│   │  │  DB Inst.  │  │        │  │   (Multi-AZ)    │ │    │
│   │  └────────────┘  │        │  └─────────────────┘ │    │
│   │                  │        │                       │    │
│   └─────────────────┘        └──────────────────────┘    │
│                                                           │
│   ┌─────────────────────────────────────────────────┐    │
│   │                    S3 (managed by RDS)           │    │
│   │         Automated Backups + Manual Snapshots     │    │
│   └─────────────────────────────────────────────────┘    │
└──────────────────────────────────────────────────────────┘
```

---

## 🔬 PoC Steps — Explore Without Creating Resources

### Step 1 — List available DB engines

```bash
aws rds describe-db-engine-versions \
  --query 'DBEngineVersions[*].[Engine,EngineVersion,DBEngineDescription]' \
  --output table \
  --region us-east-1 | head -60
```

**Expected output (truncated):**
```
-------------------------------------------------------------------
|               DescribeDBEngineVersions                          |
+-----------+--------+--------------------------------------------+
|  mysql    | 8.0.35 | MySQL Community Edition                    |
|  mysql    | 8.0.36 | MySQL Community Edition                    |
|  postgres | 15.6   | PostgreSQL                                 |
|  postgres | 16.2   | PostgreSQL                                 |
...
```

### Step 2 — List available DB instance classes for MySQL 8.0

```bash
aws rds describe-orderable-db-instance-options \
  --engine mysql \
  --engine-version 8.0.36 \
  --query 'OrderableDBInstanceOptions[*].[DBInstanceClass,StorageType,MultiAZCapable]' \
  --output table \
  --region us-east-1 | head -30
```

**Expected output (truncated):**
```
-----------------------------------------------
|   DescribeOrderableDBInstanceOptions         |
+----------------+----------+-----------------+
|  db.m5.large   | gp2      |  True           |
|  db.m5.large   | gp3      |  True           |
|  db.t3.micro   | gp2      |  True           |
|  db.t3.micro   | gp3      |  True           |
...
```

### Step 3 — Check your region's default VPC for RDS readiness

```bash
# Get default VPC ID
DEFAULT_VPC=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)
echo "Default VPC: $DEFAULT_VPC"

# List subnets in default VPC
aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$DEFAULT_VPC" \
  --query 'Subnets[*].[SubnetId,AvailabilityZone,CidrBlock]' \
  --output table
```

**Expected output:**
```
Default VPC: vpc-0abc1234def567890
-------------------------------------------------
|              DescribeSubnets                  |
+----------------------+-------------+----------+
|  subnet-0aa111bbb222 | us-east-1a  | 172.31.0.0/20 |
|  subnet-0cc333ddd444 | us-east-1b  | 172.31.16.0/20|
...
```

### Step 4 — Survey RDS pricing with Python boto3

```python
#!/usr/bin/env python3
"""List RDS on-demand pricing for db.t3.micro MySQL in us-east-1."""
import boto3
import json

# Pricing API only available in us-east-1
client = boto3.client("pricing", region_name="us-east-1")

response = client.get_products(
    ServiceCode="AmazonRDS",
    Filters=[
        {"Type": "TERM_MATCH", "Field": "databaseEngine", "Value": "MySQL"},
        {"Type": "TERM_MATCH", "Field": "instanceType",   "Value": "db.t3.micro"},
        {"Type": "TERM_MATCH", "Field": "location",       "Value": "US East (N. Virginia)"},
        {"Type": "TERM_MATCH", "Field": "deploymentOption", "Value": "Single-AZ"},
    ],
    MaxResults=1,
)

product = json.loads(response["PriceList"][0])
terms = product["terms"]["OnDemand"]
for term_key, term_val in terms.items():
    for price_key, price_val in term_val["priceDimensions"].items():
        print(f"Unit: {price_val['unit']}")
        print(f"Price: ${price_val['pricePerUnit']['USD']} / hour")
        print(f"Description: {price_val['description']}")
```

Run it:
```bash
python3 rds_pricing.py
```

**Expected output:**
```
Unit: Hrs
Price: $0.017 / hour
Description: $0.017 per RDS db.t3.micro Multi-AZ instance hour running MySQL
```

---

## 🧹 Cleanup

No resources were created in this lab — nothing to delete.

---

## ✅ What You Learned

- RDS supported engines and when to choose each
- Instance classes, storage types, and pricing model
- Multi-AZ vs Read Replicas conceptually
- How to explore RDS options using CLI and SDK without cost

---

> ➡️ Next: [Lab 02 — Launch Your First RDS Instance](../02-first-instance/README.md)
