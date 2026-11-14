# Day 50: Expand EC2 Root EBS Volume from 8 GiB to 12 GiB

## Objective

The objective of this lab is to increase the root EBS volume attached to an existing Amazon EC2 instance from **8 GiB to 12 GiB** without stopping the instance.

We will also extend the root partition and filesystem inside Linux so that the additional storage is available for use.

## Architecture

1. Identify the running EC2 instance.
2. Find its attached root EBS volume.
3. Modify the EBS volume from 8 GiB to 12 GiB.
4. Connect to the EC2 instance using SSH.
5. Extend the root partition.
6. Extend the root filesystem.
7. Verify the new available storage.

## Prerequisites

- AWS CLI configured on the AWS client.
- Running EC2 instance.
- SSH private key available at `/root/nautilus-keypair.pem`.
- Permissions to describe instances, describe volumes, and modify EBS volumes.

## Step 1: Identify the EC2 Instance

Find the instance using its Name tag:

```bash
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=nautilus-ec2" \
            "Name=instance-state-name,Values=running" \
  --query 'Reservations[].Instances[].[InstanceId,State.Name,RootDeviceName]' \
  --output table
```

Save the instance ID:

```bash
INSTANCE_ID=<your-instance-id>
```

## Step 2: Identify the Root EBS Volume

Retrieve the root device name:

```bash
ROOT_DEVICE=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].RootDeviceName' \
  --output text)

echo "$ROOT_DEVICE"
```

Find the EBS volume attached to that root device:

```bash
VOLUME_ID=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query "Reservations[0].Instances[0].BlockDeviceMappings[?DeviceName=='${ROOT_DEVICE}'].Ebs.VolumeId | [0]" \
  --output text)

echo "$VOLUME_ID"
```

Check the existing volume size:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[].[VolumeId,Size,State,AvailabilityZone]' \
  --output table
```

Expected initial size:

```text
8 GiB
```

## Step 3: Expand the EBS Volume to 12 GiB

Modify the volume:

```bash
aws ec2 modify-volume \
  --volume-id "$VOLUME_ID" \
  --size 12
```

Check the modification status:

```bash
aws ec2 describe-volumes-modifications \
  --volume-ids "$VOLUME_ID" \
  --query 'VolumesModifications[].[VolumeId,OriginalSize,TargetSize,ModificationState]' \
  --output table
```

Wait for the modification to complete:

```bash
aws ec2 wait volume-modified \
  --volume-ids "$VOLUME_ID"
```

Verify the new EBS volume size:

```bash
aws ec2 describe-volumes \
  --volume-ids "$VOLUME_ID" \
  --query 'Volumes[].[VolumeId,Size,State]' \
  --output table
```

Expected volume size:

```text
12 GiB
```

**Important:** EBS volume modification can be performed while the instance is running. There is no need to stop the EC2 instance for this size expansion.

## Step 4: Connect to the EC2 Instance Using SSH

Retrieve the public IP address:

```bash
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids "$INSTANCE_ID" \
  --query 'Reservations[0].Instances[0].PublicIpAddress' \
  --output text)

echo "$PUBLIC_IP"
```

Set the correct key permissions:

```bash
chmod 400 /root/<labname>-keypair.pem
```

Connect to the instance and its amazon linux vm:

```bash
ssh -i /root/nautilus-keypair.pem ec2-user@$PUBLIC_IP
```

If the instance uses another Linux distribution, use its appropriate SSH username.

## Step 5: Check the Current Disk and Partition

Inside the EC2 instance, execute:

```bash
lsblk
```

Check the root filesystem:

```bash
df -h /
```

Identify the root filesystem device and filesystem type:

```bash
findmnt -no SOURCE,FSTYPE /
```

## Step 6: Extend the Root Partition
For a Xen root disk:

```bash
sudo growpart /dev/xvda 1
```

## Step 7: Extend the Root Filesystem

For an **XFS** filesystem:

```bash
sudo xfs_growfs /
```

These commands extend the filesystem to use the expanded partition.

## Step 8: Verify the Final Storage

Check the disk layout:

```bash
lsblk
```

Check the root filesystem:

```bash
df -h /
```

Check the root filesystem source:

```bash
findmnt /
```

Expected result:

- Original EBS volume: 8 GiB
- Expanded EBS volume: 12 GiB
- Root partition: expanded
- Root filesystem: expanded and using the additional space

Note: Linux filesystem tools reserve some space for metadata and system usage, so the size shown by `df -h` may be slightly less than 12 GiB.

## Final Verification Checklist

- [ ] EC2 instance identified successfully.
- [ ] Root EBS volume identified successfully.
- [ ] Original volume size confirmed as 8 GiB.
- [ ] EBS volume successfully expanded to 12 GiB.
- [ ] SSH connection established.
- [ ] Root partition expanded successfully.
- [ ] Root filesystem expanded successfully.
- [ ] Final storage verified using `lsblk` and `df -h /`.

## Conclusion

In this lab, we successfully expanded an EC2 root EBS volume from 8 GiB to 12 GiB without stopping the instance.

We used AWS CLI to identify and modify the EBS volume, SSH to access the Linux instance, `growpart` to expand the root partition, and `resize2fs` or `xfs_growfs` to extend the filesystem.

This demonstrates how AWS administrators can increase EC2 storage capacity while minimizing service disruption.

---

**Technologies Used:** Amazon EC2, Amazon EBS, AWS CLI, Linux, SSH, growpart, resize2fs, XFS.
