# Lab 06 — CI/CD Pipeline: ECR → ECS Blue/Green via CodeDeploy 🔴

## Goal

Build a production-grade CI/CD pipeline: push a Docker image to ECR, trigger a CodeDeploy Blue/Green deployment to ECS, validate the new version on a test listener, then shift 100% of traffic — all automated with AWS native services.

---

## Concepts

### Pipeline Components

| Component | Role |
|-----------|------|
| **ECR** | Docker image registry — source of truth for container versions |
| **CodeBuild** | Build & push Docker image; generate `imagedefinitions.json` |
| **CodePipeline** | Orchestrates Source → Build → Deploy stages |
| **CodeDeploy** | Manages Blue/Green traffic shifting with automatic rollback |
| **ECS Service** | Target — `deploymentController: CODE_DEPLOY` |

### imagedefinitions.json vs imageDetail.json

| File | Use Case |
|------|----------|
| `imagedefinitions.json` | Rolling update deployments (ECS provider) |
| `imageDetail.json` | Blue/Green deployments (CodeDeploy provider) |

For this lab we use `imageDetail.json`:
```json
{"ImageURI":"123456789012.dkr.ecr.us-east-1.amazonaws.com/web-app:sha-abc123"}
```

---

## Architecture

```
  Developer: git push
       │
       ▼
  CodePipeline
  ┌──────────────────────────────────────────────────────┐
  │  Stage 1: Source                                      │
  │  S3 (or CodeCommit/GitHub) ──► Source artifact        │
  │                                                       │
  │  Stage 2: Build (CodeBuild)                           │
  │  ┌──────────────────────────────────────────────────┐ │
  │  │ docker build -t $ECR_REPO:$CODEBUILD_RESOLVED_.. │ │
  │  │ docker push $ECR_REPO:$IMAGE_TAG                  │ │
  │  │ echo imageDetail.json                             │ │
  │  └──────────────────────────────────────────────────┘ │
  │                                                       │
  │  Stage 3: Deploy (CodeDeploy)                         │
  │  Blue TG (current) ←── ALB ──► Green TG (new)         │
  │  Traffic: 0% → test hook → 100% on approval           │
  └──────────────────────────────────────────────────────┘
```

---

## Step-by-Step PoC

### Step 1 — Create ECR Repository

```bash
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export ECR_REPO=ecs-lab-web

aws ecr create-repository \
  --repository-name $ECR_REPO \
  --image-scanning-configuration scanOnPush=true \
  --encryption-configuration encryptionType=AES256 \
  --tags Key=Project,Value=ecs-lab \
  --output table

ECR_URI="${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPO}"
echo "ECR URI: $ECR_URI"
```

---

### Step 2 — Build and push a sample app image

```bash
# Create a trivial web app to simulate versioned deploys
mkdir -p /tmp/web-app && cd /tmp/web-app

cat > Dockerfile <<'EOF'
FROM nginx:alpine
RUN echo '<h1>Version 1.0 - Blue</h1>' > /usr/share/nginx/html/index.html
EXPOSE 80
EOF

# Authenticate to ECR
aws ecr get-login-password | docker login \
  --username AWS \
  --password-stdin "${ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com"

IMAGE_TAG="v1.0-$(date +%Y%m%d%H%M%S)"
docker build -t ${ECR_URI}:${IMAGE_TAG} .
docker tag  ${ECR_URI}:${IMAGE_TAG} ${ECR_URI}:latest
docker push ${ECR_URI}:${IMAGE_TAG}
docker push ${ECR_URI}:latest

echo "Pushed: ${ECR_URI}:${IMAGE_TAG}"
```

---

### Step 3 — Create CodeDeploy Application and Deployment Group

Blue/Green ECS deployments require the ECS service to have `deploymentController: CODE_DEPLOY`.

```bash
# CodeDeploy service role
aws iam create-role \
  --role-name AWSCodeDeployRoleForECS-lab \
  --assume-role-policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Principal":{"Service":"codedeploy.amazonaws.com"},
      "Action":"sts:AssumeRole"
    }]
  }' \
  --tags Key=Project,Value=ecs-lab \
  --output table

aws iam attach-role-policy \
  --role-name AWSCodeDeployRoleForECS-lab \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS

DEPLOY_ROLE_ARN=$(aws iam get-role \
  --role-name AWSCodeDeployRoleForECS-lab \
  --query Role.Arn --output text)

# Create CodeDeploy application
aws deploy create-application \
  --application-name ecs-lab-app \
  --compute-platform ECS \
  --tags Key=Project,Value=ecs-lab

echo "CodeDeploy application created."
```

---

### Step 4 — Create Blue and Green Target Groups

```bash
VPC_ID=$(aws ec2 describe-vpcs \
  --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text)

TG_BLUE=$(aws elbv2 create-target-group \
  --name ecs-lab-tg-blue \
  --protocol HTTP --port 80 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-path "/" \
  --tags Key=Project,Value=ecs-lab \
  --query "TargetGroups[0].TargetGroupArn" --output text)

TG_GREEN=$(aws elbv2 create-target-group \
  --name ecs-lab-tg-green \
  --protocol HTTP --port 80 \
  --vpc-id $VPC_ID --target-type ip \
  --health-check-path "/" \
  --tags Key=Project,Value=ecs-lab \
  --query "TargetGroups[0].TargetGroupArn" --output text)

echo "Blue TG:  $TG_BLUE"
echo "Green TG: $TG_GREEN"
```

---

### Step 5 — Create ECS Service with CODE_DEPLOY controller

```bash
SUBNET_A=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID \
  --query "Subnets[0].SubnetId" --output text)
SUBNET_B=$(aws ec2 describe-subnets \
  --filters Name=vpc-id,Values=$VPC_ID \
  --query "Subnets[1].SubnetId" --output text)

# Get the task SG created in Lab 03
TASK_SG=$(aws ec2 describe-security-groups \
  --filters Name=group-name,Values=ecs-task-sg Name=vpc-id,Values=$VPC_ID \
  --query "SecurityGroups[0].GroupId" --output text)

ALB_ARN=$(aws elbv2 describe-load-balancers \
  --names ecs-lab-alb \
  --query "LoadBalancers[0].LoadBalancerArn" --output text)

LISTENER_ARN=$(aws elbv2 describe-listeners \
  --load-balancer-arn $ALB_ARN \
  --query "Listeners[0].ListenerArn" --output text)

aws ecs create-service \
  --cluster lab-cluster \
  --service-name nginx-bg-service \
  --task-definition nginx-lab:2 \
  --desired-count 2 \
  --launch-type FARGATE \
  --deployment-controller type=CODE_DEPLOY \
  --health-check-grace-period-seconds 30 \
  --network-configuration "awsvpcConfiguration={
    subnets=[$SUBNET_A,$SUBNET_B],
    securityGroups=[$TASK_SG],
    assignPublicIp=DISABLED
  }" \
  --load-balancers "targetGroupArn=$TG_BLUE,containerName=nginx,containerPort=80" \
  --tags key=Project,value=ecs-lab \
  --output table
```

---

### Step 6 — Create CodeDeploy Deployment Group

```bash
cat > /tmp/codedeploy-dg.json <<EOF
{
  "applicationName": "ecs-lab-app",
  "deploymentGroupName": "ecs-lab-dg",
  "deploymentConfigName": "CodeDeployDefault.ECSAllAtOnce",
  "serviceRoleArn": "${DEPLOY_ROLE_ARN}",
  "ecsServices": [{
    "serviceName": "nginx-bg-service",
    "clusterName": "lab-cluster"
  }],
  "loadBalancerInfo": {
    "targetGroupPairInfoList": [{
      "targetGroups": [
        {"name": "ecs-lab-tg-blue"},
        {"name": "ecs-lab-tg-green"}
      ],
      "prodTrafficRoute": {"listenerArns": ["${LISTENER_ARN}"]},
      "testTrafficRoute": {}
    }]
  },
  "deploymentStyle": {
    "deploymentType": "BLUE_GREEN",
    "deploymentOption": "WITH_TRAFFIC_CONTROL"
  },
  "blueGreenDeploymentConfiguration": {
    "terminateBlueInstancesOnDeploymentSuccess": {
      "action": "TERMINATE",
      "terminationWaitTimeInMinutes": 5
    },
    "deploymentReadyOption": {
      "actionOnTimeout": "CONTINUE_DEPLOYMENT",
      "waitTimeInMinutes": 0
    }
  }
}
EOF

aws deploy create-deployment-group \
  --cli-input-json file:///tmp/codedeploy-dg.json \
  --tags Key=Project,Value=ecs-lab \
  --output table
```

---

### Step 7 — Trigger a Blue/Green deployment

```bash
# Create a new image (v2 - Green)
cat > /tmp/web-app/Dockerfile <<'EOF'
FROM nginx:alpine
RUN echo '<h1>Version 2.0 - Green</h1>' > /usr/share/nginx/html/index.html
EXPOSE 80
EOF

cd /tmp/web-app
IMAGE_TAG_V2="v2.0-$(date +%Y%m%d%H%M%S)"
docker build -t ${ECR_URI}:${IMAGE_TAG_V2} .
docker push ${ECR_URI}:${IMAGE_TAG_V2}

# Create AppSpec
cat > /tmp/appspec.json <<EOF
{
  "version": 0.0,
  "Resources": [{
    "TargetService": {
      "Type": "AWS::ECS::Service",
      "Properties": {
        "TaskDefinition": "arn:aws:ecs:${AWS_DEFAULT_REGION}:${ACCOUNT_ID}:task-definition/nginx-lab:2",
        "LoadBalancerInfo": {
          "ContainerName": "nginx",
          "ContainerPort": 80
        }
      }
    }
  }]
}
EOF

# Trigger deployment
DEPLOYMENT_ID=$(aws deploy create-deployment \
  --application-name ecs-lab-app \
  --deployment-group-name ecs-lab-dg \
  --revision '{"revisionType":"AppSpecContent","appSpecContent":{"content":'"$(cat /tmp/appspec.json | python3 -c "import sys,json; print(json.dumps(sys.stdin.read()))")"'}}' \
  --query deploymentId --output text)

echo "Deployment ID: $DEPLOYMENT_ID"

# Monitor progress
aws deploy get-deployment \
  --deployment-id $DEPLOYMENT_ID \
  --query "deploymentInfo.[status,deploymentOverview]" \
  --output table
```

---

## Cleanup

```bash
aws deploy delete-deployment-group \
  --application-name ecs-lab-app \
  --deployment-group-name ecs-lab-dg

aws deploy delete-application --application-name ecs-lab-app

aws ecs update-service --cluster lab-cluster --service nginx-bg-service --desired-count 0
aws ecs wait services-stable --cluster lab-cluster --services nginx-bg-service
aws ecs delete-service --cluster lab-cluster --service nginx-bg-service --force

aws elbv2 delete-target-group --target-group-arn $TG_BLUE
aws elbv2 delete-target-group --target-group-arn $TG_GREEN

aws ecr delete-repository --repository-name $ECR_REPO --force

aws iam detach-role-policy \
  --role-name AWSCodeDeployRoleForECS-lab \
  --policy-arn arn:aws:iam::aws:policy/AWSCodeDeployRoleForECS
aws iam delete-role --role-name AWSCodeDeployRoleForECS-lab

echo "Cleanup complete."
```

---

➡️ [Lab 07 → Production Patterns: Observability, Cost & Resilience](../07-production-patterns/README.md)
