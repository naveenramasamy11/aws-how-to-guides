# Lab 03 — Fargate Service with Application Load Balancer 🟡

## Goal

Deploy a long-running ECS **Service** on Fargate backed by an Application Load Balancer (ALB). Traffic arrives on port 443 (HTTP redirect for simplicity in the lab) and is round-robin balanced across multiple tasks in different Availability Zones.

---

## Concepts

### ECS Service vs One-off Task

| | Task (run-task) | Service |
|--|----------------|---------|
| Lifecycle | Runs once, then exits | Continuously maintained |
| Replacement | Not replaced if it fails | Scheduler replaces unhealthy tasks |
| Load Balancer | Not supported | Integrated via Target Group |
| Use case | Batch, migrations, one-off jobs | APIs, web servers, microservices |

### ALB Target Group — IP Mode

ECS Fargate uses **IP-based** target groups (not instance-based). The ALB registers each task's private IP address as a target. When a task is replaced, ECS deregisters the old IP and registers the new one automatically.

### Health Check Grace Period

When a new task starts, the ALB health checks may fail while the application warms up. `healthCheckGracePeriodSeconds` tells ECS how long to ignore ALB health check failures before marking the task unhealthy.

---

## Architecture

```
  Internet
     │ (HTTPS/HTTP :80)
     ▼
  Application Load Balancer
  ┌───────────────────────────────────────┐
  │  Listener :80 → forward → TG          │
  │  Target Group: ecs-lab-tg (IP mode)   │
  └────────┬─────────────┬────────────────┘
           │             │
  ┌────────▼──┐   ┌──────▼──────┐
  │ Task AZ-a │   │  Task AZ-b  │
  │10.0.1.x:80│   │10.0.2.x:80  │
  └───────────┘   └─────────────┘
  ECS Cluster "lab-cluster" — Fargate
```

---

## Prerequisites

Complete Lab 02 so that the following exist:
- `lab-cluster` ECS cluster
- `ecsTaskExecutionRole` IAM role
- `/ecs/nginx-lab` CloudWatch log group
- A VPC with at least 2 public subnets in different AZs

---

## Step-by-Step PoC

### Step 1 — Collect networking details

```bash
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text)

# Get 2 public subnets in different AZs
SUBNETS=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID Name=defaultForAz,Values=true \
  --query "Subnets[*].SubnetId" --output text | tr '\t' ' ')
SUBNET_A=$(echo $SUBNETS | awk '{print $1}')
SUBNET_B=$(echo $SUBNETS | awk '{print $2}')

echo "Subnet A: $SUBNET_A  Subnet B: $SUBNET_B"
```

---

### Step 2 — Create security groups

```bash
# ALB security group — allow inbound HTTP from anywhere
ALB_SG=$(aws ec2 create-security-group \
  --group-name ecs-alb-sg \
  --description "ECS ALB - allow HTTP" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=ecs-lab}]" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $ALB_SG --protocol tcp --port 80 --cidr 0.0.0.0/0

# Task security group — allow inbound from ALB only
TASK_SG=$(aws ec2 create-security-group \
  --group-name ecs-task-sg \
  --description "ECS Tasks - allow from ALB" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=ecs-lab}]" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $TASK_SG \
  --protocol tcp \
  --port 80 \
  --source-group $ALB_SG

echo "ALB SG: $ALB_SG   Task SG: $TASK_SG"
```

---

### Step 3 — Create the Application Load Balancer

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name ecs-lab-alb \
  --subnets $SUBNET_A $SUBNET_B \
  --security-groups $ALB_SG \
  --scheme internet-facing \
  --type application \
  --ip-address-type ipv4 \
  --tags Key=Project,Value=ecs-lab \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)

ALB_DNS=$(aws elbv2 describe-load-balancers \
  --load-balancer-arns $ALB_ARN \
  --query "LoadBalancers[0].DNSName" --output text)

echo "ALB ARN: $ALB_ARN"
echo "ALB DNS: $ALB_DNS"
```

---

### Step 4 — Create Target Group (IP mode)

```bash
TG_ARN=$(aws elbv2 create-target-group \
  --name ecs-lab-tg \
  --protocol HTTP \
  --port 80 \
  --vpc-id $VPC_ID \
  --target-type ip \
  --health-check-protocol HTTP \
  --health-check-path "/" \
  --health-check-interval-seconds 15 \
  --healthy-threshold-count 2 \
  --unhealthy-threshold-count 2 \
  --tags Key=Project,Value=ecs-lab \
  --query "TargetGroups[0].TargetGroupArn" --output text)

echo "Target Group ARN: $TG_ARN"
```

---

### Step 5 — Create ALB Listener

```bash
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP \
  --port 80 \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --tags Key=Project,Value=ecs-lab \
  --output table
```

---

### Step 6 — Register Task Definition (same nginx-lab, or reuse from Lab 02)

```bash
# Confirm the task definition already exists
aws ecs describe-task-definition \
  --task-definition nginx-lab \
  --query "taskDefinition.[family,revision,status]" \
  --output table
```

If it doesn't exist, run the registration from Lab 02 Step 6 first.

---

### Step 7 — Create the ECS Service

```bash
EXEC_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole"

aws ecs create-service \
  --cluster lab-cluster \
  --service-name nginx-service \
  --task-definition nginx-lab:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --health-check-grace-period-seconds 30 \
  --network-configuration "awsvpcConfiguration={
    subnets=[$SUBNET_A,$SUBNET_B],
    securityGroups=[$TASK_SG],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=nginx,containerPort=80" \
  --scheduling-strategy REPLICA \
  --deployment-configuration "minimumHealthyPercent=100,maximumPercent=200" \
  --tags key=Project,value=ecs-lab \
  --output table
```

**Expected output (abridged):**
```
--------------------------------------
|          CreateService             |
+-------------------+----------------+
|  clusterArn       | arn:aws:ecs:...|
|  serviceName      | nginx-service  |
|  desiredCount     | 2              |
|  status           | ACTIVE         |
+-------------------+----------------+
```

---

### Step 8 — Monitor service stabilisation

```bash
# Wait until the service reaches steady state
aws ecs wait services-stable \
  --cluster lab-cluster \
  --services nginx-service

# Verify running tasks
aws ecs describe-services \
  --cluster lab-cluster \
  --services nginx-service \
  --query "services[0].[serviceName,desiredCount,runningCount,pendingCount]" \
  --output table
```

**Expected output:**
```
-------------------------------------------------
|          DescribeServices                     |
+---------------+-------+--------+--------------+
| nginx-service |   2   |   2    |      0       |
+---------------+-------+--------+--------------+
```

---

### Step 9 — Test the ALB endpoint

```bash
# Wait ~60 s for ALB to become healthy
sleep 60
curl -s http://$ALB_DNS | grep -o "<title>.*</title>"
```

**Expected output:**
```html
<title>Welcome to nginx!</title>
```

---

### Step 10 — Scale the service

```bash
aws ecs update-service \
  --cluster lab-cluster \
  --service nginx-service \
  --desired-count 4 \
  --output table

aws ecs wait services-stable \
  --cluster lab-cluster \
  --services nginx-service

aws ecs describe-services \
  --cluster lab-cluster \
  --services nginx-service \
  --query "services[0].[desiredCount,runningCount]" \
  --output table
```

---

## Cleanup

```bash
# Scale to 0 first
aws ecs update-service \
  --cluster lab-cluster \
  --service nginx-service \
  --desired-count 0

aws ecs wait services-stable \
  --cluster lab-cluster \
  --services nginx-service

# Delete service
aws ecs delete-service \
  --cluster lab-cluster \
  --service nginx-service \
  --force

# Delete ALB listener, target group, and ALB
LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn $ALB_ARN \
  --query "Listeners[0].ListenerArn" --output text)
aws elbv2 delete-listener --listener-arn $LISTENER_ARN
aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN
sleep 30
aws elbv2 delete-target-group --target-group-arn $TG_ARN

# Delete security groups
aws ec2 delete-security-group --group-id $TASK_SG
aws ec2 delete-security-group --group-id $ALB_SG

echo "Cleanup complete."
```

---

➡️ [Lab 04 → Rolling Updates, Blue/Green & Auto Scaling](../04-advanced-deployments/README.md)
