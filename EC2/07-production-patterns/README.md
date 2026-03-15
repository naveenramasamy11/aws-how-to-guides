# Lab 07 — Production Patterns: CloudWatch Monitoring, Cost Optimisation & Resilience

🔴 **Difficulty:** Advanced | **Time:** 60 min | **Cost:** $0 (configuration and dashboard — no running instances)

---

## Goal

Apply production-readiness to your EC2 workloads: build a CloudWatch dashboard with the critical EC2 metrics, configure unified agent for OS-level metrics, implement Savings Plans cost analysis, and design resilience with multi-AZ + lifecycle hooks.

---

## Concepts

### CloudWatch Metrics — EC2 Default vs Custom

| Metric | Namespace | Default? | Frequency |
|--------|-----------|---------|-----------|
| CPUUtilization | AWS/EC2 | Yes | 5 min (1 min with detailed monitoring) |
| NetworkIn / NetworkOut | AWS/EC2 | Yes | 5 min |
| DiskReadOps / DiskWriteOps | AWS/EC2 | Yes (EBS) | 5 min |
| StatusCheckFailed | AWS/EC2 | Yes | 1 min |
| **mem_used_percent** | CWAgent | No — requires CloudWatch Agent | 1 min |
| **disk_used_percent** | CWAgent | No — requires CloudWatch Agent | 1 min |
| **cpu_usage_iowait** | CWAgent | No — requires CloudWatch Agent | 1 min |

### CloudWatch Agent Architecture

```
EC2 Instance
├── amazon-cloudwatch-agent (systemd service)
│   ├── Reads: /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json
│   ├── Collects: procfs (/proc/meminfo, /proc/diskstats, etc.)
│   └── Ships to: CloudWatch Metrics (namespace CWAgent)
│
└── Logs:
    ├── /var/log/messages       → CloudWatch Logs group /ec2/syslog
    └── /var/log/httpd/access_log → CloudWatch Logs group /ec2/httpd
```

### Savings Plans vs Reserved Instances

| Feature | Savings Plans | Reserved Instances |
|---------|--------------|-------------------|
| Commitment | $/hr spend level | Specific instance type |
| Flexibility | Any instance family (Compute SP) | Same family or region (RI) |
| EC2-specific | EC2 Instance SP (higher discount) | Standard / Convertible RI |
| Max discount | ~66 % (3yr, no upfront, Compute) | ~72 % (3yr, all upfront, Standard) |
| Recommendation | AWS Cost Explorer auto-suggests | Manual analysis |

### Multi-AZ Resilience Design

```
Region us-east-1
├── AZ us-east-1a
│   └── EC2 instances (min 1 in each AZ)
├── AZ us-east-1b
│   └── EC2 instances
└── AZ us-east-1c
    └── EC2 instances

ASG spans all 3 AZs:
  If AZ-a fails → ASG rebalances to AZ-b + AZ-c automatically
```

### Lifecycle Hooks

```
Instance Launch
     │
     ▼
Pending:Wait  ← hook fires; Lambda/SQS handles warm-up
     │
     ▼  (complete-lifecycle-action)
InService

Instance Terminate
     │
     ▼
Terminating:Wait  ← hook fires; Lambda drains connections, ships logs
     │
     ▼  (complete-lifecycle-action)
Terminated
```

---

## Architecture

```
CloudWatch Dashboard
├── CPU Utilisation (per-instance)
├── Memory Used % (CWAgent)
├── Disk Used % (CWAgent)
├── Network In/Out
└── ALB Request Count + Latency

CloudWatch Alarms
├── CPU > 70 % → SNS email + ASG scale-out
├── StatusCheckFailed = 1 → SNS page
└── mem_used_percent > 85 % → SNS email

Cost Optimisation
├── Compute Savings Plan (flex coverage)
├── Auto Scaling scheduled scale-in (nights/weekends)
└── gp3 instead of gp2 (20 % cost reduction, better perf)
```

---

## PoC Steps

### Step 1 — Enable detailed monitoring on existing ASG instances

```bash
export AWS_DEFAULT_REGION=us-east-1

# Enable 1-minute detailed monitoring on all running instances tagged Lab=EC2-07
# In practice replace this with your ASG instance IDs
INSTANCE_IDS=$(aws ec2 describe-instances \
  --filters "Name=instance-state-name,Values=running" \
  --query 'Reservations[*].Instances[*].InstanceId' \
  --output text | tr '\t' ' ')

if [ -n "$INSTANCE_IDS" ]; then
  aws ec2 monitor-instances --instance-ids $INSTANCE_IDS
  echo "Detailed monitoring enabled for: $INSTANCE_IDS"
else
  echo "No running instances found (launch one from a previous lab to test this step)"
fi
```

### Step 2 — Install and configure the CloudWatch Unified Agent

This user-data snippet installs and configures the agent on Amazon Linux 2023:

```bash
cat > /tmp/cw-agent-userdata.sh << 'USERDATA'
#!/bin/bash
# Install CloudWatch agent
dnf install -y amazon-cloudwatch-agent

# Write agent configuration
cat > /opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json << 'CONFIG'
{
  "metrics": {
    "namespace": "CWAgent",
    "append_dimensions": {
      "InstanceId": "${aws:InstanceId}",
      "AutoScalingGroupName": "${aws:AutoScalingGroupName}"
    },
    "metrics_collected": {
      "mem": {
        "measurement": ["mem_used_percent"],
        "metrics_collection_interval": 60
      },
      "disk": {
        "measurement": ["disk_used_percent"],
        "resources": ["/", "/data"],
        "metrics_collection_interval": 60
      },
      "cpu": {
        "measurement": ["cpu_usage_iowait", "cpu_usage_user", "cpu_usage_system"],
        "metrics_collection_interval": 60,
        "totalcpu": true
      }
    }
  },
  "logs": {
    "logs_collected": {
      "files": {
        "collect_list": [
          {
            "file_path": "/var/log/messages",
            "log_group_name": "/ec2/syslog",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 7
          },
          {
            "file_path": "/var/log/httpd/access_log",
            "log_group_name": "/ec2/httpd",
            "log_stream_name": "{instance_id}",
            "retention_in_days": 7
          }
        ]
      }
    }
  }
}
CONFIG

# Start the agent
/opt/aws/amazon-cloudwatch-agent/bin/amazon-cloudwatch-agent-ctl \
  -a fetch-config -m ec2 \
  -c file:/opt/aws/amazon-cloudwatch-agent/etc/amazon-cloudwatch-agent.json \
  -s

systemctl enable amazon-cloudwatch-agent
USERDATA

echo "CloudWatch agent user-data written to /tmp/cw-agent-userdata.sh"
echo "Include this in your Launch Template user-data for all production instances"
```

### Step 3 — Create a CloudWatch Dashboard

```bash
DASHBOARD_BODY=$(cat << 'DASH'
{
  "widgets": [
    {
      "type": "metric",
      "x": 0, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "EC2 CPU Utilisation",
        "metrics": [[ "AWS/EC2", "CPUUtilization", {"stat":"Average","period":60} ]],
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 12, "y": 0, "width": 12, "height": 6,
      "properties": {
        "title": "Memory Used % (CWAgent)",
        "metrics": [[ "CWAgent", "mem_used_percent", {"stat":"Average","period":60} ]],
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 0, "y": 6, "width": 12, "height": 6,
      "properties": {
        "title": "Network In / Out",
        "metrics": [
          [ "AWS/EC2", "NetworkIn",  {"stat":"Sum","period":60} ],
          [ "AWS/EC2", "NetworkOut", {"stat":"Sum","period":60} ]
        ],
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 12, "y": 6, "width": 12, "height": 6,
      "properties": {
        "title": "Disk Used % (CWAgent)",
        "metrics": [[ "CWAgent", "disk_used_percent", {"stat":"Maximum","period":60} ]],
        "view": "timeSeries",
        "region": "us-east-1"
      }
    },
    {
      "type": "metric",
      "x": 0, "y": 12, "width": 12, "height": 6,
      "properties": {
        "title": "EC2 Status Check Failed",
        "metrics": [[ "AWS/EC2", "StatusCheckFailed", {"stat":"Maximum","period":60} ]],
        "view": "timeSeries",
        "region": "us-east-1"
      }
    }
  ]
}
DASH
)

aws cloudwatch put-dashboard \
  --dashboard-name "EC2-Lab07-Production" \
  --dashboard-body "$DASHBOARD_BODY"

echo "Dashboard created: https://console.aws.amazon.com/cloudwatch/home?region=us-east-1#dashboards:name=EC2-Lab07-Production"
```

### Step 4 — Create production CloudWatch alarms

```bash
SNS_ARN=$(aws sns list-topics \
  --query 'Topics[?contains(TopicArn,`ec2-lab`)].TopicArn | [0]' \
  --output text 2>/dev/null || echo "arn:aws:sns:us-east-1:$(aws sts get-caller-identity --query Account --output text):ec2-alerts")

# Alarm: StatusCheckFailed (hardware/software failure)
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-prod-status-check-failed" \
  --alarm-description "Instance status check failed — possible hardware issue" \
  --namespace AWS/EC2 \
  --metric-name StatusCheckFailed \
  --statistic Maximum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions "$SNS_ARN" \
  --ok-actions "$SNS_ARN" \
  --tags Key=Lab,Value=EC2-07

# Alarm: CPU sustained high
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-prod-cpu-high" \
  --alarm-description "Sustained high CPU — investigate or scale out" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --statistic Average \
  --period 300 \
  --evaluation-periods 3 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "$SNS_ARN" \
  --tags Key=Lab,Value=EC2-07

# List the alarms
aws cloudwatch describe-alarms \
  --alarm-names "ec2-prod-status-check-failed" "ec2-prod-cpu-high" \
  --query 'MetricAlarms[*].{Name:AlarmName,State:StateValue,Threshold:Threshold}' \
  --output table
```

### Step 5 — Analyse Cost Explorer Savings Plan recommendations

```bash
# Get EC2 spend last 30 days
aws ce get-cost-and-usage \
  --time-period Start=$(date -d '30 days ago' +%Y-%m-%d),End=$(date +%Y-%m-%d) \
  --granularity MONTHLY \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}' \
  --metrics BlendedCost \
  --query 'ResultsByTime[*].{Start:TimePeriod.Start,Cost:Total.BlendedCost.Amount,Unit:Total.BlendedCost.Unit}' \
  --output table

# Get Savings Plan purchase recommendations (Compute SP, 1yr, No Upfront)
aws ce get-savings_plans_purchase_recommendation \
  --savings-plans-type "COMPUTE_SP" \
  --term-in-years "ONE_YEAR" \
  --payment-option "NO_UPFRONT" \
  --lookback-period-in-days THIRTY_DAYS \
  --query 'SavingsPlansPurchaseRecommendation.SavingsPlansPurchaseRecommendationDetails[0].{
    HourlyCommitment:HourlyCommitmentToPurchase,
    EstimatedROI:EstimatedROI,
    EstimatedSavingsAmount:EstimatedSavingsAmount,
    EstimatedSavingsPercentage:EstimatedSavingsPercentage
  }' \
  --output table 2>/dev/null || echo "No SP recommendation data available (insufficient spend history)"
```

### Step 6 — Create lifecycle hooks for graceful scale-in

```bash
# This attaches to an existing ASG named ec2-lab06-asg (from Lab 06)
# Adjust ASG name to match your environment

ASG_NAME="ec2-lab06-asg"

# Launch hook — wait up to 5 min for warm-up (e.g. config management, app registration)
aws autoscaling put-lifecycle-hook \
  --auto-scaling-group-name "$ASG_NAME" \
  --lifecycle-hook-name "ec2-warmup-hook" \
  --lifecycle-transition autoscaling:EC2_INSTANCE_LAUNCHING \
  --heartbeat-timeout 300 \
  --default-result CONTINUE

# Terminate hook — wait up to 90 sec for connection draining / log shipping
aws autoscaling put-lifecycle-hook \
  --auto-scaling-group-name "$ASG_NAME" \
  --lifecycle-hook-name "ec2-drain-hook" \
  --lifecycle-transition autoscaling:EC2_INSTANCE_TERMINATING \
  --heartbeat-timeout 90 \
  --default-result CONTINUE

# Describe hooks
aws autoscaling describe-lifecycle-hooks \
  --auto-scaling-group-name "$ASG_NAME" \
  --query 'LifecycleHooks[*].{Name:LifecycleHookName,Transition:LifecycleTransition,Timeout:HeartbeatTimeout,Default:DefaultResult}' \
  --output table 2>/dev/null || echo "ASG $ASG_NAME not found — lifecycle hooks are a reference pattern for production ASGs"
```

Expected output:
```
-------------------------------------------------------------------------------------
|                              DescribeLifecycleHooks                               |
+-----------------+---------+-------+--------------------------------------------+
|    Default      | Name    | Timeout | Transition                                |
+-----------------+---------+-------+--------------------------------------------+
|   CONTINUE      | ec2-drain-hook  | 90 | autoscaling:EC2_INSTANCE_TERMINATING  |
|   CONTINUE      | ec2-warmup-hook | 300| autoscaling:EC2_INSTANCE_LAUNCHING    |
+-----------------+---------+-------+--------------------------------------------+
```

### Step 7 — gp2 to gp3 migration (cost + performance win)

```bash
# Find all gp2 volumes in the account (typically 20% cheaper as gp3)
aws ec2 describe-volumes \
  --filters "Name=volume-type,Values=gp2" \
             "Name=status,Values=available,in-use" \
  --query 'Volumes[*].{ID:VolumeId,SizeGiB:Size,AZ:AvailabilityZone,State:State}' \
  --output table

# Migrate a specific volume from gp2 to gp3 (online, zero downtime)
# Replace vol-0example with a real volume ID
EXAMPLE_VOL="vol-0example"
aws ec2 modify-volume \
  --volume-id "$EXAMPLE_VOL" \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 2>/dev/null || echo "Replace vol-0example with a real volume ID to run migration"

# Monitor migration progress
aws ec2 describe-volumes-modifications \
  --query 'VolumesModifications[*].{Vol:VolumeId,State:ModificationState,Progress:Progress,TargetType:TargetVolumeType}' \
  --output table 2>/dev/null || echo "No volume modifications in progress"
```

---

## Cleanup

```bash
# Delete CloudWatch alarms and dashboard
aws cloudwatch delete-alarms \
  --alarm-names "ec2-prod-status-check-failed" "ec2-prod-cpu-high"

aws cloudwatch delete-dashboards --dashboard-names "EC2-Lab07-Production"

echo "Cleanup complete — no instances were created in this lab"
```

---

## Production Checklist

```
EC2 Production Readiness Checklist
====================================
[ ] IMDSv2 enforced (HttpTokens=required) on all instances
[ ] No key pairs — use SSM Session Manager
[ ] No inbound SSH (22) in Security Groups
[ ] IAM instance profile with least-privilege policies
[ ] CloudWatch Unified Agent installed (mem, disk, logs)
[ ] StatusCheckFailed alarm → SNS page
[ ] CPU / memory alarms → SNS notification
[ ] Multi-AZ ASG with min 2 AZs
[ ] Lifecycle hooks for graceful launch/terminate
[ ] gp3 volumes (not gp2) for all EBS
[ ] Detailed monitoring enabled on production instances
[ ] Savings Plans or Reserved Instances covering baseline
[ ] CloudWatch Dashboard per service
[ ] Log group retention set (not indefinite)
```

---

## Key Takeaways

- CloudWatch Unified Agent is required for memory and disk metrics — EC2 hypervisor cannot see inside the OS
- StatusCheckFailed is the most critical EC2 alarm — triggers immediate response for hardware faults
- Lifecycle hooks are essential for zero-downtime deployments: warm-up on launch, drain on terminate
- gp2 to gp3 migration is zero-downtime and online — no reason to keep gp2 volumes
- Compute Savings Plans cover EC2 + Fargate + Lambda — more flexible than Standard Reserved Instances

---

You have completed the EC2 module! Review the [module README](../README.md) for a full learning path summary.
