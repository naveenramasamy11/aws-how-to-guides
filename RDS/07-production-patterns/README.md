# 🔴 Lab 07 — Production Patterns

**Difficulty:** 🔴 Advanced
**Time:** ~50 minutes
**Goal:** Implement production-grade RDS operations: CloudWatch alarms, Performance Insights query analysis, cost optimisation with Reserved Instances and storage autoscaling, blue/green deployments for zero-downtime upgrades, and automated maintenance runbooks.

---

## 🎯 Goal

Know what breaks in production RDS deployments before it happens, fix it faster, and reduce cost — using CloudWatch, Performance Insights, and AWS-native operational features.

---

## 📖 Concepts

### Critical RDS CloudWatch Metrics

| Metric | Alarm Threshold | Action |
|--------|-----------------|--------|
| `CPUUtilization` | > 80% for 5 min | Scale up instance class |
| `FreeStorageSpace` | < 10% of allocated | Enable storage autoscaling or resize |
| `DatabaseConnections` | > 80% of max_connections | Enable RDS Proxy |
| `ReadLatency` / `WriteLatency` | > 20ms avg | Investigate slow queries via PI |
| `FreeableMemory` | < 256 MB | Upsize instance or tune buffer pool |
| `ReplicaLag` | > 30 seconds | Investigate write-heavy workloads on primary |
| `FailedSQLServerAgentJobsCount` | > 0 | Check SQL Server Agent jobs (SQL Server only) |

### Performance Insights

Performance Insights shows you **what is causing database load**, broken down by:
- Wait events (I/O, lock, network, CPU)
- Top SQL statements by load
- Top hosts and users

It answers: "Which query is killing my database right now?"

### Storage Autoscaling

When `FreeStorageSpace` drops below 10%, RDS can automatically increase allocated storage up to a configured maximum. You pay for only what is used with gp3.

### Blue/Green Deployments

RDS Blue/Green creates a **staging environment** (green) mirroring production (blue) with logical replication. You test on green, then switch over (< 1 minute downtime for DNS cutover).

```
Blue (production)   ──logical replication──▶  Green (staging)
     MySQL 8.0.32                                MySQL 8.0.36
     Running live traffic                        Test upgraded version
                            │
                     Switchover
                     (DNS flip, ~1 min)
                            │
                            ▼
Green becomes production
Blue is decommissioned
```

### Maintenance Windows

RDS applies minor patches and DB engine upgrades during the **maintenance window**. Best practice:
- Set to off-peak hours in your region
- Enable Multi-AZ to get zero-downtime minor version upgrades
- Review `PendingMaintenanceActions` before each window

---

## 🏗️ Architecture

```
┌────────────────────────────────────────────────────────────────────┐
│                           Production Stack                          │
│                                                                     │
│  ┌──────────────────┐   ┌──────────────────┐  ┌────────────────┐  │
│  │  CloudWatch       │   │  Performance      │  │  EventBridge   │  │
│  │  Dashboard        │   │  Insights         │  │  (daily audit) │  │
│  │  + Alarms         │   │  (query analysis) │  │                │  │
│  └──────────┬────────┘   └──────────────────┘  └───────┬────────┘  │
│             │ SNS alert                                  │ Lambda    │
│             ▼                                            ▼           │
│  ┌──────────────────────────────────────────────────────────────┐  │
│  │  RDS Multi-AZ (Primary + Standby)                            │  │
│  │  Blue/Green for upgrades  |  Storage Autoscaling enabled     │  │
│  └──────────────────────────────────────────────────────────────┘  │
└────────────────────────────────────────────────────────────────────┘
```

---

## 🔬 PoC Steps

### Step 1 — Launch production-grade instance

```bash
export AWS_REGION=us-east-1
export DB_ID=my-rds-prod
export DB_USER=admin
export DB_PASS="ProdPass789!"
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

VPC_ID=$(aws ec2 describe-vpcs --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
  --query 'Subnets[0:2].SubnetId' --output text | tr '\t' ' ')

aws rds create-db-subnet-group \
  --db-subnet-group-name rds-prod-subnet-group \
  --db-subnet-group-description "Prod subnet group" \
  --subnet-ids $SUBNET_IDS \
  --tags Key=Project,Value=rds-howto 2>/dev/null || true

SG_ID=$(aws ec2 create-security-group \
  --group-name rds-prod-sg --description "RDS prod SG" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=rds-howto}]" \
  --query 'GroupId' --output text 2>/dev/null || \
  aws ec2 describe-security-groups \
    --filters "Name=group-name,Values=rds-prod-sg" "Name=vpc-id,Values=$VPC_ID" \
    --query 'SecurityGroups[0].GroupId' --output text)

MY_IP=$(curl -s https://checkip.amazonaws.com)/32
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID --protocol tcp --port 3306 --cidr $MY_IP 2>/dev/null || true

aws rds create-db-instance \
  --db-instance-identifier $DB_ID \
  --db-instance-class db.t3.micro \
  --engine mysql --engine-version 8.0.36 \
  --master-username $DB_USER --master-user-password "$DB_PASS" \
  --allocated-storage 20 --max-allocated-storage 100 \
  --storage-type gp3 \
  --multi-az \
  --publicly-accessible \
  --db-subnet-group-name rds-prod-subnet-group \
  --vpc-security-group-ids $SG_ID \
  --backup-retention-period 7 \
  --deletion-protection \
  --enable-performance-insights \
  --performance-insights-retention-period 7 \
  --monitoring-interval 60 \
  --enable-cloudwatch-logs-exports '["error","slowquery","general"]' \
  --tags Key=Project,Value=rds-howto Key=Environment,Value=production

aws rds wait db-instance-available --db-instance-identifier $DB_ID
echo "Production instance ready with Multi-AZ + PI + storage autoscaling"
```

### Step 2 — Create SNS topic for alarms

```bash
ALARM_SNS=$(aws sns create-topic \
  --name rds-prod-alarms \
  --tags Key=Project,Value=rds-howto \
  --query 'TopicArn' --output text)

aws sns subscribe \
  --topic-arn $ALARM_SNS \
  --protocol email \
  --notification-endpoint naveenramasamy11@gmail.com
echo "Alarm SNS: $ALARM_SNS — confirm your subscription email"
```

### Step 3 — Create CloudWatch alarms

```bash
# Helper function
create_alarm() {
  local name=$1 metric=$2 threshold=$3 comparison=$4 unit=$5
  aws cloudwatch put-metric-alarm \
    --alarm-name "$name" \
    --alarm-description "RDS Production: $metric" \
    --namespace AWS/RDS \
    --metric-name $metric \
    --dimensions "Name=DBInstanceIdentifier,Value=$DB_ID" \
    --statistic Average \
    --period 300 \
    --evaluation-periods 2 \
    --threshold $threshold \
    --comparison-operator $comparison \
    --unit $unit \
    --alarm-actions $ALARM_SNS \
    --ok-actions $ALARM_SNS \
    --tags Key=Project,Value=rds-howto
  echo "Alarm created: $name"
}

create_alarm "rds-prod-cpu-high"          CPUUtilization       80  GreaterThanThreshold     Percent
create_alarm "rds-prod-storage-low"       FreeStorageSpace     2000000000 LessThanThreshold Bytes
create_alarm "rds-prod-connections-high"  DatabaseConnections  50  GreaterThanThreshold     Count
create_alarm "rds-prod-read-latency"      ReadLatency          0.02 GreaterThanThreshold    Seconds
create_alarm "rds-prod-write-latency"     WriteLatency         0.02 GreaterThanThreshold    Seconds
create_alarm "rds-prod-memory-low"        FreeableMemory       268435456 LessThanThreshold  Bytes
```

### Step 4 — Verify alarm list

```bash
aws cloudwatch describe-alarms \
  --alarm-name-prefix "rds-prod-" \
  --query 'MetricAlarms[*].[AlarmName,StateValue,Threshold,ComparisonOperator]' \
  --output table
```

**Expected output:**
```
--------------------------------------------------------------------------------------------
|                            DescribeAlarms                                                |
+------------------------------+-----------+-----------+----------------------------------+
|  rds-prod-connections-high   | OK        | 50.0      | GreaterThanThreshold             |
|  rds-prod-cpu-high           | OK        | 80.0      | GreaterThanThreshold             |
|  rds-prod-memory-low         | OK        | 268435456.0 | LessThanThreshold              |
|  rds-prod-read-latency       | OK        | 0.02      | GreaterThanThreshold             |
|  rds-prod-storage-low        | OK        | 2000000000.0 | LessThanThreshold             |
|  rds-prod-write-latency      | OK        | 0.02      | GreaterThanThreshold             |
+------------------------------+-----------+-----------+----------------------------------+
```

### Step 5 — Check Performance Insights top SQL

```python
#!/usr/bin/env python3
"""Retrieve top SQL statements from RDS Performance Insights."""
import boto3
from datetime import datetime, timezone, timedelta

DB_ID  = "my-rds-prod"
REGION = "us-east-1"

# Get DBI resource ID (required for PI API)
rds_client = boto3.client("rds", region_name=REGION)
db_info    = rds_client.describe_db_instances(DBInstanceIdentifier=DB_ID)
dbi_id     = db_info["DBInstances"][0]["DbiResourceId"]
print(f"DBI Resource ID: {dbi_id}")

pi_client = boto3.client("pi", region_name=REGION)
end_time  = datetime.now(timezone.utc)
start_time = end_time - timedelta(hours=1)

response = pi_client.get_resource_metrics(
    ServiceType="RDS",
    Identifier=dbi_id,
    MetricQueries=[{
        "Metric": "db.load.avg",
        "GroupBy": {"Group": "db.sql", "Limit": 5},
    }],
    StartTime=start_time,
    EndTime=end_time,
    PeriodInSeconds=3600,
)

print(f"\nTop 5 SQL by DB load (last 1 hour):")
print("-" * 80)
for result in response.get("MetricList", []):
    for key in result.get("Keys", []):
        sql_text = key["Dimensions"].get("db.sql.statement", "N/A")[:70]
        print(f"SQL: {sql_text}")
        for dp in result.get("DataPoints", []):
            if dp.get("Value"):
                print(f"  Load: {dp['Value']:.4f}")
```

Run:
```bash
python3 rds_pi_top_sql.py
```

### Step 6 — Check pending maintenance actions

```bash
aws rds describe-pending-maintenance-actions \
  --resource-identifier "arn:aws:rds:${AWS_REGION}:${ACCOUNT_ID}:db:${DB_ID}" \
  --query 'PendingMaintenanceActions[*].PendingMaintenanceActionDetails[*].[Action,ForcedApplyDate,Description]' \
  --output table

# If no ARN version works, use plain describe:
aws rds describe-pending-maintenance-actions \
  --query 'PendingMaintenanceActions[?ResourceIdentifier==`'"arn:aws:rds:${AWS_REGION}:${ACCOUNT_ID}:db:${DB_ID}"'`]' \
  --output json
```

### Step 7 — Create a Blue/Green deployment for version upgrade

```bash
# Get the current instance ARN
DB_ARN=$(aws rds describe-db-instances \
  --db-instance-identifier $DB_ID \
  --query 'DBInstances[0].DBInstanceArn' --output text)

# Create Blue/Green deployment (green uses newer engine version)
BG_ID=$(aws rds create-blue-green-deployment \
  --blue-green-deployment-name rds-prod-bg-upgrade \
  --source $DB_ARN \
  --target-engine-version 8.0.36 \
  --tags Key=Project,Value=rds-howto \
  --query 'BlueGreenDeployment.BlueGreenDeploymentIdentifier' \
  --output text)
echo "Blue/Green deployment ID: $BG_ID"

# Monitor status
aws rds describe-blue-green-deployments \
  --blue-green-deployment-identifier $BG_ID \
  --query 'BlueGreenDeployments[0].{Status:Status,GreenEnv:GreenDeploymentDetails}' \
  --output json
```

### Step 8 — Switchover (when green is ready)

```bash
# When green is AVAILABLE and you're ready to cut over:
aws rds switchover-blue-green-deployment \
  --blue-green-deployment-identifier $BG_ID \
  --switchover-timeout 300

# Monitor switchover
aws rds describe-blue-green-deployments \
  --blue-green-deployment-identifier $BG_ID \
  --query 'BlueGreenDeployments[0].[Status,SwitchoverDetails]' \
  --output json
```

### Step 9 — Cost optimisation review with Python

```python
#!/usr/bin/env python3
"""
RDS Cost Optimisation Runbook
- Checks for idle instances (low connections)
- Reports storage utilisation
- Suggests Reserved Instance savings
"""
import boto3

rds    = boto3.client("rds",        region_name="us-east-1")
cw     = boto3.client("cloudwatch", region_name="us-east-1")
from datetime import datetime, timezone, timedelta

instances = rds.describe_db_instances()["DBInstances"]

print(f"{'Instance':<25} {'Class':<15} {'Engine':<10} {'MultiAZ':<8} {'Encrypted':<10} {'Suggestion'}")
print("-" * 110)

for db in instances:
    iid   = db["DBInstanceIdentifier"]
    cls   = db["DBInstanceClass"]
    eng   = db["Engine"]
    multi = db["MultiAZ"]
    enc   = db.get("StorageEncrypted", False)

    # Get avg CPU over last 7 days
    end   = datetime.now(timezone.utc)
    start = end - timedelta(days=7)
    resp  = cw.get_metric_statistics(
        Namespace="AWS/RDS",
        MetricName="CPUUtilization",
        Dimensions=[{"Name": "DBInstanceIdentifier", "Value": iid}],
        StartTime=start, EndTime=end,
        Period=604800, Statistics=["Average"],
    )
    avg_cpu = resp["Datapoints"][0]["Average"] if resp["Datapoints"] else 0

    suggestion = []
    if avg_cpu < 5:
        suggestion.append("LOW CPU - consider downsizing or stopping")
    if not multi and eng in ("mysql", "postgres", "mariadb"):
        suggestion.append("no Multi-AZ - consider for production")
    if not enc:
        suggestion.append("NOT ENCRYPTED - security risk")

    print(f"{iid:<25} {cls:<15} {eng:<10} {str(multi):<8} {str(enc):<10} {'; '.join(suggestion) or 'OK'}")
```

Run:
```bash
python3 rds_cost_review.py
```

### Step 10 — Automated daily health check via EventBridge + Lambda

```bash
# Create a Lambda for daily health check (simplified inline)
cat > /tmp/health_check.py <<'PYEOF'
import boto3, json
from datetime import datetime, timezone, timedelta

def lambda_handler(event, context):
    rds = boto3.client("rds")
    cw  = boto3.client("cloudwatch")
    sns = boto3.client("sns")
    SNS_ARN = "REPLACE_WITH_SNS_ARN"

    issues = []
    for db in rds.describe_db_instances()["DBInstances"]:
        iid = db["DBInstanceIdentifier"]
        if db["DBInstanceStatus"] != "available":
            issues.append(f"{iid}: status={db['DBInstanceStatus']}")
        # Check storage
        resp = cw.get_metric_statistics(
            Namespace="AWS/RDS", MetricName="FreeStorageSpace",
            Dimensions=[{"Name":"DBInstanceIdentifier","Value":iid}],
            StartTime=datetime.now(timezone.utc)-timedelta(hours=1),
            EndTime=datetime.now(timezone.utc),
            Period=3600, Statistics=["Minimum"],
        )
        if resp["Datapoints"]:
            free_gb = resp["Datapoints"][0]["Minimum"] / (1024**3)
            alloc_gb = db["AllocatedStorage"]
            pct_free = (free_gb / alloc_gb) * 100
            if pct_free < 15:
                issues.append(f"{iid}: storage {pct_free:.1f}% free ({free_gb:.1f}/{alloc_gb} GB)")

    msg = "RDS Daily Health Check\n" + ("-"*40) + "\n"
    msg += "\n".join(issues) if issues else "All instances healthy"
    sns.publish(TopicArn=SNS_ARN, Subject="RDS Daily Health Check", Message=msg)
    return {"issues": issues}
PYEOF
echo "Health check Lambda code written to /tmp/health_check.py"
echo "Deploy with: aws lambda create-function --function-name rds-daily-health-check ..."
echo "Schedule with: aws events put-rule --schedule-expression 'cron(0 8 * * ? *)' ..."
```

---

## 🧹 Cleanup

```bash
# Remove deletion protection first
aws rds modify-db-instance \
  --db-instance-identifier $DB_ID \
  --no-deletion-protection \
  --apply-immediately

# Delete Blue/Green deployment
aws rds delete-blue-green-deployment \
  --blue-green-deployment-identifier $BG_ID \
  --delete-target 2>/dev/null || true

sleep 10

# Delete CloudWatch alarms
aws cloudwatch delete-alarms \
  --alarm-names \
    rds-prod-cpu-high \
    rds-prod-storage-low \
    rds-prod-connections-high \
    rds-prod-read-latency \
    rds-prod-write-latency \
    rds-prod-memory-low

# Delete SNS topic
aws sns delete-topic --topic-arn $ALARM_SNS

# Delete RDS instance
aws rds delete-db-instance \
  --db-instance-identifier $DB_ID \
  --skip-final-snapshot
aws rds wait db-instance-deleted --db-instance-identifier $DB_ID

# Cleanup networking
aws rds delete-db-subnet-group --db-subnet-group-name rds-prod-subnet-group
aws ec2 delete-security-group --group-id $SG_ID

echo "Production cleanup complete"
```

---

## ✅ What You Learned

- Which CloudWatch metrics matter and what thresholds to alarm on
- How to use Performance Insights to identify top slow SQL statements
- How Blue/Green deployments enable zero-risk engine version upgrades
- How to write a cost optimisation runbook that flags idle and under-secured instances
- How to schedule daily health checks with EventBridge and Lambda

---

> 🎉 You have completed the RDS module! Return to [Module README](../README.md) for a summary.
