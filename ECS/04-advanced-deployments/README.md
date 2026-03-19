# Lab 04 — Rolling Updates, Blue/Green & Auto Scaling 🟡

## Goal

Master ECS deployment strategies — rolling updates with controlled minimum healthy percent, Blue/Green deployments via AWS CodeDeploy, and Application Auto Scaling based on CPU utilisation.

---

## Concepts

### Deployment Strategies Compared

| Strategy | Downtime Risk | Rollback Speed | Extra Cost | Best For |
|----------|---------------|----------------|------------|---------|
| **Rolling** | Low | Manual re-deploy | None | General updates |
| **Blue/Green** (CodeDeploy) | Zero | Seconds (traffic flip) | ~2x tasks during cutover | Production critical services |
| **Canary** (CodeDeploy) | Zero | Seconds | ~2x tasks | Gradual validation |
| **Linear** (CodeDeploy) | Zero | Seconds | ~2x tasks | Compliance-gated rollouts |

### Rolling Update Parameters

| Parameter | Description |
|-----------|-------------|
| `minimumHealthyPercent` | Floor on running tasks during update (e.g. 50% = half can be stopped) |
| `maximumPercent` | Ceiling — controls over-provisioning during update (200% = double count) |

With `minimumHealthyPercent=100` and `maximumPercent=200` and `desiredCount=2`: ECS starts 2 new tasks, waits for them to pass health checks, then stops the 2 old tasks — **zero downtime**.

### Application Auto Scaling

ECS integrates with **Application Auto Scaling** (not EC2 Auto Scaling). You define:
1. A **scalable target** (the ECS service)
2. A **scaling policy** (Target Tracking on CPU or custom CloudWatch metric)

---

## Architecture — Blue/Green

```
  Traffic (100%)
       │
       ▼
  ALB Listener :80
       │
  ┌────┴──────────────────────────────┐
  │  Listener Rule                     │
  │  Default → Production TG (Blue)    │
  │  Test     → Test TG (Green)        │
  └────────────────────────────────────┘
       │                    │
  ┌────▼────┐          ┌────▼────┐
  │Blue TG  │          │Green TG │
  │(current)│  ──────► │(new)    │
  └─────────┘  CodeDeploy shifts  └─────────┘
               traffic 0% → 100%
```

---

## Step-by-Step PoC

### Part A — Rolling Update

#### Step 1 — Update the task definition with a new image tag

```bash
export AWS_DEFAULT_REGION=us-east-1
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)

# Register revision 2 of nginx-lab — simulate an app update by changing the image
cat > /tmp/nginx-task-def-v2.json <<'EOF'
{
  "family": "nginx-lab",
  "networkMode": "awsvpc",
  "requiresCompatibilities": ["FARGATE"],
  "cpu": "256",
  "memory": "512",
  "executionRoleArn": "EXEC_ROLE_PLACEHOLDER",
  "containerDefinitions": [
    {
      "name": "nginx",
      "image": "nginx:1.25-alpine",
      "portMappings": [{"containerPort": 80, "protocol": "tcp"}],
      "essential": true,
      "environment": [
        {"name": "APP_VERSION", "value": "v2"}
      ],
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

EXEC_ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/ecsTaskExecutionRole"
sed -i "s|EXEC_ROLE_PLACEHOLDER|${EXEC_ROLE_ARN}|" /tmp/nginx-task-def-v2.json

aws ecs register-task-definition \
  --cli-input-json file:///tmp/nginx-task-def-v2.json \
  --tags key=Project,value=ecs-lab \
  --query "taskDefinition.[family,revision,status]" \
  --output table
```

**Expected output:**
```
+----------+----+--------+
| nginx-lab | 2  | ACTIVE |
+----------+----+--------+
```

---

#### Step 2 — Trigger a rolling update

```bash
aws ecs update-service \
  --cluster lab-cluster \
  --service nginx-service \
  --task-definition nginx-lab:2 \
  --deployment-configuration "minimumHealthyPercent=100,maximumPercent=200" \
  --force-new-deployment \
  --output table

# Watch the deployment events
watch -n 5 'aws ecs describe-services \
  --cluster lab-cluster \
  --services nginx-service \
  --query "services[0].deployments[*].[status,desiredCount,runningCount,pendingCount,taskDefinition]" \
  --output table'
```

**During the update you'll see two deployments: PRIMARY (new) and ACTIVE (old):**
```
+---------+-------+--------+---------+--------------------+
| PRIMARY |  2    |   2    |   2     | nginx-lab:2        |
| ACTIVE  |  2    |   2    |   0     | nginx-lab:1        |
+---------+-------+--------+---------+--------------------+
```

Wait for `ACTIVE` to disappear — update complete.

---

### Part B — Application Auto Scaling

#### Step 3 — Register the ECS service as a scalable target

```bash
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/lab-cluster/nginx-service \
  --min-capacity 1 \
  --max-capacity 10

echo "Scalable target registered."
```

---

#### Step 4 — Create a Target Tracking policy on CPU utilisation

```bash
cat > /tmp/scaling-policy.json <<'EOF'
{
  "TargetValue": 50.0,
  "PredefinedMetricSpecification": {
    "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
  },
  "ScaleInCooldown":  60,
  "ScaleOutCooldown": 60
}
EOF

aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/lab-cluster/nginx-service \
  --policy-name cpu-target-tracking \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration file:///tmp/scaling-policy.json \
  --output table
```

---

#### Step 5 — Generate load and watch scaling

```bash
# In a second terminal, hit the ALB repeatedly
ALB_DNS=$(aws elbv2 describe-load-balancers \
  --names ecs-lab-alb \
  --query "LoadBalancers[0].DNSName" --output text)

# Run 500 concurrent requests (requires `hey` — install with: go install github.com/rakyll/hey@latest)
hey -n 50000 -c 200 http://$ALB_DNS/

# In another terminal, watch task count
watch -n 10 'aws ecs describe-services \
  --cluster lab-cluster --services nginx-service \
  --query "services[0].[desiredCount,runningCount]" --output table'
```

---

#### Step 6 — View scaling activity

```bash
aws application-autoscaling describe-scaling-activities \
  --service-namespace ecs \
  --resource-id service/lab-cluster/nginx-service \
  --query "ScalingActivities[*].[ActivityId,Description,StatusCode,StartTime]" \
  --output table
```

---

## Cleanup

```bash
# Remove scaling policy
aws application-autoscaling delete-scaling-policy \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/lab-cluster/nginx-service \
  --policy-name cpu-target-tracking

# Deregister scalable target
aws application-autoscaling deregister-scalable-target \
  --service-namespace ecs \
  --scalable-dimension ecs:service:DesiredCount \
  --resource-id service/lab-cluster/nginx-service

echo "Auto scaling cleanup complete."
```

> The ECS service and ALB from Lab 03 can remain for use in Lab 05.

---

➡️ [Lab 05 → IAM Roles, Secrets & VPC Isolation](../05-iam-and-security/README.md)
