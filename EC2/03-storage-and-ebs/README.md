# Lab 03 — Storage Deep-Dive: EBS, Instance Store & Snapshots

🟡 **Difficulty:** Intermediate | **Time:** 60 min | **Cost:** ~$0.05 (gp3 volumes, short-lived)

---

## Goal

Create and attach an EBS volume, benchmark it, take a snapshot, restore from snapshot, and understand the trade-offs between EBS types and instance store.

---

## Concepts

### EBS Volume Types

| Type | Use Case | Max IOPS | Max Throughput | Cost |
|------|---------|----------|----------------|------|
| **gp3** | General purpose, most workloads | 16,000 | 1,000 MB/s | $0.08/GB-mo |
| **gp2** | Legacy general purpose | 16,000 | 250 MB/s | $0.10/GB-mo |
| **io2 Block Express** | Databases, latency-sensitive | 256,000 | 4,000 MB/s | $0.125/GB-mo |
| **st1** | Streaming throughput (log processing) | 500 | 500 MB/s | $0.045/GB-mo |
| **sc1** | Cold archival storage | 250 | 250 MB/s | $0.015/GB-mo |

> **Rule of thumb:** gp3 for almost everything. Set IOPS and throughput independently — no need to overprovision GB just to get IOPS (unlike gp2).

### EBS vs Instance Store

| Feature | EBS | Instance Store |
|---------|-----|----------------|
| Persistence | Survives stop/start | Lost on stop/terminate/failure |
| Max size | 64 TiB (io2 BE) | Varies by instance (up to 60 TB on i4i) |
| Latency | ~1 ms (gp3) | ~100 µs (local NVMe) |
| Snapshots | Yes (S3-backed) | No |
| Replication | Multi-AZ with snapshots | None |
| Cost | Pay per GB + IOPS | Included in instance price |
| Best for | Databases, boot volumes | Temp scratch, buffer cache |

### Snapshot Mechanics

```
EBS Volume (gp3, 20 GiB, AZ us-east-1a)
         │
         │  CreateSnapshot
         ▼
S3 (AWS-managed, cross-AZ durable)
   [Snapshot: snap-0abc...]
         │
         │  CreateVolume --snapshot-id
         ▼
New EBS Volume (any AZ, any size >= original)
```

Snapshots are incremental — only changed blocks are stored after the first full snapshot.

---

## Architecture

```
EC2 Instance (t3.medium, us-east-1a)
├── /dev/xvda  8 GiB gp3  (root volume, from AMI)
└── /dev/xvdf  20 GiB gp3 (data volume — created in this lab)
         │
         │  Snapshot
         ▼
    snap-0abc123
         │
         │  Restore
         ▼
/dev/xvdg  20 GiB gp3  (restored volume, attached as second data vol)
```

---

## PoC Steps

### Step 1 — Launch a test instance (reuse environment from Lab 02)

```bash
export AWS_DEFAULT_REGION=us-east-1

AMI_ID=$(aws ec2 describe-images \
  --owners amazon \
  --filters "Name=name,Values=al2023-ami-2023.*-x86_64" "Name=state,Values=available" \
  --query 'sort_by(Images,&CreationDate)[-1].ImageId' --output text)

VPC_ID=$(aws ec2 describe-vpcs \
  --filters "Name=isDefault,Values=true" \
  --query 'Vpcs[0].VpcId' --output text)

SUBNET_ID=$(aws ec2 describe-subnets \
  --filters "Name=vpc-id,Values=$VPC_ID" "Name=defaultForAz,Values=true" \
  --query 'Subnets[0].SubnetId' --output text)

AZ=$(aws ec2 describe-subnets \
  --subnet-ids "$SUBNET_ID" \
  --query 'Subnets[0].AvailabilityZone' --output text)

MY_IP=$(curl -s https://checkip.amazonaws.com)

SG_ID=$(aws ec2 create-security-group \
  --group-name "ec2-lab03-sg" \
  --description "Lab 03 storage SG" \
  --vpc-id "$VPC_ID" \
  --tag-specifications "ResourceType=security-group,Tags=[{Key=Name,Value=ec2-lab03-sg},{Key=Lab,Value=EC2-03}]" \
  --query 'GroupId' --output text)

aws ec2 authorize-security-group-ingress \
  --group-id "$SG_ID" --protocol tcp --port 22 --cidr "${MY_IP}/32"

INSTANCE_ID=$(aws ec2 run-instances \
  --image-id "$AMI_ID" \
  --instance-type t3.medium \
  --key-name ec2-labs-key \
  --security-group-ids "$SG_ID" \
  --subnet-id "$SUBNET_ID" \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=ec2-lab03},{Key=Lab,Value=EC2-03}]' \
  --query 'Instances[0].InstanceId' --output text)

echo "Instance: $INSTANCE_ID  AZ: $AZ"
aws ec2 wait instance-running --instance-ids "$INSTANCE_ID"

PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Public IP: $PUBLIC_IP"
```

### Step 2 — Create a gp3 EBS volume and attach it

```bash
# Create a 20 GiB gp3 volume — must be in the SAME AZ as the instance
VOL_ID=$(aws ec2 create-volume \
  --size 20 \
  --volume-type gp3 \
  --iops 3000 \
  --throughput 125 \
  --availability-zone "$AZ" \
  --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=ec2-lab03-data},{Key=Lab,Value=EC2-03}]" \
  --query 'VolumeId' \
  --output text)

echo "Volume: $VOL_ID"

# Wait for the volume to be available
aws ec2 wait volume-available --volume-ids "$VOL_ID"

# Attach it as /dev/xvdf
aws ec2 attach-volume \
  --instance-id "$INSTANCE_ID" \
  --volume-id "$VOL_ID" \
  --device /dev/xvdf

aws ec2 wait volume-in-use --volume-ids "$VOL_ID"
echo "Volume attached"
```

### Step 3 — Format, mount, and write data

SSH into the instance:

```bash
ssh -o StrictHostKeyChecking=no -i ~/ec2-labs-key.pem ec2-user@"$PUBLIC_IP"
```

Inside the instance:

```bash
# Confirm the new block device is visible
lsblk
# Expected: xvdf  20G  disk  (unmounted)

# Create a filesystem
sudo mkfs.ext4 -L data-vol /dev/xvdf

# Mount it
sudo mkdir -p /data
sudo mount /dev/xvdf /data
sudo chown ec2-user:ec2-user /data

# Persist mount across reboots
echo "LABEL=data-vol  /data  ext4  defaults,nofail  0  2" | sudo tee -a /etc/fstab

# Write 1 GB of test data
dd if=/dev/urandom of=/data/testfile.dat bs=1M count=1024 status=progress
ls -lh /data/

# Verify filesystem usage
df -h /data
```

Expected output (partial):
```
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvdf        20G  1.1G   18G   6% /data
```

### Step 4 — Quick I/O benchmark (fio)

```bash
# Install fio
sudo dnf install -y fio

# Sequential write benchmark
fio --name=seq-write \
    --directory=/data \
    --rw=write \
    --bs=128k \
    --size=512m \
    --numjobs=1 \
    --runtime=30 \
    --time_based \
    --output-format=normal \
  | grep -E "WRITE:|bw="

# Random read IOPS benchmark
fio --name=rand-read \
    --directory=/data \
    --rw=randread \
    --bs=4k \
    --size=512m \
    --numjobs=4 \
    --runtime=30 \
    --time_based \
    --iodepth=32 \
    --output-format=normal \
  | grep -E "READ:|iops="
```

Expected output (gp3 at 3000 IOPS, 125 MB/s):
```
  WRITE: bw=125MiB/s (131MB/s), 125MiB/s-125MiB/s (131MB/s-131MB/s)
  READ: IOPS=3000, BW=11.7MiB/s (12.3MB/s)
```

```bash
exit
```

### Step 5 — Take a snapshot

```bash
SNAP_ID=$(aws ec2 create-snapshot \
  --volume-id "$VOL_ID" \
  --description "Lab 03 data volume snapshot" \
  --tag-specifications "ResourceType=snapshot,Tags=[{Key=Name,Value=ec2-lab03-snap},{Key=Lab,Value=EC2-03}]" \
  --query 'SnapshotId' \
  --output text)

echo "Snapshot: $SNAP_ID"

# Wait for completion (may take 1-2 min for 20 GiB with 1 GiB of data)
aws ec2 wait snapshot-completed --snapshot-ids "$SNAP_ID"
echo "Snapshot complete"

# Describe the snapshot
aws ec2 describe-snapshots \
  --snapshot-ids "$SNAP_ID" \
  --query 'Snapshots[*].{ID:SnapshotId,Size:VolumeSize,State:State,StartTime:StartTime}' \
  --output table
```

### Step 6 — Restore from snapshot to a new volume

```bash
# Create a new volume from the snapshot (can change AZ, size, type)
RESTORED_VOL_ID=$(aws ec2 create-volume \
  --snapshot-id "$SNAP_ID" \
  --size 20 \
  --volume-type gp3 \
  --availability-zone "$AZ" \
  --tag-specifications "ResourceType=volume,Tags=[{Key=Name,Value=ec2-lab03-restored},{Key=Lab,Value=EC2-03}]" \
  --query 'VolumeId' \
  --output text)

echo "Restored volume: $RESTORED_VOL_ID"
aws ec2 wait volume-available --volume-ids "$RESTORED_VOL_ID"

# Attach as /dev/xvdg
aws ec2 attach-volume \
  --instance-id "$INSTANCE_ID" \
  --volume-id "$RESTORED_VOL_ID" \
  --device /dev/xvdg

aws ec2 wait volume-in-use --volume-ids "$RESTORED_VOL_ID"
echo "Restored volume attached"
```

SSH back in and verify the data survived:

```bash
ssh -i ~/ec2-labs-key.pem ec2-user@"$PUBLIC_IP"

# Mount the restored volume (no format needed — filesystem already there)
sudo mkdir -p /data-restored
sudo mount /dev/xvdg /data-restored

ls -lh /data-restored/
# Expected: testfile.dat  1.0G

# Verify file integrity
md5sum /data/testfile.dat /data-restored/testfile.dat
# Both hashes should match

exit
```

---

## Cleanup

```bash
# Detach and delete volumes
aws ec2 detach-volume --volume-id "$VOL_ID"
aws ec2 detach-volume --volume-id "$RESTORED_VOL_ID"
sleep 10
aws ec2 delete-volume --volume-id "$VOL_ID"
aws ec2 delete-volume --volume-id "$RESTORED_VOL_ID"

# Delete snapshot
aws ec2 delete-snapshot --snapshot-id "$SNAP_ID"

# Terminate instance and SG
aws ec2 terminate-instances --instance-ids "$INSTANCE_ID"
aws ec2 wait instance-terminated --instance-ids "$INSTANCE_ID"
aws ec2 delete-security-group --group-id "$SG_ID"
echo "Cleanup complete"
```

---

## Key Takeaways

- EBS volumes must be in the same AZ as the instance they attach to
- gp3 decouples IOPS and throughput from size — tune independently
- Snapshots are S3-backed and incremental; they enable cross-AZ and cross-region restores
- Always add `nofail` to `/etc/fstab` entries for non-root EBS volumes to prevent boot failures
- Use `lsblk` to view block devices; `df -h` for mounted filesystem usage

---

➡️ Next: [Lab 04 — AMIs, Auto Scaling Groups & Launch Templates](../04-ami-snapshots-autoscaling/README.md)
