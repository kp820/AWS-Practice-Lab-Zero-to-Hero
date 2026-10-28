# Secure Log Aggregation on AWS (Private EC2 → Public EC2 → S3)

A hands-on AWS lab from the **Nautilus DevOps** team. Log files are collected from an EC2 instance in a **private VPC**, transferred over a **VPC peering connection** to an EC2 instance in a **public VPC**, and then pushed to a **private S3 bucket**, all built from the AWS Management Console.

---

## Architecture

```
 ┌──────────────────────────────┐         VPC Peering          ┌──────────────────────────────┐
 │  datacenter-priv-vpc         │   (datacenter-vpc-peering)   │  datacenter-pub-vpc          │
 │  ┌────────────────────────┐  │ <==========================> │  ┌────────────────────────┐  │
 │  │ datacenter-priv-ec2    │  │                              │  │ datacenter-pub-ec2     │  │
 │  │ /var/log/boots.log     │──┼── cron: scp (every minute) ─>│  │ ~/boots.log            │  │
 │  └────────────────────────┘  │                              │  │  IAM role:             │  │
 │  Route table: datacenter-    │                              │  │  datacenter-s3-role    │  │
 │  priv-rt                     │                              │  └───────────┬────────────┘  │
 └──────────────────────────────┘                              │   Route table: datacenter-   │
                                                               │   pub-rt  ──> Internet GW    │
                                                               └──────────────┼───────────────┘
                                                                              │ cron: aws s3 cp
                                                                              ▼
                                                     s3://datacenter-s3-logs-931010650/
                                                         datacenter-priv-vpc/boot/boots.log
```

## Resources

| Resource | Name |
|---|---|
| Private VPC *(pre-existing)* | `datacenter-priv-vpc` |
| Private subnet *(pre-existing)* | `datacenter-priv-subnet` |
| Private route table *(pre-existing)* | `datacenter-priv-rt` |
| Private EC2 *(pre-existing)* | `datacenter-priv-ec2` (Ubuntu) |
| Public VPC | `datacenter-pub-vpc` |
| Public subnet | `datacenter-pub-subnet` |
| Public route table | `datacenter-pub-rt` |
| Internet gateway | `datacenter-pub-igw` |
| Public EC2 | `datacenter-pub-ec2` |
| IAM role | `datacenter-s3-role` |
| S3 bucket | `datacenter-s3-logs-931010650` |
| VPC peering | `datacenter-vpc-peering` |

> **CIDR planning:** peered VPCs must not have overlapping CIDR blocks. This guide assumes the private VPC uses `10.0.0.0/16` and uses `10.1.0.0/16` for the public VPC. Check the real CIDR of `datacenter-priv-vpc` first and adjust if needed.

| Placeholder | Meaning |
|---|---|
| `<PRIV_CIDR>` | CIDR of `datacenter-priv-vpc` |
| `<PUB_CIDR>` | CIDR of `datacenter-pub-vpc` (`10.1.0.0/16`) |
| `<PUB_PUBLIC_IP>` | Public IPv4 of `datacenter-pub-ec2` |
| `<PUB_PRIVATE_IP>` | Private IPv4 of `datacenter-pub-ec2` |
| `<PRIV_PRIVATE_IP>` | Private IPv4 of `datacenter-priv-ec2` |
| `<REGION>` | AWS region of the lab |

---

## Step-by-step (AWS Console)

Log in to the console with the credentials from `showcreds` on the `aws-client` host, and make sure you stay in one region throughout.

### 1. Inspect the existing private environment
1. **VPC → Your VPCs** → note the IPv4 CIDR of `datacenter-priv-vpc`.
2. **EC2 → Instances** → select `datacenter-priv-ec2` and note its private IP and security group.

### 2. Create the public VPC
**VPC → Your VPCs → Create VPC → VPC only**
- Name: `datacenter-pub-vpc`
- IPv4 CIDR: `10.1.0.0/16`

### 3. Create the public subnet
**VPC → Subnets → Create subnet**
- VPC: `datacenter-pub-vpc`
- Name: `datacenter-pub-subnet`
- CIDR: `10.1.1.0/24`

Then **Actions → Edit subnet settings → Enable auto-assign public IPv4 address**.

### 4. Create and attach the internet gateway
**VPC → Internet gateways → Create internet gateway**
- Name: `datacenter-pub-igw`
- **Actions → Attach to VPC** → `datacenter-pub-vpc`

### 5. Create the public route table
**VPC → Route tables → Create route table**
- Name: `datacenter-pub-rt`, VPC: `datacenter-pub-vpc`
- **Routes → Edit routes → Add route:** `0.0.0.0/0` → Internet Gateway → `datacenter-pub-igw`
- **Subnet associations → Edit → select `datacenter-pub-subnet`**

### 6. Create the private S3 bucket
**S3 → Create bucket**
- Name: `datacenter-s3-logs-931010650`
- Region: same as the VPCs
- **Block all public access:** enabled (default)

### 7. Create the IAM role
**IAM → Roles → Create role**
- Trusted entity: **AWS service → EC2**
- Skip attaching managed policies, then name the role `datacenter-s3-role`
- Open the role → **Add permissions → Create inline policy → JSON**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::datacenter-s3-logs-931010650/*"
    }
  ]
}
```
Name the policy `datacenter-s3-putobject-policy`.

### 8. Launch the public EC2 instance
**EC2 → Launch instance**
- Name: `datacenter-pub-ec2`
- AMI: Ubuntu
- Instance type: `t2.micro`
- Key pair: `datacenter-key` (same as the private instance)
- Network: VPC `datacenter-pub-vpc`, subnet `datacenter-pub-subnet`, auto-assign public IP **Enable**
- Security group (new): inbound **SSH (22)** from `0.0.0.0/0` (for lab access) and from `<PRIV_CIDR>`
- **Advanced details → IAM instance profile:** `datacenter-s3-role`

### 9. Create the VPC peering connection
**VPC → Peering connections → Create peering connection**
- Name: `datacenter-vpc-peering`
- Requester: `datacenter-priv-vpc`, Accepter: `datacenter-pub-vpc` (same account, same region)
- Then **Actions → Accept request**

### 10. Update both route tables
| Route table | Destination | Target |
|---|---|---|
| `datacenter-priv-rt` | `<PUB_CIDR>` (`10.1.0.0/16`) | Peering connection `datacenter-vpc-peering` |
| `datacenter-pub-rt` | `<PRIV_CIDR>` | Peering connection `datacenter-vpc-peering` |

### 11. Open the private instance's security group
In the security group of `datacenter-priv-ec2`, add an inbound rule: **SSH (22)** from `<PUB_CIDR>`.

---

## Configuration on the instances

All commands below run from the `aws-client` host unless stated otherwise.

### A. Prepare the SSH key and connect to the public instance
```bash
chmod 400 /root/.ssh/datacenter-key.pem
ssh -i /root/.ssh/datacenter-key.pem ubuntu@<PUB_PUBLIC_IP>
```

### B. Install AWS CLI v2 on the public instance
```bash
sudo apt update && sudo apt install -y unzip curl
curl "https://awscli.amazonaws.com/awscliv2.zip" -o awscliv2.zip
unzip -q awscliv2.zip && sudo ./aws/install
aws --version
```

Verify the IAM role is attached (should return the role name):
```bash
curl -s http://169.254.169.254/latest/meta-data/iam/security-credentials/
```
> If IMDSv2 is enforced, request a token first:
> `TOKEN=$(curl -s -X PUT http://169.254.169.254/latest/api/token -H "X-aws-ec2-metadata-token-ttl-seconds: 60")`
> then add `-H "X-aws-ec2-metadata-token: $TOKEN"` to the call.

### C. Reach the private instance (via the public instance as a jump host)
The private instance has no public IP, so it is reached through the peering connection. From `aws-client`:

```bash
# Copy the key to the private instance (hops through the public instance)
scp -i /root/.ssh/datacenter-key.pem \
    -o ProxyCommand="ssh -i /root/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no -W %h:%p ubuntu@<PUB_PUBLIC_IP>" \
    -o StrictHostKeyChecking=no \
    /root/.ssh/datacenter-key.pem ubuntu@<PRIV_PRIVATE_IP>:/home/ubuntu/.ssh/datacenter-key.pem

# Log in to the private instance
ssh -i /root/.ssh/datacenter-key.pem \
    -o ProxyCommand="ssh -i /root/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no -W %h:%p ubuntu@<PUB_PUBLIC_IP>" \
    -o StrictHostKeyChecking=no \
    ubuntu@<PRIV_PRIVATE_IP>
```

### D. Cron job on the private instance (push log to public instance)
On `datacenter-priv-ec2`:
```bash
chmod 400 ~/.ssh/datacenter-key.pem

# Make sure the log exists and is readable by the ubuntu user
ls -l /var/log/boots.log || sudo touch /var/log/boots.log
sudo chmod 644 /var/log/boots.log

# Test the transfer manually first
scp -i ~/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no \
    /var/log/boots.log ubuntu@<PUB_PRIVATE_IP>:/home/ubuntu/boots.log

crontab -e
```
Add:
```cron
* * * * * scp -i /home/ubuntu/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no /var/log/boots.log ubuntu@<PUB_PRIVATE_IP>:/home/ubuntu/boots.log
```

### E. Cron job on the public instance (push log to S3)
On `datacenter-pub-ec2`:
```bash
# Test manually first
/usr/local/bin/aws s3 cp /home/ubuntu/boots.log \
  s3://datacenter-s3-logs-931010650/datacenter-priv-vpc/boot/boots.log --region <REGION>

crontab -e
```
Add:
```cron
* * * * * /usr/local/bin/aws s3 cp /home/ubuntu/boots.log s3://datacenter-s3-logs-931010650/datacenter-priv-vpc/boot/boots.log --region <REGION>
```

---

## Verification

1. Wait 1-2 minutes for both cron jobs to run.
2. **S3 console** → `datacenter-s3-logs-931010650` → `datacenter-priv-vpc/` → `boot/` → `boots.log` should be present.
3. Or from the CLI:
   ```bash
   aws s3 ls s3://datacenter-s3-logs-931010650/datacenter-priv-vpc/boot/
   ```
4. Inspect cron activity if needed: `grep CRON /var/log/syslog | tail`

## Troubleshooting

| Symptom | Likely cause |
|---|---|
| `scp` from private to public times out | Missing peering routes, or SSH (22) not allowed from the private CIDR in the public security group |
| Cannot SSH to the private instance | Private SG does not allow SSH from `<PUB_CIDR>`, or the peering route is missing |
| Peering shows `pending-acceptance` | Accept the request in the VPC console |
| `AccessDenied` on `s3 cp` | Role not attached to the instance, or the policy resource ARN is wrong (must end in `/*`) |
| `aws: command not found` in cron | Use the full path `/usr/local/bin/aws` |
| `Permission denied` reading the log | Run `sudo chmod 644 /var/log/boots.log` on the private instance |
| CIDR overlap error on peering | Pick a different CIDR for the public VPC |

## Security Notes
- The S3 bucket keeps **Block Public Access** enabled, and the role grants only `s3:PutObject` on this one bucket.
- For production, replace `0.0.0.0/0` SSH access with a specific IP or use SSM Session Manager, and use a dedicated key pair for the private → public transfer instead of reusing `datacenter-key`.

## Cleanup
Delete in this order to avoid dependency errors: EC2 instances → peering connection → IGW (detach, then delete) → subnet → route table → VPC → IAM role → S3 bucket (empty it first).
