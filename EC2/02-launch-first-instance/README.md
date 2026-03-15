# Lab 02 — Launch Your First EC2 Instance: CLI, SSH & User-Data

🟢 **Difficulty:** Beginner | **Time:** 45 min | **Cost:** ~$0.01 (t3.micro, ~15 min)

---

## Goal

Launch a fully functioning EC2 instance using only the AWS CLI: create a security group, select the latest Amazon Linux 2023 AMI, bootstrap a web server via user-data, SSH in, and cleanly terminate everything.

---

## Concepts

### Instance Launch Flow

```
aws ec2 run-instances
        │
        ├── AMI ID           (what OS/software to start from)
        ├── Instance Type    (how much CPU / RAM)
        ├── Key Pair         (how you SSH in)
        ├── Security Group   (what traffic is allowed)
        ├── Subnet           (where in the VPC it lives)
        └── User Data        (bootstrap script at first boot)
```

### User Data Execution

```
First Boot
    │
    ▼
cloud-init reads /etc/cloud/cloud.cfg
    │
    ▼
Runs user-data script as root
    │
    ▼
Logs written to /var/log/cloud-init-output.log
```

### Security Group Rule Logic

| Direction | Protocol | Port | Source/Dest | Effect |
|-----------|----------|------|-------------|--------|
| Inbound | TCP | 22 | Your IP/32 | Allow SSH |
| Inbound | TCP | 80 | 0.0.0.0/0 | Allow HTTP |
| Outbound | All | All | 0.0.0.0/0 | Allow all egress |

---

## Architecture

```
Your Laptop
    │  SSH :22 / HTTP :80
    ▼
Internet Gateway
    │
    ▼
Public Subnet  10.0.1.0/24
┌──────────────────────────────────────────┐
│  EC2 t3.micro                            │
│  Amazon Linux 2023                       │
│  Security Group: sg-lab02                │
│  EBS gp3 8 GiB (root)                   │
│                                          │
│  User-Data bootstraps:                   │
│    dnf install -y httpd                  │
│    systemctl enable --now httpd          │
│    echo "Hello EC2" > /var/www/html/...  │
└──────────────────────────────────────────┘
```

---

## Prerequisites

- Key pair `ec2-labs-key` created in Lab 01 setup (`.pem` at `~/ec2-labs-key.pem`)
- Default VPC exists (or substitute `--subnet-id` with your own)

---

## PoC Steps

### Step 1 — Gather environment values

```bash
export AWS_DEFAULT_REGION=us-east-1

# Latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters \
    "Name=name,Values=al2023-ami-2023.*-x86_64" \
    "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' \
  --output text)
echo "AMI: $AMI_ID"

# Default VPC ID
VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' \
  --output text)
echo "VPC: $VPC_ID"

# First public subnet in the default VPC
SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" \
            "Name=defaultForAz,Values=true" \
  --query 'Subnets[0].SubnetId' \
  --output text)
echo "Subnet: $SUBNET_ID"

# Your current public IP for SSH allow-listing
MY_IP=$(curl -s https://checkip.amazonaws.com)
echo "My IP: $MY_IP"
```

Expected output:
```
AMI: ami-0abcdef1234567890
VPC: vpc-0a1b2c3d4e5f
Subnet: subnet-0a1b2c3d4e5f
My IP: 203.0.113.42
```

### Step 2 — Create a security group

```bash
SG_ID=$(aws ec2 create-security-group \
  --group-name "ec2-lab02-sg" \
  --description "Lab 02 security group" \
  --vpc-id "$VPC_ID" \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ec2-lab02-sg},{Key=Lab,Value=EC2-02}]" \
  --query 'GroupId' \
  --output text)
echo "SG: $SG_ID"

# Allow SSH from your IP only
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 22 \
  --cidr "${MY_IP}/32"

# Allow HTTP from anywhere (for the web server demo)
aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" \
  --protocol tcp \
  --port 80 \
  --cidr "0.0.0.0/0"

echo "Security group rules created"
```

### Step 3 — Write the user-data bootstrap script

```bash
cat > /tmp/userdata.sh << 'USERDATA'
#!/bin/bash
set -euxo pipefail
dnf update -y
dnf install -y httpd

# Create a simple page showing instance metadata (IMDSv2)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id)
AZ=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/placement/availability-zone)
ITYPE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-type)

cat > /var/www/html/index.html << HTML
<html><body style="font-family:monospace;padding:20px;">
<h2>Hello from EC2!</h2>
<p>Instance ID : ${INSTANCE_ID}</p>
<p>Type        : ${ITYPE}</p>
<p>AZ          : ${AZ}</p>
<p>Bootstrapped: $(date -u)</p>
</body></html>
HTML

systemctl enable --now httpd
USERDATA

echo "User-data script written"
```

### Step 4 — Launch the instance

```bash
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.micro \
  --key-name ec2-labs-key \
  --security-group-ids "$SG_ID" \
  --subnet-id "$SUBNET_ID" \
  --associate-public-ip-address \
  --user-data file:///tmp/userdata.sh \
  --block-device-mappings '[{"DeviceName":"/dev/xvda","Ebs":{"VolumeSize":8,"VolumeType":"gp3","DeleteOnTermination":true}}]' \
  --tag-specifications \
    'ResourceType=instance,Tags=[{Key=Name,Value=ec2-lab02},{Key=Lab,Value=EC2-02}]' \
    'ResourceType=volume,Tags=[{Key=Name,Value=ec2-lab02-root},{Key=Lab,Value=EC2-02}]' \
  --query 'Instances[0].InstanceId' \
  --output text)

echo "Launched: $INSTANCE_ID"
```

Expected output:
```
Launched: i-0a1b2c3d4e5f67890
```

### Step 5 — Wait for the instance to reach running state

```bash
echo "Waiting for instance to be running..."
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)
echo "Public IP: $PUBLIC_IP"

# Describe the instance
aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[*].Instances[*].{ID:InstanceId,State:State.Name,IP:PublicIpAddress,Type:InstanceType,AZ:Placement.AvailabilityZone}' \
  --output table
```

Expected output:
```
--------------------------------------------------------------------
|                       DescribeInstances                         |
+---------+----------+---------------+-----------+----------------+
|   AZ    |    ID    |     IP        |   State   |     Type       |
+---------+----------+---------------+-----------+----------------+
|us-east-1a|i-0a1b... |203.0.113.10   |running    |t3.micro        |
+---------+----------+---------------+-----------+----------------+
```

### Step 6 — SSH into the instance

```bash
# Wait another 30 seconds for cloud-init / SSH daemon to fully start
sleep 30

ssh -o StrictHostKeyChecking=no \
    -i ~/ec2-labs-key.pem \
    ec2-user@"$PUBLIC_IP"
```

Once inside the instance:

```bash
# Verify the web server is running
systemctl status httpd

# Check user-data log
sudo tail -30 /var/log/cloud-init-output.log

# Check the page
curl localhost

# View instance metadata via IMDSv2
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
  -H "X-aws-ec2-metadata-token-ttl-seconds: 300")
curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
  http://169.254.169.254/latest/meta-data/instance-id

exit
```

### Step 7 — Test the web page from your laptop

```bash
curl "http://$PUBLIC_IP"
```

Expected output:
```html
<html><body style="font-family:monospace;padding:20px;">
<h2>Hello from EC2!</h2>
<p>Instance ID : i-0a1b2c3d4e5f67890</p>
<p>Type        : t3.micro</p>
<p>AZ          : us-east-1a</p>
<p>Bootstrapped: Sun Mar 15 08:05:22 UTC 2026</p>
</body></html>
```

---

## Cleanup

```bash
# Terminate the instance
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"
echo "Instance terminated"

# Delete the security group
aws ec2 delete-security-group --group-id "$SG_ID"
echo "Security group deleted"
```

---

## Key Takeaways

- `run-instances` is the single CLI command that combines AMI, instance type, key pair, SG, subnet, and user-data
- User-data runs as root at first boot — ideal for package install and config
- Always restrict SSH to your IP (`/32`) — never use `0.0.0.0/0` for port 22
- Use `aws ec2 wait instance-running` instead of polling in scripts
- IMDSv2 requires a token PUT request before reading metadata (security improvement over IMDSv1)

---

➡️ Next: [Lab 03 — Storage Deep-Dive: EBS, Instance Store & Snapshots](../03-storage-and-ebs/README.md)
