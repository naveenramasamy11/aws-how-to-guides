# Lab 03 — Deploying Workloads 🟡

## Goal
Deploy a real containerised application (NGINX + a simple Python API) to EKS using Deployments, Services, ConfigMaps, and Secrets. Practice rolling updates, rollbacks, health probes, and resource requests/limits.

---

## Concepts

### Core Kubernetes Objects

| Object | Purpose | Key Fields |
|--------|---------|-----------|
| **Deployment** | Declarative replica-set of pods | `replicas`, `selector`, `template`, `strategy` |
| **Service** | Stable DNS + VIP in front of pods | `type`, `selector`, `ports` |
| **ConfigMap** | Inject non-secret config as env vars or files | `data`, `binaryData` |
| **Secret** | Inject sensitive data (base64-encoded) | `type`, `data`, `stringData` |
| **Namespace** | Logical cluster isolation | `name` |

### Service Types

| Type | Reachable From | Use Case |
|------|---------------|----------|
| `ClusterIP` (default) | Inside cluster only | Inter-service communication |
| `NodePort` | Node IP + high port (30000–32767) | Simple external access (dev only) |
| `LoadBalancer` | Internet (creates AWS CLB/NLB) | Prod external access |
| `ExternalName` | Inside cluster → external DNS | Alias external services |

### Rolling Update Strategy

```
maxSurge: 1       → at most 1 extra pod above desired during update
maxUnavailable: 0 → never drop below desired count
```

---

## Architecture

```
Internet / kubectl
     │
     ▼
Service (type: LoadBalancer)  ──► AWS Classic/Network Load Balancer
     │
     ├── Pod (nginx:1.27)   [Deployment: web-app, replica 1]
     ├── Pod (nginx:1.27)   [Deployment: web-app, replica 2]
     └── Pod (nginx:1.27)   [Deployment: web-app, replica 3]
          │
     ConfigMap ──► /etc/nginx/conf.d/default.conf
     Secret    ──► env var APP_SECRET_KEY
```

---

## PoC Steps

### Step 1 — Create a dedicated namespace

```bash
kubectl create namespace lab03
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   1h
# kube-system       Active   1h
# lab03             Active   2s
```

### Step 2 — Create a ConfigMap

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: ConfigMap
metadata:
  name: web-config
  namespace: lab03
  labels:
    app: web-app
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  nginx.conf: |
    server {
        listen 80;
        server_name _;
        location / {
            root /usr/share/nginx/html;
            index index.html;
            add_header X-Powered-By "EKS-Lab03";
        }
        location /healthz {
            return 200 "healthy\n";
            add_header Content-Type text/plain;
        }
    }
EOF

kubectl get configmap web-config -n lab03 -o yaml
```

### Step 3 — Create a Secret

```bash
# Secrets are base64-encoded (NOT encrypted by default — use KMS/ASCP for prod)
kubectl create secret generic web-secret \
  --namespace lab03 \
  --from-literal=APP_SECRET_KEY=my-super-secret-12345 \
  --from-literal=DB_PASSWORD=lab03-db-pass \
  --dry-run=client -o yaml | kubectl apply -f -

# Verify (values are base64)
kubectl get secret web-secret -n lab03 -o jsonpath='{.data.APP_SECRET_KEY}' | base64 --decode
# my-super-secret-12345
```

### Step 4 — Create the Deployment

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: web-app
  namespace: lab03
  labels:
    app: web-app
    version: v1
spec:
  replicas: 3
  selector:
    matchLabels:
      app: web-app
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: web-app
        version: v1
    spec:
      containers:
        - name: nginx
          image: nginx:1.27
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "100m"
              memory: "128Mi"
            limits:
              cpu: "250m"
              memory: "256Mi"
          env:
            - name: APP_ENV
              valueFrom:
                configMapKeyRef:
                  name: web-config
                  key: APP_ENV
            - name: APP_SECRET_KEY
              valueFrom:
                secretKeyRef:
                  name: web-secret
                  key: APP_SECRET_KEY
          volumeMounts:
            - name: nginx-config
              mountPath: /etc/nginx/conf.d
          livenessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            httpGet:
              path: /healthz
              port: 80
            initialDelaySeconds: 3
            periodSeconds: 5
      volumes:
        - name: nginx-config
          configMap:
            name: web-config
            items:
              - key: nginx.conf
                path: default.conf
EOF

# Watch rollout
kubectl rollout status deployment/web-app -n lab03
# Waiting for deployment "web-app" rollout to finish: 0 of 3 updated replicas are available...
# deployment "web-app" successfully rolled out
```

### Step 5 — Expose the Deployment with a Service

```bash
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Service
metadata:
  name: web-app-svc
  namespace: lab03
  labels:
    app: web-app
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: "nlb"
spec:
  type: LoadBalancer
  selector:
    app: web-app
  ports:
    - name: http
      protocol: TCP
      port: 80
      targetPort: 80
EOF

# Wait for the ELB to be provisioned (~60-90 seconds)
kubectl get svc web-app-svc -n lab03 -w
# NAME          TYPE           CLUSTER-IP      EXTERNAL-IP                                        PORT(S)
# web-app-svc   LoadBalancer   172.20.47.143   abc123.elb.us-east-1.amazonaws.com                80:31234/TCP

# Test (once EXTERNAL-IP is populated)
ELB=$(kubectl get svc web-app-svc -n lab03 -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl -s http://$ELB/healthz
# healthy
curl -I http://$ELB/
# HTTP/1.1 200 OK
# X-Powered-By: EKS-Lab03
```

### Step 6 — Perform a rolling update

```bash
# Update to nginx 1.27.3
kubectl set image deployment/web-app nginx=nginx:1.27.3 -n lab03

# Watch rolling update in real-time
kubectl rollout status deployment/web-app -n lab03 -w
# Waiting for deployment "web-app" rollout to finish: 1 out of 3 new replicas have been updated...
# Waiting for deployment "web-app" rollout to finish: 2 out of 3 new replicas have been updated...
# deployment "web-app" successfully rolled out

# Confirm new image
kubectl describe deployment web-app -n lab03 | grep Image:
# Image: nginx:1.27.3

# View rollout history
kubectl rollout history deployment/web-app -n lab03
# REVISION  CHANGE-CAUSE
# 1         <none>
# 2         <none>
```

### Step 7 — Rollback

```bash
# Roll back to revision 1 (nginx:1.27)
kubectl rollout undo deployment/web-app -n lab03 --to-revision=1

kubectl rollout status deployment/web-app -n lab03
# deployment "web-app" successfully rolled out

kubectl describe deployment web-app -n lab03 | grep Image:
# Image: nginx:1.27
```

### Step 8 — Scale manually

```bash
# Scale up to 5 replicas
kubectl scale deployment web-app --replicas=5 -n lab03

# Verify
kubectl get pods -n lab03 -l app=web-app
# NAME                       READY   STATUS    RESTARTS   AGE
# web-app-xxxxxxxxx-aaaaa    1/1     Running   0          10m
# web-app-xxxxxxxxx-bbbbb    1/1     Running   0          10m
# web-app-xxxxxxxxx-ccccc    1/1     Running   0          10m
# web-app-xxxxxxxxx-ddddd    1/1     Running   0          30s
# web-app-xxxxxxxxx-eeeee    1/1     Running   0          30s
```

---

## Cleanup

```bash
kubectl delete namespace lab03
# namespace "lab03" deleted — this deletes all resources in the namespace
```

---

## Next Lab
➡️ [Lab 04 — Networking & Ingress](../04-networking-ingress/README.md)
