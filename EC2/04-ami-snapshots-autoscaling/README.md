# Lab 04 — AMIs, Auto Scaling Groups & Launch Templates

🟡 **Difficulty:** Intermediate | **Time:** 75 min | **Cost:** ~$0.20 (3 x t3.micro, ~30 min)

---

## Goal

Bake a custom AMI from a configured instance, create a versioned Launch Template, deploy an Auto Scaling Group across multiple AZs, and trigger a scale-out event to watch ASG launch new instances automatically.

---

## Concepts

### AMI Baking vs User-Data

| Approach | Boot time | Consistency | Use when |
|----------|-----------|-------------|----------|
| User-data bootstrap | Slow (packages install at boot) | Moderate | Dev/test, fast iteration |
| Baked AMI | Fast (software pre-installed) | High | Production, fast scale-out |
| Hybrid (AMI + user-data config) | Medium | High | Best practice for prod |

### Launch Template vs Launch Configuration

| Feature | Launch Template (LT) | Launch Config (LC) |
|---------|---------------------|-------------------|
| Versioning | Yes (v1, v2, v3...) | No |
| Mixed instance types | Yes | No |
| Spot + On-Demand mix | Yes | No |
| IMDSv2 support | Yes | No |
| Status | Recommended | Deprecated |

### Auto Scaling Group Key Concepts

```
ASG
├── Min capacity     — floor; ASG will never go below this
├── Desired capacity — target; ASG will converge to this
├── Max capacity     — ceiling; ASG will never go above this
├── Health check     — EC2 (default) or ELB
├── Cooldown period  — seconds to wait after a scaling event
└── Scaling policies
    ├── Target Tracking  — "keep CPU at 50 %"  (simplest)
    ├── Step Scaling     — step-based thresholds
    └── Scheduled        — time-based scale
```

---

## Architecture

```
                   Auto Scaling Group
     ┌──────────────────────────────────────────────┐
     │  Min=1  Desired=2  Max=5                      │
     │                                              │
     │  ┌────────────┐      ┌────────────┐          │
     │  │ EC2 (AZ-a) │      │ EC2 (AZ-b) │  ← 2 on launch
     │  └────────────┘      └────────────┘          │
     │       ▲                                      │
     │  CPU > 70 %  →  scale out to 3, 4, 5...      │
     └──────────────────────────────────────────────┘
             ▲
     Launch Template v2
     (custom AMI + httpd + gp3 8 GiB)
```

---

## PoC Steps

### Step 1 — Launch a "golden" instance to bake the AMI

```bash
export AWS_DEFAULT_REGION=us-east-1

AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)

SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query 'Subnets[0].SubnetId' --output text)

MY_IP=$(curl -s https://checkip.amazonaws.com)

SG_ID=$(aws ec2 create-security-group \
  --group-name "ec2-lab04-sg" \
  --description "Lab 04 ASG SG" \
  --vpc-id "$VPC_ID" \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ec2-lab04-sg},{Key=Lab,Value=EC2-04}]" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" --protocol tcp --port 22 --cidr "${MY_IP}/32"
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" --protocol tcp --port 80 --cidr "0.0.0.0/0"

GOLDEN_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --key-name ec2-labs-key \
  --security-group-ids "$SG_ID" \
  --subnet-id "$SUBNET_ID" \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ec2-lab04-golden},{Key=Lab,Value=EC2-04}]' \
  --query 'Instances[0].InstanceId' --output text)

aws ec2 wait instance-running --instance-ids "$GOLDEN_ID"

GOLDEN_IP=$(aws ec2 describe-instances \
  --instance-ids "$GOLDEN_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Golden instance: $GOLDEN_ID  IP: $GOLDEN_IP"
```

### Step 2 — Install software on the golden instance

```bash
ssh -o StrictHostKeyChecking=no -i ~/ec2-labs-key.pem ec2-user@"$GOLDEN_IP" << 'REMOTE'
sudo dnf update -y
sudo dnf install -y httpd stress-ng

# Create a static page that ASG instances will serve
sudo bash -c 'cat > /var/www/html/index.html << HTML
<html><body style="font-family:monospace;padding:20px;">
<h2>AMI Baked Instance</h2>
<p>Served from a pre-baked AMI — fast boot, consistent config.</p>
</body></html>
HTML'

sudo systemctl enable httpd
# Do NOT start httpd here — let user-data handle env-specific config at launch
sudo dnf clean all
REMOTE

echo "Golden instance configured"
```

### Step 3 — Stop the instance and create an AMI

```bash
# Stop the instance before baking (for filesystem consistency)
aws ec2 stop-instances --instance-ids "$GOLDEN_ID"
aws ec2 wait instance-stopped --instance-ids "$GOLDEN_ID"

CUSTOM_AMI=$(aws ec2 create-image \
  --instance-id "$GOLDEN_ID" \
  --name "ec2-lab04-httpd-ami-$(date +%Y%m%d%H%M)" \
  --description "Lab 04 golden AMI with httpd pre-installed" \
  --no-reboot \
  --tag-specifications "ResourceType=image,Tags=[{Key=Name,Value=ec2-lab04-ami},{Key=Lab,Value=EC2-04}]" \
  --query 'ImageId' \
  --output text)

echo "AMI: $CUSTOM_AMI"
echo "Waiting for AMI to be available (2-5 min)..."
aws ec2 wait image-available --image-ids "$CUSTOM_AMI"
echo "AMI ready"
```

### Step 4 — Create a Launch Template (v1)

```bash
# Minimal user-data: start httpd and inject instance-id into page
USER_DATA=$(base64 << 'USERDATA'
#!/bin/bash
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
INAME=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
sed -i "s/AMI Baked Instance/ASG Instance: $INAME/" /var/www/html/index.html
systemctl start httpd
USERDATA
)

LT_ID=$(aws ec2 create-launch-template \
  --launch-template-name "ec2-lab04-lt" \
  --version-description "v1 - httpd AMI" \
  --launch-template-data "{
    \"ImageId\": \"$CUSTOM_AMI\",
    \"InstanceType\": \"t3.micro\",
    \"KeyName\": \"ec2-labs-key\",
    \"SecurityGroupIds\": [\"$SG_ID\"],
    \"UserData\": \"$USER_DATA\",
    \"MetadataOptions\": {
      \"HttpTokens\": \"required\",
      \"HttpPutResponseHopLimit\": 1
    },
    \"TagSpecifications\": [{
      \"ResourceType\": \"instance\",
      \"Tags\": [{\"Key\": \"Name\", \"Value\": \"ec2-lab04-asg\"}, {\"Key\": \"Lab\", \"Value\": \"EC2-04\"}]
    }]
  }" \
  --tag-specifications "ResourceType=launch-template,Tags=[{Key=Name,Value=ec2-lab04-lt},{Key=Lab,Value=EC2-04}]" \
  --query 'LaunchTemplate.LaunchTemplateId' \
  --output text)

echo "Launch Template: $LT_ID"
```

### Step 5 — Create the Auto Scaling Group

```bash
# Get all AZs with subnets for multi-AZ deployment
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query 'Subnets[*].SubnetId' \
  --output text | tr '\t' ',')
echo "Subnets: $SUBNET_IDS"

aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "ec2-lab04-asg" \
  --launch-template "LaunchTemplateId=$LT_ID,Version=1" \
  --min-size 1 \
  --desired-capacity 2 \
  --max-size 5 \
  --vpc-zone-identifier "$SUBNET_IDS" \
  --health-check-type EC2 \
  --health-check-grace-period 120 \
  --tags "Key=Name,Value=ec2-lab04-asg,PropagateAtLaunch=false" \
         "Key=Lab,Value=EC2-04,PropagateAtLaunch=false"

echo "ASG created"
```

### Step 6 — Add a Target Tracking scaling policy

```bash
aws autoscaling put-scaling-policy \
  --auto-scaling-group-name "ec2-lab04-asg" \
  --policy-name "ec2-lab04-cpu-target" \
  --policy-type "TargetTrackingScaling" \
  --target-tracking-configuration '{
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ASGAverageCPUUtilization"
    },
    "TargetValue": 50.0,
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'

echo "Scaling policy attached"
```

### Step 7 — Watch ASG activity and verify instances

```bash
# Check ASG state
aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names "ec2-lab04-asg" \
  --query 'AutoScalingGroups[0].{Min:MinSize,Max:MaxSize,Desired:DesiredCapacity,Instances:Instances[*].{ID:InstanceId,State:LifecycleState,Health:HealthStatus}}' \
  --output json

# List ASG activities
aws autoscaling describe-scaling-activities \
  --auto-scaling-group-name "ec2-lab04-asg" \
  --max-items 5 \
  --query 'Activities[*].{Time:StartTime,Status:StatusCode,Cause:Cause}' \
  --output table
```

Expected output:
```json
{
    "Min": 1, "Max": 5, "Desired": 2,
    "Instances": [
        {"ID": "i-0abc...", "State": "InService", "Health": "Healthy"},
        {"ID": "i-0def...", "State": "InService", "Health": "Healthy"}
    ]
}
```

### Step 8 — Simulate scale-out by updating desired capacity

```bash
# Scale out to 4 instances manually (simulates high demand)
aws autoscaling set-desired-capacity \
  --auto-scaling-group-name "ec2-lab04-asg" \
  --desired-capacity 4 \
  --honor-cooldown

# Watch instances being launched
watch -n 5 'aws autoscaling describe-auto-scaling-groups \
  --auto-scaling-group-names ec2-lab04-asg \
  --query "AutoScalingGroups[0].Instances[*].{ID:InstanceId,State:LifecycleState}" \
  --output table'
```

---

## Cleanup

```bash
# Delete ASG (this terminates all instances it manages)
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name "ec2-lab04-asg" \
  --force-delete

# Terminate golden instance
aws ec2 terminate-instances --instance-ids "$GOLDEN_ID"
aws ec2 wait instance-terminated --instance-ids "$GOLDEN_ID"

# Deregister AMI and delete snapshot
SNAP_ID=$(aws ec2 describe-images \
  --image-ids "$CUSTOM_AMI" \
  --query 'Images[0].BlockDeviceMappings[0].Ebs.SnapshotId' --output text)
aws ec2 deregister-image --image-id "$CUSTOM_AMI"
aws ec2 delete-snapshot --snapshot-id "$SNAP_ID"

# Delete launch template and security group
aws ec2 delete-launch-template --launch-template-id "$LT_ID"
aws ec2 delete-security-group --group-id "$SG_ID"
echo "Cleanup complete"
```

---

## Key Takeaways

- Baked AMIs dramatically reduce scale-out boot times compared to user-data-only bootstrapping
- Launch Templates replace Launch Configurations — use LTs for all new ASGs
- ASG desired capacity converges automatically; target tracking is the simplest scaling policy
- Always multi-AZ: spread subnets across at least 2 AZs so the ASG survives AZ failure
- Deregister the AMI and delete its EBS snapshot during cleanup to avoid ongoing storage costs

---

➡️ Next: [Lab 05 — IAM Roles, Security Groups & SSM Session Manager](../05-iam-security/README.md)
