# Lab 04 — Networking & Ingress 🟡

## Goal
Install the AWS Load Balancer Controller, deploy multiple services, and route HTTP traffic using an Ingress resource backed by an Application Load Balancer (ALB). Understand VPC CNI prefix delegation and enforce basic network policies.

---

## Concepts

### AWS Load Balancer Controller vs In-Tree Provider

| Feature | In-Tree (legacy) | AWS Load Balancer Controller (recommended) |
|---------|-----------------|-------------------------------------------|
| ALB support | No | Yes |
| NLB IP mode | No | Yes |
| Ingress (L7 routing) | No | Yes |
| Target group health checks | Basic | Advanced |
| Maintenance status | Deprecated | Active |

### Ingress vs Service LoadBalancer

| | Service `type: LoadBalancer` | Ingress + ALB |
|-|------------------------------|---------------|
| Layer | L4 (TCP/UDP) | L7 (HTTP/HTTPS) |
| Cost | 1 LB per Service | 1 ALB for N services |
| Path routing | No | Yes |
| Host-based routing | No | Yes |
| TLS termination | Limited | Full ACM integration |

### VPC CNI Prefix Delegation
By default each secondary IP on an ENI occupies 1 pod slot. With prefix delegation (`/28` blocks), each ENI slot holds 16 IPs — greatly increasing pod density per node.

```bash
# Enable prefix delegation (recommended for large clusters)
kubectl set env daemonset aws-node -n kube-system \
  ENABLE_PREFIX_DELEGATION=true \
  WARM_PREFIX_TARGET=1
```

---

## Architecture

```
Internet
   │
   ▼
AWS Application Load Balancer (ALB)
   │
   ├── /api/*   ──► Service: api-svc      ──► Pod: api-app (port 8080)
   │                                           Pod: api-app
   │
   └── /*       ──► Service: frontend-svc ──► Pod: frontend (port 80)
                                               Pod: frontend
                                               Pod: frontend

Ingress resource (Kubernetes) controls ALB routing rules
AWS Load Balancer Controller watches Ingress resources and manages the ALB
```

---

## PoC Steps

### Step 1 — Install the AWS Load Balancer Controller via Helm

```bash
# Add the EKS charts repo
helm repo add eks https://aws.github.io/eks-charts
helm repo update

# Create IRSA service account for the controller
# (Requires the eksctl cluster from Lab 02 which enabled albIngress addon policy)
eksctl create iamserviceaccount \
  --cluster my-eks-cluster \
  --namespace kube-system \
  --name aws-load-balancer-controller \
  --attach-policy-arn arn:aws:iam::aws:policy/ElasticLoadBalancingFullAccess \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEC2FullAccess \
  --region us-east-1 \
  --override-existing-serviceaccounts \
  --approve

# Get your AWS account ID and VPC ID
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
VPC_ID=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query 'cluster.resourcesVpcConfig.vpcId' \
  --output text)

echo "Account: $ACCOUNT_ID  VPC: $VPC_ID"

# Install the controller
helm upgrade --install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=my-eks-cluster \
  --set serviceAccount.create=false \
  --set serviceAccount.name=aws-load-balancer-controller \
  --set region=us-east-1 \
  --set vpcId=$VPC_ID

# Verify
kubectl get deployment -n kube-system aws-load-balancer-controller
# NAME                           READY   UP-TO-DATE   AVAILABLE
# aws-load-balancer-controller   2/2     2            2
```

### Step 2 — Create a namespace and deploy two apps

```bash
kubectl create namespace lab04

# --- Frontend (nginx) ---
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: lab04
  labels:
    app: frontend
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "50m"
              memory: "64Mi"
            limits:
              cpu: "100m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: lab04
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
    - port: 80
      targetPort: 80
EOF

# --- API backend (httpbin — great for API testing) ---
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: api-app
  namespace: lab04
  labels:
    app: api-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: api-app
  template:
    metadata:
      labels:
        app: api-app
    spec:
      containers:
        - name: httpbin
          image: kennethreitz/httpbin:latest
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "200m"
              memory: "256Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: api-svc
  namespace: lab04
spec:
  type: ClusterIP
  selector:
    app: api-app
  ports:
    - port: 8080
      targetPort: 80
EOF

kubectl get pods -n lab04
# NAME                        READY   STATUS    RESTARTS   AGE
# api-app-xxxxxxxxx-aaaaa     1/1     Running   0          30s
# api-app-xxxxxxxxx-bbbbb     1/1     Running   0          30s
# frontend-xxxxxxxxx-ccccc    1/1     Running   0          30s
# frontend-xxxxxxxxx-ddddd    1/1     Running   0          30s
```

### Step 3 — Get the public subnet IDs for the ALB

```bash
# Tag public subnets so the ALB controller can find them
# eksctl already sets these tags; verify:
aws ec2 describe-subnets \
  --filters "Name=tag:kubernetes.io/role/elb,Values=1" \
            "Name=tag:alpha.eksctl.io/cluster-name,Values=my-eks-cluster" \
  --query 'Subnets[*].{ID:SubnetId, AZ:AvailabilityZone, CIDR:CidrBlock}' \
  --output table
# +------------------+-----------+-----------------+
# | ID               | AZ        | CIDR            |
# +------------------+-----------+-----------------+
# | subnet-0abc...   | us-east-1a| 10.0.0.0/19     |
# | subnet-0def...   | us-east-1b| 10.0.32.0/19    |
# +------------------+-----------+-----------------+
```

### Step 4 — Create the Ingress resource (ALB)

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: web-ingress
  namespace: lab04
  annotations:
    kubernetes.io/ingress.class: alb
    alb.ingress.kubernetes.io/scheme: internet-facing
    alb.ingress.kubernetes.io/target-type: ip
    alb.ingress.kubernetes.io/healthcheck-path: /healthz
    alb.ingress.kubernetes.io/tags: Project=eks-how-to-guide,Environment=lab
spec:
  rules:
    - http:
        paths:
          - path: /api
            pathType: Prefix
            backend:
              service:
                name: api-svc
                port:
                  number: 8080
          - path: /
            pathType: Prefix
            backend:
              service:
                name: frontend-svc
                port:
                  number: 80
EOF

# Watch the ALB being created (takes ~60-90 seconds)
kubectl get ingress web-ingress -n lab04 -w
# NAME          CLASS    HOSTS   ADDRESS                                            PORTS   AGE
# web-ingress   <none>   *       k8s-lab04-webingre-xxxx.us-east-1.elb.amazonaws.com   80      90s
```

### Step 5 — Test path-based routing

```bash
ALB=$(kubectl get ingress web-ingress -n lab04 \
  -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')

echo "ALB: $ALB"

# Frontend
curl -s -o /dev/null -w "%{http_code}" http://$ALB/
# 200

# API (httpbin /api/get returns JSON of request headers)
curl -s http://$ALB/api/get | python3 -m json.tool | head -20
# {
#   "args": {},
#   "headers": {
#     "Host": "k8s-lab04...",
#     "X-Forwarded-For": "...",
#     "X-Forwarded-Port": "80",
#     "X-Forwarded-Proto": "http"
#   },
#   "origin": "10.0.96.x",
#   "url": "http://k8s-lab04.../api/get"
# }
```

### Step 6 — Apply a basic NetworkPolicy

```bash
# Deny all ingress to lab04 by default, then allow only from within lab04
cat <<'EOF' | kubectl apply -f -
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: deny-all-ingress
  namespace: lab04
spec:
  podSelector: {}
  policyTypes:
    - Ingress
---
apiVersion: networking.k8s.io/v1
kind: NetworkPolicy
metadata:
  name: allow-intra-namespace
  namespace: lab04
spec:
  podSelector: {}
  policyTypes:
    - Ingress
  ingress:
    - from:
        - namespaceSelector:
            matchLabels:
              kubernetes.io/metadata.name: lab04
EOF

kubectl get networkpolicy -n lab04
# NAME                    POD-SELECTOR   AGE
# deny-all-ingress        <none>         5s
# allow-intra-namespace   <none>         5s
```

---

## Cleanup

```bash
kubectl delete namespace lab04
# ALB is automatically deleted when the Ingress resource is removed
```

---

## Next Lab
➡️ [Lab 05 — IAM & Security (IRSA, RBAC)](../05-iam-security/README.md)
