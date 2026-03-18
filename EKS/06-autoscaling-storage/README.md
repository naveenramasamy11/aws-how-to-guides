# Lab 06 — Autoscaling & Storage (HPA, Cluster Autoscaler, EBS CSI, EFS CSI) 🔴

## Goal
Configure Horizontal Pod Autoscaler (HPA) to scale pods based on CPU, deploy the Cluster Autoscaler to add/remove EC2 nodes on demand, and provision persistent storage using the EBS CSI driver (ReadWriteOnce) and EFS CSI driver (ReadWriteMany).

---

## Concepts

### Autoscaling in EKS — Three Layers

| Layer | Tool | Scales What | Trigger |
|-------|------|-------------|---------|
| Pod | HPA | Replicas in a Deployment | CPU/Memory/custom metrics |
| Pod | VPA (not covered) | CPU/memory requests per pod | Resource usage over time |
| Node | Cluster Autoscaler | EC2 nodes in managed node group | Pending pods (unschedulable) |
| Node | Karpenter (advanced) | EC2 nodes (any type) | Pending pods, consolidation |

### Storage: EBS vs EFS on EKS

| Feature | EBS (gp3) | EFS |
|---------|-----------|-----|
| Access Mode | ReadWriteOnce (1 node) | ReadWriteMany (many nodes) |
| Protocol | Block | NFS |
| Throughput | Up to 1000 MB/s | Bursting / provisioned |
| Cross-AZ | No | Yes |
| Use case | Databases, stateful single pods | Shared config, ML datasets, CMS |
| CSI Driver | `ebs.csi.aws.com` | `efs.csi.aws.com` |

### StorageClass Reclaim Policies

| Policy | Behaviour when PVC deleted |
|--------|---------------------------|
| `Retain` | PV kept, must be manually deleted |
| `Delete` | PV and underlying EBS/EFS resource automatically deleted |

---

## Architecture

```
CPU load generator pod ──► triggers HPA
     │
     ▼
HPA: web-app desired replicas 2 → 8 (CPU > 50%)
     │
     ▼
Cluster Autoscaler: pending pods → scale nodegroup 2 → 4 nodes
     │
     ▼
New EC2 nodes join cluster, pods scheduled

Stateful app pod
  ├── EBS PVC ──► EBS gp3 volume (single pod, AZ-pinned)
  └── EFS PVC ──► EFS NFS share (multiple pods, cross-AZ)
```

---

## PoC Steps

### Step 1 — Install metrics-server (required for HPA)

```bash
kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/latest/download/components.yaml

# Wait for metrics-server to be ready
kubectl wait deployment metrics-server \
  -n kube-system --for=condition=Available --timeout=120s

# Verify node/pod metrics are available
kubectl top nodes
# NAME                           CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
# ip-10-0-96-12.ec2.internal     85m          4%     512Mi           14%
# ip-10-0-128-45.ec2.internal    72m          3%     498Mi           13%

kubectl top pods -A
```

### Step 2 — Deploy a CPU-intensive app and configure HPA

```bash
kubectl create namespace lab06

# Deploy a PHP-Apache app (classic HPA demo)
cat <<'EOF' | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: php-apache
  namespace: lab06
spec:
  replicas: 1
  selector:
    matchLabels:
      app: php-apache
  template:
    metadata:
      labels:
        app: php-apache
    spec:
      containers:
        - name: php-apache
          image: registry.k8s.io/hpa-example
          ports:
            - containerPort: 80
          resources:
            requests:
              cpu: "200m"
              memory: "64Mi"
            limits:
              cpu: "500m"
              memory: "128Mi"
---
apiVersion: v1
kind: Service
metadata:
  name: php-apache
  namespace: lab06
spec:
  selector:
    app: php-apache
  ports:
    - port: 80
EOF

# Create HPA: scale 1-10 replicas, target 50% CPU
kubectl autoscale deployment php-apache \
  --namespace lab06 \
  --cpu-percent=50 \
  --min=1 \
  --max=10

kubectl get hpa -n lab06
# NAME         REFERENCE               TARGETS    MINPODS  MAXPODS  REPLICAS
# php-apache   Deployment/php-apache   0%/50%     1        10       1
```

### Step 3 — Generate load to trigger HPA

```bash
# Run load generator in background (sends requests for 2 minutes)
kubectl run load-generator \
  --namespace lab06 \
  --image=busybox:1.36 \
  --restart=Never \
  -- /bin/sh -c "while true; do wget -q -O- http://php-apache.lab06.svc.cluster.local; done" &

# Watch HPA react (check every 15 seconds)
kubectl get hpa php-apache -n lab06 -w
# NAME         REFERENCE               TARGETS     MINPODS  MAXPODS  REPLICAS
# php-apache   Deployment/php-apache   0%/50%      1        10       1
# php-apache   Deployment/php-apache   235%/50%    1        10       1
# php-apache   Deployment/php-apache   235%/50%    1        10       4
# php-apache   Deployment/php-apache   82%/50%     1        10       7

# Stop load generator
kubectl delete pod load-generator -n lab06 --ignore-not-found

# HPA scales back down after ~5 minutes (stabilization window)
kubectl get hpa php-apache -n lab06
```

### Step 4 — Install the Cluster Autoscaler

```bash
# The eksctl cluster config in Lab 02 already set the required IAM and tags
# Deploy the Cluster Autoscaler
curl -O https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# Update the cluster name in the manifest
sed -i 's/<YOUR CLUSTER NAME>/my-eks-cluster/g' cluster-autoscaler-autodiscover.yaml

kubectl apply -f cluster-autoscaler-autodiscover.yaml

# Annotate to prevent CA from evicting itself
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"

# Verify
kubectl get deployment cluster-autoscaler -n kube-system
# NAME                 READY   UP-TO-DATE   AVAILABLE
# cluster-autoscaler   1/1     1            1
```

### Step 5 — Trigger Cluster Autoscaler by requesting more nodes

```bash
# Deploy 20 replicas — each needs 200m CPU. With 2 t3.medium nodes (2 vCPU each),
# only ~8-10 pods fit. The rest will be Pending, triggering CA.
kubectl scale deployment php-apache --replicas=20 -n lab06

# Watch for Pending pods
kubectl get pods -n lab06 | grep Pending

# Watch CA logs
kubectl logs -n kube-system \
  -l app=cluster-autoscaler \
  --tail=20 -f | grep -E "scale|node"
# I0115 10:35:00.000000  scale up triggered for node group
# I0115 10:37:00.000000  node ip-10-0-96-200.ec2.internal joined the cluster

# Watch nodes scale up
kubectl get nodes -w
# New nodes appear with STATUS: Ready after ~2-3 minutes

# Scale back down
kubectl scale deployment php-apache --replicas=1 -n lab06
```

### Step 6 — Install EBS CSI Driver and create a PVC

```bash
# Add EBS CSI driver as an EKS managed add-on
aws eks create-addon \
  --cluster-name my-eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --region us-east-1 \
  --tags Project=eks-how-to-guide

aws eks describe-addon \
  --cluster-name my-eks-cluster \
  --addon-name aws-ebs-csi-driver \
  --region us-east-1 \
  --query 'addon.{Name:addonName, Status:status, Version:addonVersion}' \
  --output table

# Create a StorageClass for gp3
cat <<'EOF' | kubectl apply -f -
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: ebs-gp3
  annotations:
    storageclass.kubernetes.io/is-default-class: "false"
provisioner: ebs.csi.aws.com
parameters:
  type: gp3
  encrypted: "true"
volumeBindingMode: WaitForFirstConsumer   # provision in same AZ as pod
reclaimPolicy: Delete
allowVolumeExpansion: true
EOF

# Create a PVC (ReadWriteOnce)
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: app-data-pvc
  namespace: lab06
spec:
  accessModes:
    - ReadWriteOnce
  storageClassName: ebs-gp3
  resources:
    requests:
      storage: 5Gi
EOF

# Create a pod that uses the PVC
cat <<'EOF' | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: ebs-writer
  namespace: lab06
spec:
  containers:
    - name: app
      image: busybox:1.36
      command: ["/bin/sh", "-c",
        "echo 'EKS Lab06 EBS data written at '$(date) > /data/output.txt && cat /data/output.txt && sleep 3600"]
      volumeMounts:
        - name: data-vol
          mountPath: /data
      resources:
        requests:
          cpu: "50m"
          memory: "32Mi"
  volumes:
    - name: data-vol
      persistentVolumeClaim:
        claimName: app-data-pvc
EOF

kubectl wait --for=condition=Ready pod/ebs-writer -n lab06 --timeout=120s
kubectl logs ebs-writer -n lab06
# EKS Lab06 EBS data written at Mon Jan 15 10:40:00 UTC 2024

# Check PVC is bound
kubectl get pvc app-data-pvc -n lab06
# NAME           STATUS   VOLUME             CAPACITY   ACCESS MODES   STORAGECLASS   AGE
# app-data-pvc   Bound    pvc-xxxx-yyyy      5Gi        RWO            ebs-gp3        2m
```

---

## Cleanup

```bash
kubectl delete namespace lab06
kubectl delete storageclass ebs-gp3
aws eks delete-addon --cluster-name my-eks-cluster --addon-name aws-ebs-csi-driver --region us-east-1
kubectl delete -f cluster-autoscaler-autodiscover.yaml 2>/dev/null || true
```

---

## Next Lab
➡️ [Lab 07 — Production Patterns](../07-production-patterns/README.md)
