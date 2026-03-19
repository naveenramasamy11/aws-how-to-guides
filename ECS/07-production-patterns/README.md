# Lab 07 — Production Patterns: Observability, Cost Optimisation & Resilience 🔴

## Goal

Operate ECS at production scale: build a Container Insights dashboard, set CloudWatch alarms with SNS notifications, configure FARGATE_SPOT with graceful draining, implement structured logging with FireLens/Fluent Bit, and use Capacity Provider strategies for maximum cost efficiency.

---

## Concepts

### Container Insights

ECS Container Insights is a CloudWatch feature that collects CPU, memory, network, and disk metrics at the **cluster, service, and task** level without any manual metric publishing. It also aggregates performance logs for individual containers.

| Metric | Namespace | Description |
|--------|-----------|-------------|
| `CpuUtilized` | ECS/ContainerInsights | vCPU consumed by all tasks in service |
| `MemoryUtilized` | ECS/ContainerInsights | Memory consumed (MB) |
| `RunningTaskCount` | ECS/ContainerInsights | Current running tasks |
| `PendingTaskCount` | ECS/ContainerInsights | Tasks not yet running |
| `TaskCount` | ECS/ContainerInsights | Desired vs actual |

### FireLens — Structured Log Routing

FireLens is an ECS log router sidecar (Fluent Bit or Fluentd) that intercepts container stdout/stderr and forwards to multiple destinations simultaneously: CloudWatch, S3, OpenSearch, Datadog, Splunk, etc.

```
  App Container stdout
         │
         ▼
  FireLens Sidecar (Fluent Bit)
  ├──► CloudWatch Logs (hot storage)
  ├──► S3 (cold/archive)
  └──► OpenSearch (search/analytics)
```

### FARGATE_SPOT Draining

When AWS reclaims a Spot task, it sends a **SIGTERM** 120 seconds before termination. Your app should:
1. Catch SIGTERM — stop accepting new requests
2. Drain in-flight requests
3. Exit cleanly within 120 s

ECS also sends a task state change notification to EventBridge so you can automate draining logic.

---

## Architecture — Production ECS Cluster

```
  ┌──────────────────────────────────────────────────────────────────┐
  │  ECS Cluster — Capacity Provider Strategy                         │
  │  FARGATE weight=1 (base) + FARGATE_SPOT weight=4 (savings)       │
  │                                                                    │
  │  ┌─────────────────┐   ┌─────────────────┐   ┌────────────────┐  │
  │  │  Task (on-demand)│   │Task (Spot) x 3  │   │  FireLens     │  │
  │  │  App Container   │   │  App Container  │   │  Sidecar      │  │
  │  │  FireLens sidecar│   │  FireLens sidecar   │  (Fluent Bit) │  │
  │  └─────────────────┘   └─────────────────┘   └────────────────┘  │
  └──────────────────────────────────────────────────────────────────┘
           │ metrics                  │ logs
           ▼                          ▼
  CloudWatch Container Insights    CloudWatch Logs + S3
           │
  CloudWatch Alarms ──► SNS ──► PagerDuty/Slack
```

---

## Step-by-Step PoC

### Step 1 — Enable Container Insights on the Cluster

```bash
export AWS_DEFAULT_REGION=us-east-1

aws ecs update-cluster-settings \
  --cluster lab-cluster \
  --settings name=containerInsights,value=enabled \
  --output table

# Verify
aws ecs describe-clusters \
  --clusters lab-cluster \
  --query "clusters[0].settings" \
  --output table
```

**Expected output:**
```
+--------------------+---------+
|  containerInsights |  enabled |
+--------------------+---------+
```

---

### Step 2 — Create CloudWatch Alarms for the Service

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Alarm: CPU utilisation > 80% for 5 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-nginx-service-HighCPU" \
  --alarm-description "ECS nginx-service CPU > 80%" \
  --metric-name CpuUtilized \
  --namespace ECS/ContainerInsights \
  --dimensions \
    Name=ClusterName,Value=lab-cluster \
    Name=ServiceName,Value=nginx-service \
  --statistic Average \
  --period 60 \
  --evaluation-periods 5 \
  --threshold 80 \
  --comparison-operator GreaterThanThreshold \
  --treat-missing-data notBreaching \
  --tags Key=Project,Value=ecs-lab

# Alarm: Running task count drops below 1
aws cloudwatch put-metric-alarm \
  --alarm-name "ECS-nginx-service-NoRunningTasks" \
  --alarm-description "ECS nginx-service has 0 running tasks" \
  --metric-name RunningTaskCount \
  --namespace ECS/ContainerInsights \
  --dimensions \
    Name=ClusterName,Value=lab-cluster \
    Name=ServiceName,Value=nginx-service \
  --statistic Minimum \
  --period 60 \
  --evaluation-periods 2 \
  --threshold 1 \
  --comparison-operator LessThanThreshold \
  --treat-missing-data breaching \
  --tags Key=Project,Value=ecs-lab

echo "Alarms created."
```

---

### Step 3 — Set up SNS Notifications for Alarms

```bash
# Create SNS topic
TOPIC_ARN=$(aws sns create-topic \
  --name ecs-lab-alerts \
  --tags Key=Project,Value=ecs-lab \
  --query TopicArn --output text)

# Subscribe your email
aws sns subscribe \
  --topic-arn $TOPIC_ARN \
  --protocol email \
  --notification-endpoint naveenramasamy11@gmail.com

# Attach SNS to both alarms
for ALARM in "ECS-nginx-service-HighCPU" "ECS-nginx-service-NoRunningTasks"; do
  aws cloudwatch put-metric-alarm \
    --alarm-name $ALARM \
    --alarm-actions $TOPIC_ARN \
    --ok-actions $TOPIC_ARN
done

echo "SNS notifications configured. Check your email to confirm the subscription."
```

---

### Step 4 — Configure FARGATE_SPOT with graceful draining

```bash
# Update cluster default capacity provider strategy — 80% Spot, 20% on-demand
aws ecs put-cluster-capacity-providers \
  --cluster lab-cluster \
  --capacity-providers FARGATE FARGATE_SPOT \
  --default-capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=1 \
    capacityProvider=FARGATE_SPOT,weight=4 \
  --output table

# Update service to use the cluster default strategy
aws ecs update-service \
  --cluster lab-cluster \
  --service nginx-service \
  --capacity-provider-strategy \
    capacityProvider=FARGATE,weight=1,base=1 \
    capacityProvider=FARGATE_SPOT,weight=4 \
  --force-new-deployment \
  --output table
```

---

### Step 5 — FireLens structured logging task definition

```bash
cat > /tmp/nginx-firelens-task.json <<EOF
{
  "family": "nginx-firelens",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole",
  "containerDefinitions": [
    {
      "name": "log_router",
      "image": "amazon/aws-for-fluent-bit:stable",
      "essential": true,
      "firelensConfiguration": {
        "type": "fluentbit",
        "options": {
          "enable-ecs-log-metadata": "true"
        }
      },
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/firelens-router",
          "awslogs-region": "${AWS_DEFAULT_REGION}",
          "awslogs-stream-prefix": "firelens"
        }
      },
      "memoryReservation": 50
    },
    {
      "name": "nginx",
      "image": "nginx:alpine",
      "essential": true,
      "portMappings": [{"containerPort": 80, "protocol": "tcp"}],
      "dependsOn": [{"containerName": "log_router", "condition": "START"}],
      "logConfiguration": {
        "logDriver": "awsfirelens",
        "options": {
          "Name":           "cloudwatch_logs",
          "region":         "${AWS_DEFAULT_REGION}",
          "log_group_name": "/ecs/nginx-firelens",
          "log_stream_prefix": "from-firelens/"
        }
      }
    }
  ]
}
EOF

aws logs create-log-group --log-group-name /ecs/firelens-router \
  --tags Project=ecs-lab 2>/dev/null || true
aws logs create-log-group --log-group-name /ecs/nginx-firelens \
  --tags Project=ecs-lab 2>/dev/null || true

aws ecs register-task-definition \
  --cli-input-json file:///tmp/nginx-firelens-task.json \
  --tags key=Project,value=ecs-lab \
  --query "taskDefinition.[family,revision,status]" \
  --output table
```

---

### Step 6 — View Container Insights metrics in CloudWatch

```bash
# List available ECS metrics in Container Insights
aws cloudwatch list-metrics \
  --namespace ECS/ContainerInsights \
  --dimensions Name=ClusterName,Value=lab-cluster \
  --query "Metrics[*].[MetricName,Dimensions]" \
  --output table | head -50

# Get CPU utilised over the last 30 minutes
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name CpuUtilized \
  --dimensions \
    Name=ClusterName,Value=lab-cluster \
    Name=ServiceName,Value=nginx-service \
  --start-time $(date -u -d "30 minutes ago" +%Y-%m-%dT%H:%M:%SZ 2>/dev/null || \
                  date -u -v-30M +%Y-%m-%dT%H:%M:%SZ) \
  --end-time   $(date -u +%Y-%m-%dT%H:%M:%SZ) \
  --period 60 \
  --statistics Average Maximum \
  --query "Datapoints[*].[Timestamp,Average,Maximum]" \
  --output table
```

---

### Step 7 — Cost estimation and optimisation report

```bash
# List all tasks and their launch types to identify Spot vs on-demand split
aws ecs list-tasks \
  --cluster lab-cluster \
  --output text | \
xargs -I{} aws ecs describe-tasks \
  --cluster lab-cluster \
  --tasks {} \
  --query "tasks[*].[taskArn,launchType,capacityProviderName,cpu,memory,createdAt]" \
  --output table 2>/dev/null || \
aws ecs describe-tasks \
  --cluster lab-cluster \
  --tasks $(aws ecs list-tasks --cluster lab-cluster --query "taskArns" --output text) \
  --query "tasks[*].[taskArn,capacityProviderName,cpu,memory]" \
  --output table
```

---

## Production Best Practices Summary

| Practice | Implementation |
|----------|---------------|
| Right-size tasks | Start with 0.25 vCPU / 0.5 GB; use Container Insights to tune |
| Use FARGATE_SPOT for stateless workloads | 70% cost reduction; ensure graceful SIGTERM handling |
| Private subnets + VPC endpoints | Eliminate NAT gateway data charges (~$0.045/GB) |
| ECR lifecycle policies | Delete untagged images older than 30 days automatically |
| Task count alarms | Alert on `RunningTaskCount < desiredCount` |
| Deployment circuit breaker | Auto-rollback on failed deployments |
| Log retention policy | Set CloudWatch log group retention (e.g. 30 days) to control costs |

```bash
# Enable deployment circuit breaker (auto-rollback on failure)
aws ecs update-service \
  --cluster lab-cluster \
  --service nginx-service \
  --deployment-configuration \
    "deploymentCircuitBreaker={enable=true,rollback=true},minimumHealthyPercent=100,maximumPercent=200" \
  --output table

# Set log group retention
aws logs put-retention-policy \
  --log-group-name /ecs/nginx-lab \
  --retention-in-days 30

# ECR lifecycle policy — delete untagged images after 1 day, keep only latest 10 tagged
aws ecr put-lifecycle-policy \
  --repository-name ecs-lab-web \
  --lifecycle-policy-text '{
    "rules":[
      {"rulePriority":1,"description":"Remove untagged after 1 day",
       "selection":{"tagStatus":"untagged","countType":"sinceImagePushed","countUnit":"days","countNumber":1},
       "action":{"type":"expire"}},
      {"rulePriority":2,"description":"Keep 10 tagged images",
       "selection":{"tagStatus":"tagged","tagPrefixList":["v"],"countType":"imageCountMoreThan","countNumber":10},
       "action":{"type":"expire"}}
    ]
  }' 2>/dev/null && echo "ECR lifecycle policy applied." || echo "ECR repo may not exist — skip."
```

---

## Full Cleanup — All ECS Lab Resources

```bash
# Services
for SVC in nginx-service nginx-bg-service; do
  aws ecs update-service --cluster lab-cluster --service $SVC --desired-count 0 2>/dev/null || true
  aws ecs wait services-stable --cluster lab-cluster --services $SVC 2>/dev/null || true
  aws ecs delete-service --cluster lab-cluster --service $SVC --force 2>/dev/null || true
done

# Cluster
aws ecs delete-cluster --cluster lab-cluster 2>/dev/null || true

# ALB
ALB_ARN=$(aws elbv2 describe-load-balancers --names ecs-lab-alb \
  --query "LoadBalancers[0].LoadBalancerArn" --output text 2>/dev/null)
if [ "$ALB_ARN" != "None" ] && [ -n "$ALB_ARN" ]; then
  LISTENER=$(aws elbv2 describe-listeners --load-balancer-arn $ALB_ARN \
    --query "Listeners[0].ListenerArn" --output text 2>/dev/null)
  aws elbv2 delete-listener --listener-arn $LISTENER 2>/dev/null || true
  aws elbv2 delete-load-balancer --load-balancer-arn $ALB_ARN 2>/dev/null || true
  sleep 30
fi

for TG_NAME in ecs-lab-tg ecs-lab-tg-blue ecs-lab-tg-green; do
  TG=$(aws elbv2 describe-target-groups --names $TG_NAME \
    --query "TargetGroups[0].TargetGroupArn" --output text 2>/dev/null)
  [ -n "$TG" ] && aws elbv2 delete-target-group --target-group-arn $TG 2>/dev/null || true
done

# CloudWatch
for LG in /ecs/nginx-lab /ecs/nginx-firelens /ecs/firelens-router; do
  aws logs delete-log-group --log-group-name $LG 2>/dev/null || true
done
for ALARM in "ECS-nginx-service-HighCPU" "ECS-nginx-service-NoRunningTasks"; do
  aws cloudwatch delete-alarms --alarm-names "$ALARM" 2>/dev/null || true
done

# SNS
SNS_ARN=$(aws sns list-topics --query "Topics[?contains(TopicArn,'ecs-lab-alerts')].TopicArn" \
  --output text 2>/dev/null)
[ -n "$SNS_ARN" ] && aws sns delete-topic --topic-arn $SNS_ARN 2>/dev/null || true

# IAM
aws iam detach-role-policy \
  --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy 2>/dev/null || true
aws iam delete-role-policy --role-name ecsTaskExecutionRole \
  --policy-name secrets-manager-ecs-lab 2>/dev/null || true
aws iam delete-role --role-name ecsTaskExecutionRole 2>/dev/null || true

# Security groups
VPC_ID=$(aws ec2 describe-vpcs --filters Name=isDefault,Values=true \
  --query "Vpcs[0].VpcId" --output text)
for SG_NAME in ecs-alb-sg ecs-task-sg ecs-lab-sg; do
  SG_ID=$(aws ec2 describe-security-groups \
    --filters Name=group-name,Values=$SG_NAME Name=vpc-id,Values=$VPC_ID \
    --query "SecurityGroups[0].GroupId" --output text 2>/dev/null)
  [ -n "$SG_ID" ] && aws ec2 delete-security-group --group-id $SG_ID 2>/dev/null || true
done

echo "All ECS lab resources cleaned up."
```

---

## Congratulations!

You have completed the full ECS learning path:

| Lab | Topic |
|-----|-------|
| 01 | ECS concepts & architecture |
| 02 | First cluster, task definition, and run-task |
| 03 | Fargate service with ALB |
| 04 | Rolling updates, Blue/Green, auto scaling |
| 05 | IAM least-privilege, Secrets Manager, ECS Exec |
| 06 | CI/CD pipeline with ECR + CodeDeploy Blue/Green |
| 07 | Container Insights, FireLens, FARGATE_SPOT, cost optimisation |

Return to the [ECS Module Overview](../README.md) or explore other modules in the [aws-how-to-guides](../../README.md) repository.
