# 🟡 Lab 03 — Backups, Snapshots & Parameter Groups

**Difficulty:** 🟡 Intermediate
**Time:** ~40 minutes
**Goal:** Configure automated backups, create and restore from a manual snapshot, tune MySQL with a custom parameter group, and understand point-in-time restore (PITR).

---

## 🎯 Goal

Master RDS data protection mechanisms: automated backups with PITR, manual snapshots for long-term retention, and parameter groups for engine tuning.

---

## 📖 Concepts

### Automated Backups vs Manual Snapshots

| Feature | Automated Backup | Manual Snapshot |
|---------|-----------------|-----------------|
| Trigger | Daily window (automatic) | User-initiated |
| Retention | 1–35 days | Until you delete it |
| Survives instance deletion | No (unless `--delete-automated-backups false`) | Yes |
| Point-in-time restore | ✅ Yes (to any second in retention window) | ❌ No (only snapshot moment) |
| Storage cost | Free up to DB size | $0.095/GB-mo after free tier |

### Transaction Logs

RDS ships transaction logs to S3 every **5 minutes**. Combined with daily snapshots, this enables PITR to **any 5-minute window** within the retention period.

### Parameter Groups

A **parameter group** is a named set of engine configuration values. You cannot modify the default parameter group, so you must create a custom one.

| Parameter | Default | Effect When Changed |
|-----------|---------|---------------------|
| `max_connections` | Auto (RAM-based) | Increase for high-concurrency workloads |
| `slow_query_log` | 0 (OFF) | Enable to capture slow queries |
| `long_query_time` | 10 (seconds) | Lower to 1s to catch more slow queries |
| `innodb_buffer_pool_size` | Auto | Controls InnoDB cache size |
| `character_set_server` | utf8mb4 | Default character set |

### Apply Methods

| Method | Effect |
|--------|--------|
| `immediate` | Applied right away (may cause brief disruption) |
| `pending-reboot` | Applied at next maintenance window or manual reboot |

---

## 🏗️ Architecture

```
┌──────────────────────────────────────────────────────────┐
│                    RDS Service                            │
│                                                           │
│  ┌──────────────────────┐    ┌───────────────────────┐  │
│  │   RDS Instance        │───▶│  Parameter Group      │  │
│  │   (my-rds-demo)       │    │  (custom-mysql8-pg)   │  │
│  └──────────┬────────────┘    └───────────────────────┘  │
│             │                                             │
│             │ Daily backup + TX logs                      │
│             ▼                                             │
│  ┌──────────────────────┐    ┌───────────────────────┐  │
│  │  Automated Backups    │    │   Manual Snapshot     │  │
│  │  (S3 - RDS managed)   │    │   (my-rds-snap-*)     │  │
│  └──────────────────────┘    └───────────────────────┘  │
│             │                           │                 │
│             ▼                           ▼                 │
│  ┌──────────────────────┐    ┌───────────────────────┐  │
│  │  PITR Restore         │    │  Restored from snap   │  │
│  │  (new instance)       │    │  (new instance)       │  │
│  └──────────────────────┘    └───────────────────────┘  │
└──────────────────────────────────────────────────────────┘
```

---

## 🔬 PoC Steps

### Step 1 — Launch instance with backup retention

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

# Create subnet group (skip if already exists from Lab 02)
aws rds create-db-subnet-group \
  --db-subnet-group-name rds-lab-subnet-group \
  --db-subnet-group-description "Lab subnet group" \
  --subnet-ids $SUBNET_IDS \
  --tags Key=Project,Value=rds-howto 2>/dev/null || echo "Subnet group already exists"

SG_ID=$(aws ec2 create-security-group \
  --group-name rds-lab-sg \
  --description "RDS lab SG" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=rds-howto}]" \
  --query 'GroupId' --output text 2>/dev/null || \
  aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=rds-lab-sg" "Name=vpc-id,Values=$VPC_ID" \
    --query 'SecurityGroups[0].GroupId' --output text)

MY_IP=$(curl -s https://checkip.amazonaws.com)/32
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 3306 --cidr $MY_IP 2>/dev/null || true

# Launch with 7-day backup retention
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
  --backup-retention-period 7 \
  --preferred-backup-window "03:00-04:00" \
  --preferred-maintenance-window "Mon:04:00-Mon:05:00" \
  --tags Key=Project,Value=rds-howto \
  --query 'DBInstance.[DBInstanceIdentifier,DBInstanceStatus]' \
  --output table

aws rds wait db-instance-available --db-instance-identifier $DB_ID
echo "Instance ready"
```

### Step 2 — Check automated backup configuration

```bash
aws rds describe-db-instances \
  --db-instance-identifier $DB_ID \
  --query 'DBInstances[0].{BackupRetentionPeriod:BackupRetentionPeriod,PreferredBackupWindow:PreferredBackupWindow,LatestRestorableTime:LatestRestorableTime}' \
  --output table
```

**Expected output:**
```
-------------------------------------------------------------------
|                     DescribeDBInstances                         |
+------------------------+--------+------------------------------+
|  BackupRetentionPeriod | 7      |                              |
|  PreferredBackupWindow | 03:00-04:00 |                         |
|  LatestRestorableTime  | 2026-03-17T08:05:00.000Z |           |
+------------------------+--------+------------------------------+
```

### Step 3 — Create a custom parameter group

```bash
# Create parameter group for MySQL 8.0
aws rds create-db-parameter-group \
  --db-parameter-group-name custom-mysql8-pg \
  --db-parameter-group-family mysql8.0 \
  --description "Custom MySQL 8.0 parameter group for tuning" \
  --tags Key=Project,Value=rds-howto \
  --query 'DBParameterGroup.DBParameterGroupName' \
  --output text
```

**Expected output:**
```
custom-mysql8-pg
```

### Step 4 — Modify parameters

```bash
aws rds modify-db-parameter-group \
  --db-parameter-group-name custom-mysql8-pg \
  --parameters \
    "ParameterName=slow_query_log,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=long_query_time,ParameterValue=1,ApplyMethod=immediate" \
    "ParameterName=max_connections,ParameterValue=100,ApplyMethod=pending-reboot" \
    "ParameterName=character_set_server,ParameterValue=utf8mb4,ApplyMethod=pending-reboot" \
  --query 'DBParameterGroupName' \
  --output text
```

**Expected output:**
```
custom-mysql8-pg
```

### Step 5 — Apply parameter group to the RDS instance

```bash
aws rds modify-db-instance \
  --db-instance-identifier $DB_ID \
  --db-parameter-group-name custom-mysql8-pg \
  --apply-immediately \
  --query 'DBInstance.[DBInstanceIdentifier,PendingModifiedValues]' \
  --output json
```

**Expected output:**
```json
[
    "my-rds-demo",
    {
        "DBParameterGroupName": "custom-mysql8-pg"
    }
]
```

### Step 6 — Verify parameter group was applied

```bash
aws rds describe-db-instances \
  --db-instance-identifier $DB_ID \
  --query 'DBInstances[0].DBParameterGroups' \
  --output table
```

**Expected output:**
```
-------------------------------------------------
|         DescribeDBInstances                   |
+-----------------------+-----------------------+
|  DBParameterGroupName | ParameterApplyStatus  |
+-----------------------+-----------------------+
|  custom-mysql8-pg     | pending-reboot        |
+-----------------------+-----------------------+
```

### Step 7 — Create a manual snapshot

```bash
SNAP_ID=my-rds-snap-$(date +%Y%m%d-%H%M)
aws rds create-db-snapshot \
  --db-instance-identifier $DB_ID \
  --db-snapshot-identifier $SNAP_ID \
  --tags Key=Project,Value=rds-howto \
  --query 'DBSnapshot.[DBSnapshotIdentifier,Status]' \
  --output table
```

**Expected output:**
```
----------------------------------------
|        CreateDBSnapshot              |
+----------------------+---------------+
|  my-rds-snap-20260317-0800 | creating |
+----------------------+---------------+
```

```bash
# Wait for snapshot to complete
aws rds wait db-snapshot-completed \
  --db-snapshot-identifier $SNAP_ID
echo "Snapshot ready: $SNAP_ID"
```

### Step 8 — List all snapshots for the instance

```bash
aws rds describe-db-snapshots \
  --db-instance-identifier $DB_ID \
  --query 'DBSnapshots[*].[DBSnapshotIdentifier,SnapshotType,Status,SnapshotCreateTime,AllocatedStorage]' \
  --output table
```

**Expected output:**
```
-------------------------------------------------------------------------------------------------------
|                                     DescribeDBSnapshots                                             |
+------------------------------+----------+-----------+----------------------------+------------------+
|  my-rds-snap-20260317-0800  | manual    | available | 2026-03-17T08:05:00.000Z | 20              |
|  rds:my-rds-demo-2026-03-17 | automated | available | 2026-03-17T03:10:00.000Z | 20              |
+------------------------------+----------+-----------+----------------------------+------------------+
```

### Step 9 — Restore a new instance from snapshot

```bash
aws rds restore-db-instance-from-db-snapshot \
  --db-instance-identifier my-rds-restored \
  --db-snapshot-identifier $SNAP_ID \
  --db-instance-class db.t3.micro \
  --publicly-accessible \
  --db-subnet-group-name rds-lab-subnet-group \
  --vpc-security-group-ids $SG_ID \
  --tags Key=Project,Value=rds-howto \
  --query 'DBInstance.[DBInstanceIdentifier,DBInstanceStatus]' \
  --output table
```

**Expected output:**
```
------------------------------------
|  RestoreDBInstanceFromDBSnapshot |
+------------------+---------------+
|  my-rds-restored | creating      |
+------------------+---------------+
```

### Step 10 — Python: Check latest restorable time

```python
#!/usr/bin/env python3
"""Display backup windows and PITR availability for an RDS instance."""
import boto3
from datetime import datetime, timezone

client = boto3.client("rds", region_name="us-east-1")

response = client.describe_db_instances(DBInstanceIdentifier="my-rds-demo")
db = response["DBInstances"][0]

latest = db.get("LatestRestorableTime")
now = datetime.now(timezone.utc)

print(f"Instance:                {db['DBInstanceIdentifier']}")
print(f"Backup Retention (days): {db['BackupRetentionPeriod']}")
print(f"Backup Window:           {db['PreferredBackupWindow']}")
print(f"Latest Restorable Time:  {latest.strftime('%Y-%m-%d %H:%M:%S UTC') if latest else 'N/A'}")
if latest:
    lag = now - latest
    print(f"PITR lag:                {int(lag.total_seconds() / 60)} minutes behind now")
```

Run:
```bash
python3 rds_pitr_check.py
```

**Expected output:**
```
Instance:                my-rds-demo
Backup Retention (days): 7
Backup Window:           03:00-04:00
Latest Restorable Time:  2026-03-17 08:10:00 UTC
PITR lag:                5 minutes behind now
```

---

## 🧹 Cleanup

```bash
# Delete restored instance
aws rds delete-db-instance \
  --db-instance-identifier my-rds-restored \
  --skip-final-snapshot

# Delete manual snapshot
aws rds delete-db-snapshot \
  --db-snapshot-identifier $SNAP_ID

# Delete original instance
aws rds delete-db-instance \
  --db-instance-identifier $DB_ID \
  --skip-final-snapshot

aws rds wait db-instance-deleted --db-instance-identifier $DB_ID
aws rds wait db-instance-deleted --db-instance-identifier my-rds-restored 2>/dev/null || true

# Delete parameter group (only after instances using it are deleted)
aws rds delete-db-parameter-group --db-parameter-group-name custom-mysql8-pg

# Delete subnet group and security group
aws rds delete-db-subnet-group --db-subnet-group-name rds-lab-subnet-group
aws ec2 delete-security-group --group-id $SG_ID

echo "Cleanup complete"
```

---

## ✅ What You Learned

- Difference between automated backups and manual snapshots
- How PITR works using daily snapshots + transaction logs
- How to create and apply a custom parameter group
- How to create a snapshot and restore a new instance from it

---

> ➡️ Next: [Lab 04 — Read Replicas & RDS Proxy](../04-read-replicas-proxy/README.md)
