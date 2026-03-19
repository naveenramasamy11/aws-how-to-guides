# Lab 05 — IAM Roles, Secrets & VPC Isolation 🔴

## Goal

Harden an ECS deployment: apply least-privilege IAM task roles, inject secrets from Secrets Manager (not environment variables), restrict network paths with security group rules and private subnets, and enable ECS Exec for break-glass debugging — all without giving containers shell access by default.

---

## Concepts

### ECS IAM Role Hierarchy

```
  AWS Account
  └── IAM Role: ecsTaskExecutionRole  (used by ECS control plane)
      ├── Pull image from ECR
      ├── Write logs to CloudWatch
      └── Read secrets from Secrets Manager (for injection)

  └── IAM Role: ecsTaskRole  (used by your application code)
      ├── s3:GetObject on arn:aws:s3:::my-bucket/*
      ├── dynamodb:PutItem on arn:aws:dynamodb:::table/my-table
      └── secretsmanager:GetSecretValue (if app fetches at runtime)
```

### Secrets Injection — Comparison

| Method | How | Security |
|--------|-----|---------|
| Plain `environment` | Value in task definition JSON (base64 in API) | Visible in console, CloudTrail |
| `secrets` (SSM/Secrets Manager ref) | ECS fetches at startup, injects as env var | Not in task definition; requires execution role |
| App fetches at runtime via SDK | App calls Secrets Manager directly | Most flexible; needs task role, not execution role |

Using `secrets` in `containerDefinitions` is the recommended middle ground — value never appears in the task definition.

### VPC Endpoint for ECS

To run Fargate tasks in **private subnets with no NAT gateway**, create VPC Interface Endpoints for:
- `com.amazonaws.<region>.ecr.dkr` — ECR image pull
- `com.amazonaws.<region>.ecr.api` — ECR API
- `com.amazonaws.<region>.logs` — CloudWatch Logs
- `com.amazonaws.<region>.secretsmanager` — Secrets Manager
- `com.amazonaws.<region>.ssm` — ECS Exec SSM

---

## Architecture

```
  Private Subnet (10.0.3.0/24)              VPC Endpoints
  ┌──────────────────────────┐    ───────►  ECR (dkr/api)
  │ ECS Task                 │             CloudWatch Logs
  │ ┌────────────────────┐   │    ───────►  Secrets Manager
  │ │ App Container       │   │    ───────►  SSM (ECS Exec)
  │ │ Task Role attached  │   │
  │ └────────────────────┘   │
  └──────────────────────────┘
       No IGW / NAT needed
  Security Group: allow inbound 8080 from ALB SG only
  Security Group: allow outbound 443 to VPC endpoints SG
```

---

## Step-by-Step PoC

### Step 1 — Create a least-privilege Task Role

```bash
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Create role
aws iam create-role \
  --role-name ecsTaskRole-nginx \
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

# Inline policy: read one S3 bucket (example least-privilege)
aws iam put-role-policy \
  --role-name ecsTaskRole-nginx \
  --policy-name s3-readonly-lab \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":["s3:GetObject","s3:ListBucket"],
      "Resource":[
        "arn:aws:s3:::ecs-lab-assets-'${ACCOUNT_ID}'",
        "arn:aws:s3:::ecs-lab-assets-'${ACCOUNT_ID}'/*"
      ]
    }]
  }'

echo "Task role created."
```

---

### Step 2 — Store a secret in Secrets Manager

```bash
# Create a secret (simulating a DB password)
SECRET_ARN=$(aws secretsmanager create-secret \
  --name /ecs-lab/db-password \
  --description "ECS lab database password" \
  --secret-string '{"password":"S3cur3P@ss!","username":"appuser"}' \
  --tags Key=Project,Value=ecs-lab \
  --query ARN --output text)

echo "Secret ARN: $SECRET_ARN"
```

---

### Step 3 — Grant execution role permission to read the secret

```bash
# Inline policy on the execution role
aws iam put-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-name secrets-manager-ecs-lab \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":[
        "secretsmanager:GetSecretValue",
        "kms:Decrypt"
      ],
      "Resource":"'${SECRET_ARN}'"
    }]
  }'

echo "Execution role updated."
```

---

### Step 4 — Register hardened task definition with secret injection

```bash
cat > /tmp/nginx-secure-task.json <<'EOF'
{
  "family": "nginx-secure",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "EXEC_ROLE_PLACEHOLDER",
  "taskRoleArn":      "TASK_ROLE_PLACEHOLDER",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:alpine",
      "portMappings": [{"containerPort": 80, "protocol": "tcp"}],
      "essential": true,
      "secrets": [
        {
          "name": "DB_PASSWORD",
          "valueFrom": "SECRET_ARN_PLACEHOLDER:password::"
        }
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group":         "/ecs/nginx-lab",
          "awslogs-region":        "us-east-1",
          "awslogs-stream-prefix": "ecs-secure"
        }
      }
    }
  ]
}
EOF

EXEC_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole"
TASK_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskRole-nginx"

sed -i \
  -e "s|EXEC_ROLE_PLACEHOLDER|${EXEC_ROLE_ARN}|" \
  -e "s|TASK_ROLE_PLACEHOLDER|${TASK_ROLE_ARN}|" \
  -e "s|SECRET_ARN_PLACEHOLDER|${SECRET_ARN}|" \
  /tmp/nginx-secure-task.json

aws ecs register-task-definition \
  --cli-input-json file:///tmp/nginx-secure-task.json \
  --tags key=Project,value=ecs-lab \
  --query "taskDefinition.[family,revision,status]" \
  --output table
```

---

### Step 5 — Enable ECS Exec on the service

ECS Exec uses SSM Session Manager to give you an interactive shell in a running container — no SSH, no exposed ports needed.

```bash
# Update service to enable ECS Exec
aws ecs update-service \
  --cluster lab-cluster \
  --service nginx-service \
  --task-definition nginx-secure:1 \
  --enable-execute-command \
  --force-new-deployment \
  --output table

# Grant the task role SSM permissions for ECS Exec
aws iam put-role-policy \
  --role-name ecsTaskRole-nginx \
  --policy-name ecs-exec-ssm \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":[
        "ssmmessages:CreateControlChannel",
        "ssmmessages:CreateDataChannel",
        "ssmmessages:OpenControlChannel",
        "ssmmessages:OpenDataChannel"
      ],
      "Resource":"*"
    }]
  }'
```

---

### Step 6 — Exec into a running container

```bash
# Wait for tasks to be RUNNING
aws ecs wait services-stable --cluster lab-cluster --services nginx-service

# Get a running task ARN
TASK_ARN=$(aws ecs list-tasks \
  --cluster lab-cluster \
  --service-name nginx-service \
  --desired-status RUNNING \
  --query "taskArns[0]" --output text)

# Exec into the nginx container
aws ecs execute-command \
  --cluster lab-cluster \
  --task $TASK_ARN \
  --container nginx \
  --interactive \
  --command "/bin/sh"

# Inside the container — verify the injected secret (as env var)
# echo $DB_PASSWORD
# exit
```

---

### Step 7 — Verify no sensitive data in task definition

```bash
aws ecs describe-task-definition \
  --task-definition nginx-secure:1 \
  --query "taskDefinition.containerDefinitions[0].secrets" \
  --output table
```

**Expected output — only the ARN reference, not the actual value:**
```
+-------------+----------------------------------------------+
| DB_PASSWORD | arn:aws:secretsmanager:...:password::        |
+-------------+----------------------------------------------+
```

---

## Cleanup

```bash
# Delete the secret
aws secretsmanager delete-secret \
  --secret-id /ecs-lab/db-password \
  --force-delete-without-recovery

# Remove inline policies
aws iam delete-role-policy --role-name ecsTaskRole-nginx --policy-name s3-readonly-lab
aws iam delete-role-policy --role-name ecsTaskRole-nginx --policy-name ecs-exec-ssm
aws iam delete-role-policy --role-name ecsTaskExecutionRole --policy-name secrets-manager-ecs-lab

# Delete task role
aws iam delete-role --role-name ecsTaskRole-nginx

echo "Cleanup complete."
```

---

➡️ [Lab 06 → CI/CD Integration Pattern: ECR → ECS via CodeDeploy](../06-integration-patterns/README.md)
