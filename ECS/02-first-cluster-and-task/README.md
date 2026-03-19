# Lab 02 — First Cluster, Task Definition & Run Task 🟢

## Goal

Create an ECS cluster, register a task definition using the public `nginx` image, run a one-off Fargate task, and verify the container is reachable — all using AWS CLI.

---

## Prerequisites

- AWS CLI v2 configured (`aws configure`)
- An existing VPC with at least one public subnet (a default VPC works)
- IAM permissions: `ecs:*`, `ecr:*`, `iam:PassRole`, `logs:*`

---

## Concepts

### Task Definition Revisions

Every `register-task-definition` call creates a new **revision** (`:1`, `:2`, …). ECS never deletes old revisions — you deregister them explicitly when no longer needed.

### Execution Role vs Task Role

| Role | Purpose |
|------|---------|
| **Execution Role** (`ecsTaskExecutionRole`) | ECS pulls the image from ECR and writes CloudWatch logs on your behalf |
| **Task Role** | Your application code's IAM identity — grants access to S3, DynamoDB, SQS, etc. |

---

## Architecture

```
  AWS CLI
     │
     ├──► ECS: CreateCluster ──► Cluster "lab-cluster"
     │
     ├──► IAM: CreateRole ──► ecsTaskExecutionRole
     │
     ├──► CloudWatch Logs: CreateLogGroup ──► /ecs/nginx-lab
     │
     ├──► ECS: RegisterTaskDefinition ──► nginx-lab:1
     │
     └──► ECS: RunTask ──► Fargate task (nginx:alpine)
                               │
                         VPC / Public Subnet
                         Public IP assigned
                         Port 80 reachable
```

---

## Step-by-Step PoC

### Step 1 — Set environment variables

```bash
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text)
export SUBNET_ID=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID Name=defaultForAz,Values=true \
  --query "Subnets[0].SubnetId" --output text)

echo "Account : $ACCOUNT_ID"
echo "VPC     : $VPC_ID"
echo "Subnet  : $SUBNET_ID"
```

**Expected output:**
```
Account : 123456789012
VPC     : vpc-0abc12345
Subnet  : subnet-0abc12345
```

---

### Step 2 — Create a security group that allows inbound HTTP

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name ecs-lab-sg \
  --description "ECS lab - allow HTTP" \
  --vpc-id $VPC_ID \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Project,Value=ecs-lab}]" \
  --query GroupId --output text)

aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 80 \
  --cidr 0.0.0.0/0

echo "Security Group: $SG_ID"
```

---

### Step 3 — Create the ECS Task Execution Role

```bash
# Create the role
aws iam create-role \
  --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"Service":"ecs-tasks.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }]
  }' \
  --tags Key=Project,Value=ecs-lab \
  --output table

# Attach the AWS-managed policy
aws iam attach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

echo "Execution role created."
```

> If the role already exists you'll see an `EntityAlreadyExists` error — that's fine, just continue.

---

### Step 4 — Create CloudWatch Log Group

```bash
aws logs create-log-group \
  --log-group-name /ecs/nginx-lab \
  --tags Project=ecs-lab

echo "Log group /ecs/nginx-lab created."
```

---

### Step 5 — Create the ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name lab-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1 \
  --tags key=Project,value=ecs-lab \
  --output table
```

**Expected output:**
```
--------------------------------------
|          CreateCluster             |
+--------------------+---------------+
|  clusterArn        | arn:aws:ecs:...|
|  clusterName       | lab-cluster   |
|  status            | ACTIVE        |
+--------------------+---------------+
```

---

### Step 6 — Register the Task Definition

```bash
cat > /tmp/nginx-task-def.json <<'EOF'
{
  "family": "nginx-lab",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "EXECUTION_ROLE_ARN_PLACEHOLDER",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:alpine",
      "portMappings": [{"containerPort": 80, "protocol": "tcp"}],
      "essential": true,
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group":         "/ecs/nginx-lab",
          "awslogs-region":        "us-east-1",
          "awslogs-stream-prefix": "ecs"
        }
      }
    }
  ]
}
EOF

# Inject the real execution role ARN
EXEC_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole"
sed -i "s|EXECUTION_ROLE_ARN_PLACEHOLDER|${EXEC_ROLE_ARN}|" /tmp/nginx-task-def.json

aws ecs register-task-definition \
  --cli-input-json file:///tmp/nginx-task-def.json \
  --tags key=Project,value=ecs-lab \
  --query "taskDefinition.[family,revision,status]" \
  --output table
```

**Expected output:**
```
----------------------------
|  RegisterTaskDefinition  |
+----------+----+-----------+
|  nginx-lab | 1 | ACTIVE   |
+----------+----+-----------+
```

---

### Step 7 — Run a Fargate Task

```bash
TASK_ARN=$(aws ecs run-task \
  --cluster lab-cluster \
  --launch-type FARGATE \
  --task-definition nginx-lab:1 \
  --count 1 \
  --network-configuration "awsvpcConfiguration={
    subnets=[$SUBNET_ID],
    securityGroups=[$SG_ID],
    assignPublicIp=ENABLED
  }" \
  --tags key=Project,value=ecs-lab \
  --query "tasks[0].taskArn" --output text)

echo "Task ARN: $TASK_ARN"
```

---

### Step 8 — Wait for RUNNING state and get the public IP

```bash
# Wait until task is RUNNING (up to 90 seconds)
aws ecs wait tasks-running \
  --cluster lab-cluster \
  --tasks $TASK_ARN

# Get the ENI attached to the task
ENI_ID=$(aws ecs describe-tasks \
  --cluster lab-cluster \
  --tasks $TASK_ARN \
  --query "tasks[0].attachments[0].details[?name=='networkInterfaceId'].value" \
  --output text)

# Get public IP from the ENI
PUBLIC_IP=$(aws ec2 describe-network-interfaces \
  --network-interface-ids $ENI_ID \
  --query "NetworkInterfaces[0].Association.PublicIp" \
  --output text)

echo "Task is RUNNING at: http://$PUBLIC_IP"
```

---

### Step 9 — Verify the container responds

```bash
curl -s http://$PUBLIC_IP | grep -o "<title>.*</title>"
```

**Expected output:**
```html
<title>Welcome to nginx!</title>
```

---

### Step 10 — Inspect logs

```bash
# List log streams for this task
TASK_SHORT=$(echo $TASK_ARN | awk -F'/' '{print $NF}')

aws logs describe-log-streams \
  --log-group-name /ecs/nginx-lab \
  --query "logStreams[*].logStreamName" \
  --output table

# Tail recent log events
aws logs get-log-events \
  --log-group-name /ecs/nginx-lab \
  --log-stream-name "ecs/nginx/${TASK_SHORT}" \
  --limit 20 \
  --query "events[*].[timestamp,message]" \
  --output table
```

---

## Cleanup

```bash
# Stop the task
aws ecs stop-task \
  --cluster lab-cluster \
  --task $TASK_ARN \
  --reason "Lab cleanup"

# Delete cluster (only if no active services)
aws ecs delete-cluster --cluster lab-cluster --output table

# Delete log group
aws logs delete-log-group --log-group-name /ecs/nginx-lab

# Delete security group (wait a minute for ENI to detach first)
sleep 60
aws ec2 delete-security-group --group-id $SG_ID

echo "Cleanup complete."
```

---

➡️ [Lab 03 → Fargate Service with ALB](../03-fargate-service/README.md)
