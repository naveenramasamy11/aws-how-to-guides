# Lab 05 — IAM & Security (IRSA, RBAC, Secrets Manager CSI) 🔴

## Goal
Implement least-privilege AWS access for pods using IAM Roles for Service Accounts (IRSA), enforce Kubernetes RBAC for team-level access control, mount AWS Secrets Manager secrets directly into pods using the ASCP CSI driver, and audit with CloudTrail.

---

## Concepts

### IRSA — IAM Roles for Service Accounts

Without IRSA, all pods on a node share the node's IAM role — a significant blast radius. IRSA maps a Kubernetes Service Account to a specific IAM Role using OIDC federation.

```
Pod → uses K8s Service Account → K8s SA annotated with IAM Role ARN
         │
         ▼ (OIDC token projection)
AWS STS AssumeRoleWithWebIdentity → short-lived credentials (1 hr TTL)
         │
         ▼
Pod gets scoped AWS permissions for that role only
```

### RBAC Key Objects

| Object | Scope | Purpose |
|--------|-------|---------|
| `ClusterRole` | Cluster-wide | Define permissions (verbs on resources) |
| `ClusterRoleBinding` | Cluster-wide | Bind ClusterRole to users/groups/SAs |
| `Role` | Namespace | Define permissions scoped to namespace |
| `RoleBinding` | Namespace | Bind Role to users/groups/SAs |

### AWS Secrets Manager CSI Driver (ASCP)
Mounts secrets from Secrets Manager or Parameter Store as files or env vars in pods. Secrets are fetched at pod start and optionally rotated (re-read every `rotationPollInterval`). Combined with IRSA, each pod only has access to its own secrets.

---

## Architecture

```
Pod (service-account: s3-reader-sa)
  │
  ├── OIDC token (auto-projected by EKS)
  │
  ▼
AWS STS: AssumeRoleWithWebIdentity
  │
  ▼
IAM Role: eks-s3-reader-role  (only allows s3:GetObject on specific bucket)
  │
  ▼
Amazon S3 Bucket: my-lab05-data-bucket
```

```
Pod (has SecretProviderClass volume)
  │
  ▼
ASCP CSI Driver (DaemonSet on each node)
  │
  ▼
AWS Secrets Manager: my-app/lab05/db-credentials
  │ (value mounted as /mnt/secrets/db-credentials)
  ▼
Pod reads /mnt/secrets/db-credentials at runtime
```

---

## PoC Steps

### Step 1 — Verify OIDC provider is configured

```bash
# eksctl creates the OIDC provider automatically
OIDC_URL=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text)

echo "OIDC URL: $OIDC_URL"
# https://oidc.eks.us-east-1.amazonaws.com/id/EXAMPLED539D4633E53DE1B71EXAMPLE

# Confirm provider is registered in IAM
aws iam list-open-id-connect-providers \
  --query 'OpenIDConnectProviderList[*].Arn' \
  --output table
# ┌───────────────────────────────────────────────────────────────────────────────────┐
# │  arn:aws:iam::123456789012:oidc-provider/oidc.eks.us-east-1.amazonaws.com/id/XXX │
# └───────────────────────────────────────────────────────────────────────────────────┘
```

### Step 2 — Create an S3 bucket and upload sample data

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="eks-lab05-data-${ACCOUNT_ID}"

aws s3 mb s3://$BUCKET_NAME --region us-east-1

# Upload sample data
echo "Secret config data: db_host=prod-rds.cluster.us-east-1.rds.amazonaws.com" \
  > /tmp/app-config.txt

aws s3 cp /tmp/app-config.txt s3://$BUCKET_NAME/config/app-config.txt \
  --tags "Project=eks-how-to-guide"

aws s3 ls s3://$BUCKET_NAME/config/
# 2024-01-15 10:30:00       68 app-config.txt
```

### Step 3 — Create the IAM Role for the pod (IRSA)

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
OIDC_ID=$(aws eks describe-cluster \
  --name my-eks-cluster \
  --region us-east-1 \
  --query "cluster.identity.oidc.issuer" \
  --output text | sed 's|https://||')

# Create the trust policy
cat > /tmp/irsa-trust-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::${ACCOUNT_ID}:oidc-provider/${OIDC_ID}"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "${OIDC_ID}:sub": "system:serviceaccount:lab05:s3-reader-sa",
          "${OIDC_ID}:aud": "sts.amazonaws.com"
        }
      }
    }
  ]
}
EOF

# Create the role
aws iam create-role \
  --role-name eks-s3-reader-role \
  --assume-role-policy-document file:///tmp/irsa-trust-policy.json \
  --tags Key=Project,Value=eks-how-to-guide \
  --query 'Role.{RoleName:RoleName, RoleArn:Arn}' \
  --output table

ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/eks-s3-reader-role"

# Create and attach a least-privilege policy
cat > /tmp/s3-read-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:ListBucket"
      ],
      "Resource": [
        "arn:aws:s3:::${BUCKET_NAME}",
        "arn:aws:s3:::${BUCKET_NAME}/*"
      ]
    }
  ]
}
EOF

aws iam create-policy \
  --policy-name eks-lab05-s3-read \
  --policy-document file:///tmp/s3-read-policy.json \
  --tags Key=Project,Value=eks-how-to-guide \
  --query 'Policy.{PolicyName:PolicyName, Arn:Arn}' \
  --output table

aws iam attach-role-policy \
  --role-name eks-s3-reader-role \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/eks-lab05-s3-read"

echo "Role ARN: $ROLE_ARN"
```

### Step 4 — Create the Kubernetes Service Account with IRSA annotation

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
ROLE_ARN="arn:aws:iam::${ACCOUNT_ID}:role/eks-s3-reader-role"
BUCKET_NAME="eks-lab05-data-${ACCOUNT_ID}"

kubectl create namespace lab05

cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: ServiceAccount
metadata:
  name: s3-reader-sa
  namespace: lab05
  annotations:
    eks.amazonaws.com/role-arn: "${ROLE_ARN}"
    eks.amazonaws.com/token-expiration: "3600"
EOF

kubectl describe serviceaccount s3-reader-sa -n lab05
# Name:         s3-reader-sa
# Namespace:    lab05
# Annotations:  eks.amazonaws.com/role-arn: arn:aws:iam::123456789012:role/eks-s3-reader-role
```

### Step 5 — Deploy a pod using the IRSA service account and test S3 access

```bash
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: s3-reader-pod
  namespace: lab05
  labels:
    app: s3-reader
spec:
  serviceAccountName: s3-reader-sa
  containers:
    - name: aws-cli
      image: amazon/aws-cli:latest
      command: ["sleep", "3600"]
      resources:
        requests:
          cpu: "50m"
          memory: "64Mi"
        limits:
          cpu: "100m"
          memory: "128Mi"
      env:
        - name: AWS_REGION
          value: "us-east-1"
        - name: BUCKET_NAME
          value: "${BUCKET_NAME}"
  restartPolicy: Never
EOF

kubectl wait --for=condition=Ready pod/s3-reader-pod -n lab05 --timeout=60s

# Test: Pod should be able to read from the bucket
kubectl exec -n lab05 s3-reader-pod -- \
  aws s3 cp s3://$BUCKET_NAME/config/app-config.txt /tmp/config.txt \
  && kubectl exec -n lab05 s3-reader-pod -- cat /tmp/config.txt
# Secret config data: db_host=prod-rds.cluster.us-east-1.rds.amazonaws.com

# Confirm the pod is using IRSA (not node role)
kubectl exec -n lab05 s3-reader-pod -- \
  aws sts get-caller-identity --query '{Account:Account, Arn:Arn}' --output table
# ┌──────────────────────┬─────────────────────────────────────────────────────────────────────┐
# │  Account             │  123456789012                                                       │
# │  Arn                 │  arn:aws:sts::123456789012:assumed-role/eks-s3-reader-role/...      │
# └──────────────────────┴─────────────────────────────────────────────────────────────────────┘
```

### Step 6 — Set up Kubernetes RBAC for a read-only namespace user

```bash
# Create a Role: read-only access in lab05
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
  name: lab05-reader
  namespace: lab05
rules:
  - apiGroups: [""]
    resources: ["pods", "services", "configmaps"]
    verbs: ["get", "list", "watch"]
  - apiGroups: ["apps"]
    resources: ["deployments", "replicasets"]
    verbs: ["get", "list", "watch"]
EOF

# Bind the role to a (hypothetical) user "alice"
cat <<'EOF' | kubectl apply -f -
apiVersion: rbac.authorization.k8s.io/v1
kind: RoleBinding
metadata:
  name: lab05-reader-binding
  namespace: lab05
subjects:
  - kind: User
    name: alice
    apiGroup: rbac.authorization.k8s.io
roleRef:
  kind: Role
  name: lab05-reader
  apiGroup: rbac.authorization.k8s.io
EOF

# Check alice's permissions using auth can-i
kubectl auth can-i get pods --namespace lab05 --as alice
# yes

kubectl auth can-i delete pods --namespace lab05 --as alice
# no

kubectl auth can-i create deployments --namespace lab05 --as alice
# no
```

---

## Cleanup

```bash
ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
BUCKET_NAME="eks-lab05-data-${ACCOUNT_ID}"

kubectl delete namespace lab05

# IAM cleanup
aws iam detach-role-policy \
  --role-name eks-s3-reader-role \
  --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/eks-lab05-s3-read"
aws iam delete-policy --policy-arn "arn:aws:iam::${ACCOUNT_ID}:policy/eks-lab05-s3-read"
aws iam delete-role --role-name eks-s3-reader-role

# S3 cleanup
aws s3 rm s3://$BUCKET_NAME --recursive
aws s3 rb s3://$BUCKET_NAME
```

---

## Next Lab
➡️ [Lab 06 — Autoscaling & Storage](../06-autoscaling-storage/README.md)
