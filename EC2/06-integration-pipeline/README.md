# Lab 06 — Integration Pipeline: ALB + ASG + Lambda + SNS

🔴 **Difficulty:** Advanced | **Time:** 90 min | **Cost:** ~$0.30 (ALB + 2 t3.micro ~30 min)

---

## Goal

Build a production-style event-driven pipeline: an Application Load Balancer distributes traffic to an Auto Scaling Group; CloudWatch alarms trigger a Lambda function that publishes scale notifications to an SNS topic.

---

## Concepts

### ALB vs NLB vs CLB

| Type | Layer | Protocol | Features |
|------|-------|----------|----------|
| **ALB** (Application) | 7 | HTTP/HTTPS/gRPC | Path/host routing, WAF, sticky sessions |
| **NLB** (Network) | 4 | TCP/UDP/TLS | Ultra-low latency, static IP, PrivateLink |
| **CLB** (Classic) | 4/7 | HTTP/TCP | Legacy — avoid for new workloads |

### ALB Component Model

```
ALB (Internet-facing)
  └── Listener (port 80, protocol HTTP)
       └── Default Rule
            └── Forward to Target Group
                 └── Target Group (type=instance)
                      ├── EC2 Instance A  (healthy)
                      └── EC2 Instance B  (healthy)
```

### Event-Driven Scale Notification Pipeline

```
CloudWatch Alarm (CPU > 70%)
        │
        │  triggers
        ▼
    Lambda Function
        │  publishes
        ▼
     SNS Topic
        │  fans out to
        ├── Email subscriber
        └── (extensible: Slack, PagerDuty, JIRA)
```

---

## Architecture

```
Internet
   │
   ▼
ALB (Internet-facing)
   │  HTTP :80
   ▼
Target Group (ec2-lab06-tg)
   ├── EC2 (AZ-a)  ←── ASG manages these
   └── EC2 (AZ-b)
         │
         │ CPU alarm
         ▼
   CloudWatch → Lambda → SNS → Email
```

---

## PoC Steps

### Step 1 — Environment setup

```bash
export AWS_DEFAULT_REGION=us-east-1
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)

# Get all default subnets across AZs
SUBNET_IDS=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query 'Subnets[*].SubnetId' \
  --output text | tr '\t' ',')

echo "Account: $ACCOUNT_ID  VPC: $VPC_ID  Subnets: $SUBNET_IDS"
```

### Step 2 — Create security groups (ALB + EC2)

```bash
# ALB security group — accept HTTP from internet
ALB_SG=$(aws ec2 create-security-group \
  --group-name "ec2-lab06-alb-sg" \
  --description "Lab 06 ALB SG" \
  --vpc-id "$VPC_ID" \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ec2-lab06-alb-sg},{Key=Lab,Value=EC2-06}]" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$ALB_SG" --protocol tcp --port 80 --cidr 0.0.0.0/0

# EC2 security group — accept HTTP only from ALB SG (no direct internet access)
EC2_SG=$(aws ec2 create-security-group \
  --group-name "ec2-lab06-ec2-sg" \
  --description "Lab 06 EC2 SG" \
  --vpc-id "$VPC_ID" \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ec2-lab06-ec2-sg},{Key=Lab,Value=EC2-06}]" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$EC2_SG" \
  --protocol tcp --port 80 \
  --source-group "$ALB_SG"

echo "ALB SG: $ALB_SG  EC2 SG: $EC2_SG"
```

### Step 3 — Create ALB and Target Group

```bash
# Create the ALB (subnet IDs as space-separated list for ALB)
SUBNET_LIST=$(echo $SUBNET_IDS | tr ',' ' ')

ALB_ARN=$(aws elbv2 create-load-balancer \
  --name ec2-lab06-alb \
  --subnets $SUBNET_LIST \
  --security-groups "$ALB_SG" \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --tags Key=Name,Value=ec2-lab06-alb Key=Lab,Value=EC2-06 \
  --query 'LoadBalancers[0].LoadBalancerArn' \
  --output text)

echo "ALB ARN: $ALB_ARN"

# Create target group (HTTP, health check on /)
TG_ARN=$(aws elbv2 create-target-group \
  --name ec2-lab06-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id "$VPC_ID" \
  --health-check-path / \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 3 \
  --tags Key=Name,Value=ec2-lab06-tg Key=Lab,Value=EC2-06 \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text)

echo "Target Group: $TG_ARN"

# Create listener
aws elbv2 create-listener \
  --load-balancer-arn "$ALB_ARN" \
  --protocol HTTP \
  --port 80 \
  --default-actions "Type=forward,TargetGroupArn=$TG_ARN"

echo "Listener created"
```

### Step 4 — Create Launch Template and ASG pointing to TG

```bash
USER_DATA=$(base64 << 'USERDATA'
#!/bin/bash
dnf install -y httpd
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 60")
INAME=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
echo "<html><body><h2>Lab 06 — Instance: $INAME</h2></body></html>" > /var/www/html/index.html
systemctl enable --now httpd
USERDATA
)

LT_ID=$(aws ec2 create-launch-template \
  --launch-template-name "ec2-lab06-lt" \
  --version-description "v1" \
  --launch-template-data "{
    \"ImageId\": \"$AMI_ID\",
    \"InstanceType\": \"t3.micro\",
    \"KeyName\": \"ec2-labs-key\",
    \"SecurityGroupIds\": [\"$EC2_SG\"],
    \"UserData\": \"$USER_DATA\",
    \"MetadataOptions\": {\"HttpTokens\": \"required\", \"HttpPutResponseHopLimit\": 1},
    \"TagSpecifications\": [{
      \"ResourceType\": \"instance\",
      \"Tags\": [{\"Key\": \"Name\", \"Value\": \"ec2-lab06-asg\"}, {\"Key\": \"Lab\", \"Value\": \"EC2-06\"}]
    }]
  }" \
  --tag-specifications "ResourceType=launch-template,Tags=[{Key=Lab,Value=EC2-06}]" \
  --query 'LaunchTemplate.LaunchTemplateId' --output text)

# ASG attached to the target group
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name "ec2-lab06-asg" \
  --launch-template "LaunchTemplateId=$LT_ID,Version=1" \
  --min-size 2 \
  --desired-capacity 2 \
  --max-size 6 \
  --target-group-arns "$TG_ARN" \
  --vpc-zone-identifier "$SUBNET_IDS" \
  --health-check-type ELB \
  --health-check-grace-period 120 \
  --tags "Key=Name,Value=ec2-lab06-asg,PropagateAtLaunch=false"

echo "ASG created and linked to ALB target group"
```

### Step 5 — Create SNS topic and subscribe your email

```bash
SNS_ARN=$(aws sns create-topic \
  --name ec2-lab06-scale-alerts \
  --tags Key=Lab,Value=EC2-06 \
  --query 'TopicArn' --output text)

echo "SNS Topic: $SNS_ARN"

# Subscribe your email (replace with real address)
aws sns subscribe \
  --topic-arn "$SNS_ARN" \
  --protocol email \
  --notification-endpoint "naveenramasamy11@gmail.com"

echo "Check your inbox and confirm the SNS subscription before proceeding"
```

### Step 6 — Create Lambda function that posts scale events to SNS

```bash
# Lambda execution role
cat > /tmp/lambda-trust.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": { "Service": "lambda.amazonaws.com" },
    "Action": "sts:AssumeRole"
  }]
}
EOF

LAMBDA_ROLE_ARN=$(aws iam create-role \
  --role-name ec2-lab06-lambda-role \
  --assume-role-policy-document file:///tmp/lambda-trust.json \
  --query 'Role.Arn' --output text)

aws iam attach-role-policy \
  --role-name ec2-lab06-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

aws iam attach-role-policy \
  --role-name ec2-lab06-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess

sleep 10  # IAM propagation

# Write Lambda code
mkdir -p /tmp/lambda_pkg

cat > /tmp/lambda_pkg/handler.py << PYEOF
import boto3, json, os, datetime

sns = boto3.client('sns')
SNS_ARN = os.environ['SNS_TOPIC_ARN']

def lambda_handler(event, context):
    alarm = json.loads(event['Records'][0]['Sns']['Message'])
    name  = alarm.get('AlarmName', 'Unknown')
    state = alarm.get('NewStateValue', 'Unknown')
    reason = alarm.get('NewStateReason', '')
    ts    = datetime.datetime.utcnow().isoformat()

    message = (
        f"EC2 Auto Scaling Alert\n"
        f"Alarm    : {name}\n"
        f"New State: {state}\n"
        f"Reason   : {reason}\n"
        f"Time     : {ts} UTC\n"
    )
    print(message)

    sns.publish(
        TopicArn=SNS_ARN,
        Subject=f"[EC2 Lab06] Alarm {state}: {name}",
        Message=message
    )
    return {"statusCode": 200, "body": "Notification sent"}
PYEOF

cd /tmp/lambda_pkg && zip -r /tmp/lab06-lambda.zip handler.py

LAMBDA_ARN=$(aws lambda create-function \
  --function-name ec2-lab06-scale-notifier \
  --runtime python3.12 \
  --role "$LAMBDA_ROLE_ARN" \
  --handler handler.lambda_handler \
  --zip-file fileb:///tmp/lab06-lambda.zip \
  --environment "Variables={SNS_TOPIC_ARN=$SNS_ARN}" \
  --timeout 30 \
  --tags Lab=EC2-06 \
  --query 'FunctionArn' --output text)

echo "Lambda: $LAMBDA_ARN"
```

### Step 7 — Create CloudWatch alarm and wire to Lambda

```bash
# CloudWatch Alarm: average CPU > 70 % for 2 evaluation periods of 1 min
aws cloudwatch put-metric-alarm \
  --alarm-name "ec2-lab06-cpu-high" \
  --alarm-description "ASG CPU over 70% — scale alert" \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions "Name=AutoScalingGroupName,Value=ec2-lab06-asg" \
  --statistic Average \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 70 \
  --comparison-operator GreaterThanThreshold \
  --alarm-actions "$LAMBDA_ARN" \
  --treat-missing-data notBreaching \
  --tags Key=Lab,Value=EC2-06

# Grant CloudWatch permission to invoke Lambda
aws lambda add-permission \
  --function-name ec2-lab06-scale-notifier \
  --statement-id cloudwatch-invoke \
  --action lambda:InvokeFunction \
  --principal lambda.alarms.cloudwatch.amazonaws.com \
  --source-arn "arn:aws:cloudwatch:${AWS_DEFAULT_REGION}:${ACCOUNT_ID}:alarm:ec2-lab06-cpu-high"

echo "CloudWatch alarm created and wired to Lambda"
```

### Step 8 — Test the end-to-end pipeline

```bash
# Wait for ALB to be active
aws elbv2 wait load-balancer-available --load-balancer-arns "$ALB_ARN"

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns "$ALB_ARN" \
  --query 'LoadBalancers[0].DNSName' --output text)

echo "ALB DNS: $ALB_DNS"

# Wait for instances to pass health checks (~2 min)
sleep 120

# Test the ALB (hit it multiple times to verify load balancing)
for i in {1..5}; do
  curl -s "http://$ALB_DNS" | grep "Instance:"
  sleep 2
done
```

Expected output (two different instance IDs alternating):
```
<h2>Lab 06 — Instance: i-0abc1234</h2>
<h2>Lab 06 — Instance: i-0def5678</h2>
<h2>Lab 06 — Instance: i-0abc1234</h2>
```

---

## Cleanup

```bash
# Delete ASG
aws autoscaling delete-auto-scaling-group \
  --auto-scaling-group-name "ec2-lab06-asg" --force-delete

# Delete ALB components
aws elbv2 delete-load-balancer --load-balancer-arn "$ALB_ARN"
sleep 10
aws elbv2 delete-target-group --target-group-arn "$TG_ARN"

# Delete CloudWatch alarm
aws cloudwatch delete-alarms --alarm-names "ec2-lab06-cpu-high"

# Delete Lambda
aws lambda delete-function --function-name ec2-lab06-scale-notifier

# Delete SNS
aws sns delete-topic --topic-arn "$SNS_ARN"

# Delete IAM
aws iam detach-role-policy --role-name ec2-lab06-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole
aws iam detach-role-policy --role-name ec2-lab06-lambda-role \
  --policy-arn arn:aws:iam::aws:policy/AmazonSNSFullAccess
aws iam delete-role --role-name ec2-lab06-lambda-role

# Delete launch template and security groups
aws ec2 delete-launch-template --launch-template-id "$LT_ID"
aws ec2 wait instances-terminated 2>/dev/null || true
aws ec2 delete-security-group --group-id "$EC2_SG" 2>/dev/null
aws ec2 delete-security-group --group-id "$ALB_SG" 2>/dev/null

echo "Cleanup complete"
```

---

## Key Takeaways

- EC2 SG should only accept traffic from ALB SG (not from 0.0.0.0/0) for a proper DMZ pattern
- ALB health check grace period (120s) gives user-data time to complete before health checks start
- Lambda can be subscribed to CloudWatch alarms via SNS (alarm → SNS → Lambda)
- Always add `aws lambda add-permission` after creating the function; missing this causes silent failures
- Test load balancing by hitting the ALB multiple times and verifying different instance IDs respond

---

➡️ Next: [Lab 07 — Production Patterns: Monitoring, Cost & Resilience](../07-production-patterns/README.md)
