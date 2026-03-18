# Lab 07 — Production Patterns (Monitoring, Logging, Upgrades) 🔴

## Goal
Enable CloudWatch Container Insights for cluster observability, configure Fluent Bit for centralised pod log shipping to CloudWatch Logs, implement pod disruption budgets, set resource quotas per namespace, and perform a safe in-place Kubernetes version upgrade.

---

## Concepts

### Observability Stack on EKS

| Layer | Tool | What It Captures |
|-------|------|-----------------|
| Metrics | CloudWatch Container Insights | Node CPU/mem, pod restarts, disk I/O |
| Metrics | Prometheus + Grafana (alt) | Custom app metrics, K8s API metrics |
| Logs | Fluent Bit DaemonSet → CloudWatch Logs | All pod stdout/stderr |
| Logs | CloudWatch Logs Insights | SQL-like queries over log streams |
| Traces | AWS X-Ray (optional) | Distributed tracing across services |
| Audit | EKS Control Plane Logging | K8s API audit trail in CloudWatch |

### Cost Optimisation Checklist

| Practice | Impact |
|----------|--------|
| Set CPU/memory requests accurately | Prevents over-provisioning |
| Use Spot instances for non-critical node groups | Up to 70% cheaper |
| Enable Cluster Autoscaler or Karpenter | Scale to zero when idle |
| Use `WaitForFirstConsumer` StorageClass | Avoid cross-AZ EBS charges |
| Use Savings Plans / Reserved Instances for baseline | 40-60% discount |
| Delete unused load balancers (Ingress resources) | ELB costs $0.02/hr idle |

### Kubernetes Version Upgrade Strategy

```
EKS supports N-2 versions. Upgrade path:
  1.29 → 1.30 → 1.31 (cannot skip minor versions)

Safe upgrade order:
  1. Upgrade control plane (EKS) — zero downtime, managed by AWS
  2. Upgrade add-ons (vpc-cni, coredns, kube-proxy)
  3. Upgrade managed node groups (rolling replacement)
```

---

## Architecture

```
All Worker Nodes
  └── Fluent Bit DaemonSet
        ├── /var/log/containers/*.log  (pod stdout/stderr)
        └── CloudWatch Logs → /aws/containerinsights/my-eks-cluster/application

CloudWatch Container Insights
  ├── ContainerInsights namespace
  │   └── CloudWatch Agent DaemonSet (node-level metrics)
  └── AWS Console: CloudWatch → Container Insights → my-eks-cluster
```

---

## PoC Steps

### Step 1 — Enable CloudWatch Container Insights

```bash
# The ClusterConfig in Lab 02 enabled CloudWatch addon policy on the node group
# Install Container Insights using the quick-start script
ClusterName="my-eks-cluster"
RegionName="us-east-1"
FluentBitHttpPort='2020'
FluentBitReadFromHead='Off'
[[ ${FluentBitReadFromHead} = 'On' ]] && FluentBitReadFromTail='Off' || FluentBitReadFromTail='On'
[[ -z ${FluentBitHttpPort} ]] && FluentBitHttpServer='Off' || FluentBitHttpServer='On'

curl -s https://raw.githubusercontent.com/aws-samples/amazon-cloudwatch-container-insights/latest/k8s-deployment-manifest-templates/deployment-mode/daemonset/container-insights-monitoring/quickstart/cwagent-fluent-bit-quickstart.yaml \
  | sed "s/{{cluster_name}}/${ClusterName}/;s/{{region_name}}/${RegionName}/;s/{{http_server_toggle}}/${FluentBitHttpServer}/;s/{{http_server_port}}/${FluentBitHttpPort}/;s/{{read_from_head}}/${FluentBitReadFromHead}/;s/{{read_from_tail}}/${FluentBitReadFromTail}/" \
  | kubectl apply -f -

# Verify agents are running (one per node)
kubectl get pods -n amazon-cloudwatch
# NAME                          READY   STATUS    RESTARTS   AGE
# cloudwatch-agent-xxxxx        1/1     Running   0          2m   <- metrics
# fluent-bit-yyyyy              1/1     Running   0          2m   <- logs
# fluent-bit-zzzzz              1/1     Running   0          2m   <- logs
```

### Step 2 — Generate sample application logs to ship to CloudWatch

```bash
kubectl create namespace lab07

cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: log-generator
  namespace: lab07
spec:
  replicas: 2
  selector:
    matchLabels:
      app: log-generator
  template:
    metadata:
      labels:
        app: log-generator
    spec:
      containers:
        - name: generator
          image: busybox:1.36
          command:
            - /bin/sh
            - -c
            - |
              i=0
              while true; do
                echo "{\"timestamp\":\"$(date -u +%Y-%m-%dT%H:%M:%SZ)\",\"level\":\"INFO\",\"msg\":\"Request processed\",\"request_id\":\"req-$i\",\"duration_ms\":$((RANDOM % 200 + 10))}"
                i=$((i+1))
                sleep 2
              done
          resources:
            requests:
              cpu: "10m"
              memory: "16Mi"
            limits:
              cpu: "20m"
              memory: "32Mi"
EOF

kubectl rollout status deployment/log-generator -n lab07

# View logs locally
kubectl logs -l app=log-generator -n lab07 --tail=5
# {"timestamp":"2024-01-15T10:45:00Z","level":"INFO","msg":"Request processed","request_id":"req-12","duration_ms":145}
```

### Step 3 — Query logs in CloudWatch Logs Insights

```bash
# Get the log group name (Fluent Bit ships to /aws/containerinsights/<cluster>/application)
LOG_GROUP="/aws/containerinsights/my-eks-cluster/application"

# Wait 2-3 minutes for logs to arrive, then run a query
QUERY_ID=$(aws logs start-query \
  --log-group-name "$LOG_GROUP" \
  --start-time $(date -d '10 minutes ago' +%s) \
  --end-time $(date +%s) \
  --query-string 'fields @timestamp, log | filter kubernetes.namespace_name = "lab07" | sort @timestamp desc | limit 5' \
  --query 'queryId' \
  --output text \
  --region us-east-1)

echo "Query ID: $QUERY_ID"
sleep 5

aws logs get-query-results \
  --query-id "$QUERY_ID" \
  --region us-east-1 \
  --query 'results[*][?field==`log`].value' \
  --output text
# {"timestamp":"2024-01-15T10:47:30Z","level":"INFO","msg":"Request processed","request_id":"req-42","duration_ms":87}
```

### Step 4 — Configure Pod Disruption Budget

```bash
# PDB prevents too many pods being down during voluntary disruptions (node drains, upgrades)
cat <<'EOF' | kubectl apply -f -
apiVersion: policy/v1
kind: PodDisruptionBudget
metadata:
  name: log-generator-pdb
  namespace: lab07
spec:
  minAvailable: 1      # at least 1 pod must be running during disruptions
  selector:
    matchLabels:
      app: log-generator
EOF

kubectl get pdb -n lab07
# NAME                 MIN AVAILABLE   MAX UNAVAILABLE   ALLOWED DISRUPTIONS   AGE
# log-generator-pdb   1               N/A               1                     5s
```

### Step 5 — Apply Resource Quotas to the namespace

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ResourceQuota
metadata:
  name: lab07-quota
  namespace: lab07
spec:
  hard:
    requests.cpu: "500m"
    requests.memory: "512Mi"
    limits.cpu: "1"
    limits.memory: "1Gi"
    pods: "10"
    services: "5"
    persistentvolumeclaims: "3"
---
apiVersion: v1
kind: LimitRange
metadata:
  name: lab07-limitrange
  namespace: lab07
spec:
  limits:
    - type: Container
      default:
        cpu: "100m"
        memory: "128Mi"
      defaultRequest:
        cpu: "50m"
        memory: "64Mi"
      max:
        cpu: "500m"
        memory: "512Mi"
EOF

kubectl describe resourcequota lab07-quota -n lab07
# Name:                   lab07-quota
# Namespace:              lab07
# Resource                Used   Hard
# --------                ----   ----
# limits.cpu              40m    1
# limits.memory           64Mi   1Gi
# pods                    2      10
# requests.cpu            20m    500m
# requests.memory         32Mi   512Mi
```

### Step 6 — Upgrade the EKS cluster (control plane)

```bash
# Check current version
aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query 'cluster.{Version:version, Status:status, PlatformVersion:platformVersion}' \
  --output table

# Check available upgrade versions
aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query 'cluster.{CurrentVersion:version}' \
  --output text

# STEP 1: Upgrade control plane to next minor version (e.g., 1.30 → 1.31)
# This is zero-downtime; AWS rolls the control plane across AZs
aws eks update-cluster-version \
  --name my-eks-cluster \
  --kubernetes-version 1.31 \
  --region us-east-1

# Monitor upgrade progress (~10 minutes)
watch -n 30 "aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query 'cluster.{Status:status, Version:version}' \
  --output table"

# STEP 2: Update EKS add-ons to versions compatible with new K8s version
for ADDON in coredns kube-proxy vpc-cni; do
  LATEST=$(aws eks describe-addon-versions \
    --addon-name $ADDON \
    --kubernetes-version 1.31 \
    --region us-east-1 \
    --query 'addons[0].addonVersions[0].addonVersion' \
    --output text)
  echo "Updating $ADDON to $LATEST"
  aws eks update-addon \
    --cluster-name my-eks-cluster \
    --addon-name $ADDON \
    --addon-version $LATEST \
    --resolve-conflicts OVERWRITE \
    --region us-east-1
done

# STEP 3: Upgrade managed node group (rolling node replacement)
eksctl upgrade nodegroup \
  --cluster my-eks-cluster \
  --name ng-workers \
  --kubernetes-version 1.31 \
  --region us-east-1

# Verify all nodes on new version
kubectl get nodes -o wide
# NAME                           STATUS   VERSION    OS-IMAGE
# ip-10-0-96-12.ec2.internal     Ready    v1.31.x    Amazon Linux 2
# ip-10-0-128-45.ec2.internal    Ready    v1.31.x    Amazon Linux 2
```

### Step 7 — Set up a CloudWatch Alarm for pod restart rate

```bash
# Create an alarm that fires when any pod restarts more than 5 times in 5 minutes
aws cloudwatch put-metric-alarm \
  --alarm-name "EKS-PodRestarts-High" \
  --alarm-description "Pod restart count exceeding threshold in my-eks-cluster" \
  --namespace ContainerInsights \
  --metric-name pod_number_of_container_restarts \
  --dimensions Name=ClusterName,Value=my-eks-cluster \
  --statistic Sum \
  --period 300 \
  --evaluation-periods 1 \
  --threshold 5 \
  --comparison-operator GreaterThanOrEqualToThreshold \
  --alarm-actions "arn:aws:sns:us-east-1:$(aws sts get-caller-identity --query Account --output text):eks-alerts" \
  --treat-missing-data notBreaching \
  --region us-east-1 \
  --tags Key=Project,Value=eks-how-to-guide

aws cloudwatch describe-alarms \
  --alarm-names "EKS-PodRestarts-High" \
  --query 'MetricAlarms[*].{Name:AlarmName, State:StateValue, Threshold:Threshold}' \
  --output table
# ┌────────────────────────────┬───────────────┬────────────┐
# │  Name                      │  State        │  Threshold │
# ├────────────────────────────┼───────────────┼────────────┤
# │  EKS-PodRestarts-High      │  INSUFFICIENT │  5.0       │
# └────────────────────────────┴───────────────┴────────────┘
```

---

## Cleanup

```bash
# Delete namespace resources
kubectl delete namespace lab07 amazon-cloudwatch

# Delete CloudWatch alarm
aws cloudwatch delete-alarms --alarm-names "EKS-PodRestarts-High" --region us-east-1

# Full cluster teardown (if done with all labs)
eksctl delete cluster --name my-eks-cluster --region us-east-1
# This deletes: cluster, node groups, VPC, CloudFormation stacks (takes ~10 min)
```

---

## Production Readiness Checklist

| Item | Done? |
|------|-------|
| IRSA configured for all service accounts needing AWS access | [ ] |
| RBAC roles limit namespace access per team | [ ] |
| Resource requests/limits set on all containers | [ ] |
| Resource quotas applied per namespace | [ ] |
| HPA configured for traffic-facing deployments | [ ] |
| Cluster Autoscaler or Karpenter enabled | [ ] |
| PodDisruptionBudgets set for critical workloads | [ ] |
| CloudWatch Container Insights enabled | [ ] |
| Fluent Bit shipping logs to CloudWatch Logs | [ ] |
| Control plane logging enabled (audit, api, authenticator) | [ ] |
| CloudWatch alarms on pod restarts and node CPU | [ ] |
| EKS version upgrade runbook documented | [ ] |

---

## You've Completed the EKS Module!

Return to the [EKS module overview](../README.md) or go back to the [repository root](../../README.md).
