# Secure Log Aggregation on AWS (Private EC2 → Public EC2 → S3)

A hands-on AWS lab from the **Datacenter DevOps** team. Log files are collected from an EC2 instance in a **private VPC**, transferred over a **VPC peering connection** to an EC2 instance in a **public VPC**, and then pushed to a **private S3 bucket**, all built from the AWS Management Console.

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

# Secure Log Aggregation on AWS: Private EC2 → Public EC2 → S3

A hands-on AWS lab from the **Nautilus DevOps** team. A log file is collected from an EC2 instance in a **private VPC**, transferred over a **VPC peering connection** to an EC2 instance in a **public VPC**, and then pushed to a **private S3 bucket**. Everything is built in the AWS Management Console, and the transfers are automated with two cron jobs.

---

**Data flow:** `boots.log` on the private instance → `scp` over `datacenter-vpc-peering` → `/home/ubuntu/boots.log` on the public instance → `aws s3 cp` using the instance's IAM role → S3.

## Resources

| Resource | Name | Notes |
|---|---|---|
| Private VPC | `datacenter-priv-vpc` | Pre-existing |
| Private subnet | `datacenter-priv-subnet` | Pre-existing |
| Private route table | `datacenter-priv-rt` | Pre-existing |
| Private EC2 | `datacenter-priv-ec2` | Pre-existing, Ubuntu |
| Public VPC | `datacenter-pub-vpc` | Created in this lab |
| Public subnet | `datacenter-pub-subnet` | Auto-assign public IPv4 enabled |
| Internet gateway | `datacenter-pub-igw` | Attached to the public VPC |
| Public route table | `datacenter-pub-rt` | `0.0.0.0/0` → IGW, plus peering route |
| Public EC2 | `datacenter-pub-ec2` | Same key pair as the private instance |
| IAM role | `datacenter-s3-role` | `s3:PutObject` only |
| S3 bucket | `datacenter-s3-logs-183190420` | Block all public access ON |
| VPC peering | `datacenter-vpc-peering` | Private ↔ public |
| Key pair | `datacenter-key` | File at `/root/.ssh/datacenter-key.pem` on `aws-client` |

## Prerequisites and variables

- Credentials from the `showcreds` command on the `aws-client` host, used to log in to the console.
- Work in **one region** the whole time. This guide uses `us-east-1`.
- **Choose non-overlapping CIDRs.** Peered VPCs cannot overlap.

| Private VPC CIDR | Use for the public VPC |
|---|---|
| `10.10.0.0/16` | `10.0.0.0/16` |
| `10.0.0.0/16` | `10.1.0.0/16` |

Values used below (replace them with yours):

| Name | Example | Meaning |
|---|---|---|
| `PUB_IP` | `18.233.160.251` | Public IPv4 of `datacenter-pub-ec2` |
| `PUB_PRIV_IP` | `10.0.2.158` | Private IPv4 of `datacenter-pub-ec2` |
| `PRIV_IP` | `10.10.1.27` | Private IPv4 of `datacenter-priv-ec2` |

---

## Part 1: Build the infrastructure (console)

### Step 1. Inspect the private side
- **VPC → Your VPCs**: note the CIDR of `datacenter-priv-vpc`.
- **EC2 → Instances**: note the private IP of `datacenter-priv-ec2`.

### Step 2. Create the public VPC
**VPC → Create VPC → VPC only**
- Name: `datacenter-pub-vpc`
- IPv4 CIDR: a non-overlapping `/16`, such as `10.0.0.0/16`

### Step 3. Create the public subnet
**VPC → Subnets → Create subnet**
- VPC: `datacenter-pub-vpc`, name: `datacenter-pub-subnet`, CIDR: for example `10.0.1.0/24`
- Then **Actions → Edit subnet settings → Enable auto-assign public IPv4 address → Save**

### Step 4. Create and attach the internet gateway
**VPC → Internet gateways → Create internet gateway**
- Name: `datacenter-pub-igw`
- **Actions → Attach to VPC → `datacenter-pub-vpc`**

### Step 5. Create the public route table
**VPC → Route tables → Create route table**
- Name: `datacenter-pub-rt`, VPC: `datacenter-pub-vpc`
- **Routes → Edit routes → Add route:** `0.0.0.0/0` → Internet Gateway → `datacenter-pub-igw`
- **Subnet associations → Edit** and select `datacenter-pub-subnet`

### Step 6. Create the S3 bucket
**S3 → Create bucket**
- Name: `datacenter-s3-logs-183190420`, same region as the VPCs
- Keep **Block all public access** turned on

### Step 7. Create the IAM role
**IAM → Roles → Create role → AWS service → EC2**
- Skip managed policies and name the role `datacenter-s3-role`
- Open the role → **Add permissions → Create inline policy → JSON**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:PutObject",
      "Resource": "arn:aws:s3:::datacenter-s3-logs-183190420/*"
    }
  ]
}
```
- Name the policy `datacenter-s3-putobject-policy`

### Step 8. Launch the public EC2 instance
**EC2 → Launch instance**
- Name: `datacenter-pub-ec2`, AMI: Ubuntu, type: `t2.micro`
- Key pair: `datacenter-key` (the same one as the private instance)
- Network: VPC `datacenter-pub-vpc`, subnet `datacenter-pub-subnet`, auto-assign public IP **Enable**
- Security group: SSH (22) from `0.0.0.0/0` and from the private VPC CIDR
- **Advanced details → IAM instance profile → `datacenter-s3-role`**

> Do not skip the IAM role. Without it, uploads fail with `Unable to locate credentials`. If you forget, attach it later via **Actions → Security → Modify IAM role**.

Note the instance's **public IP** and **private IP**.

### Step 9. Create the VPC peering
**VPC → Peering connections → Create peering connection**
- Name: `datacenter-vpc-peering`
- Requester: `datacenter-priv-vpc`, accepter: `datacenter-pub-vpc`
- **Actions → Accept request**, then wait for the status **Active**

### Step 10. Add the peering routes
**VPC → Route tables**

| Route table | Destination | Target |
|---|---|---|
| `datacenter-priv-rt` | public VPC CIDR | `datacenter-vpc-peering` |
| `datacenter-pub-rt` | private VPC CIDR | `datacenter-vpc-peering` |

Each table points at the **other** VPC's CIDR. Also confirm `datacenter-priv-subnet` is associated with `datacenter-priv-rt`.

### Step 11. Allow SSH into the private instance
In the security group of `datacenter-priv-ec2`, add an inbound rule: **SSH (22)** from the **public VPC CIDR**.

---

## Part 2: Configure the instances (terminal)

All commands start on the `aws-client` host. After each login, check the prompt to be sure you are on the right machine:
- Public instance: `ubuntu@ip-10-0-2-158`
- Private instance: `ubuntu@ip-10-10-1-27`

### Step 12. Log in to the public instance
```bash
chmod 400 /root/.ssh/datacenter-key.pem
ssh -i /root/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no ubuntu@<PUB_IP>
```

### Step 13. Install the AWS CLI (on the public instance)
```bash
which aws || (sudo apt update && sudo apt install -y unzip curl && \
  curl -s "https://awscli.amazonaws.com/awscliv2.zip" -o awscliv2.zip && \
  unzip -q awscliv2.zip && sudo ./aws/install)
which aws
aws --version
exit
```
Note what `which aws` prints (`/usr/bin/aws` or `/usr/local/bin/aws`).

### Step 14. Copy the key to the private instance (from `aws-client`)
The private instance has no public IP, so the copy hops through the public instance:
```bash
scp -i /root/.ssh/datacenter-key.pem \
 -o ProxyCommand="ssh -i /root/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no -W %h:%p ubuntu@<PUB_IP>" \
 -o StrictHostKeyChecking=no \
 /root/.ssh/datacenter-key.pem ubuntu@<PRIV_IP>:/home/ubuntu/.ssh/datacenter-key.pem
```

### Step 15. Log in to the private instance (from `aws-client`)
```bash
ssh -i /root/.ssh/datacenter-key.pem \
 -o ProxyCommand="ssh -i /root/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no -W %h:%p ubuntu@<PUB_IP>" \
 -o StrictHostKeyChecking=no \
 ubuntu@<PRIV_IP>
```

### Step 16. Cron job 1: private → public (on the private instance)

**a) Prepare the key and the log file**
```bash
chmod 400 ~/.ssh/datacenter-key.pem
ls -l ~/.ssh/datacenter-key.pem /var/log/boots.log
```
If `boots.log` is missing: `sudo touch /var/log/boots.log && sudo chmod 644 /var/log/boots.log`.

**b) Test the copy by hand. It must finish with no error before you continue.**
```bash
scp -i /home/ubuntu/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no \
  /var/log/boots.log ubuntu@<PUB_PRIV_IP>:/home/ubuntu/boots.log
```

**c) Install the cron job**
```bash
cat <<'EOF' | crontab -
* * * * * /usr/bin/scp -i /home/ubuntu/.ssh/datacenter-key.pem -o StrictHostKeyChecking=no /var/log/boots.log ubuntu@<PUB_PRIV_IP>:/home/ubuntu/boots.log >> /home/ubuntu/scp-upload.log 2>&1
EOF
crontab -l
exit
```
Replace `<PUB_PRIV_IP>` with the real IP (for example `10.0.2.158`) **before** pasting. `crontab -l` must show exactly one line with a real IP and no `<>` left.

### Step 17. Cron job 2: public → S3 (on the public instance)

Log in again from `aws-client`:
```bash
ssh -i /root/.ssh/datacenter-key.pem ubuntu@<PUB_IP>
```

**a) Check the file arrived (wait a minute if it's missing)**
```bash
ls -l /home/ubuntu/boots.log
```

**b) Test the upload by hand. You must see an `upload:` line.**
```bash
aws s3 cp /home/ubuntu/boots.log \
  s3://datacenter-s3-logs-183190420/datacenter-priv-vpc/boot/boots.log --region us-east-1
```

**c) Install the cron job.** `$(which aws)` is expanded by your shell right now, so the saved line has the full path:
```bash
echo "* * * * * $(which aws) s3 cp /home/ubuntu/boots.log s3://datacenter-s3-logs-183190420/datacenter-priv-vpc/boot/boots.log --region us-east-1 >> /home/ubuntu/s3-upload.log 2>&1" | crontab -
crontab -l
```
The saved line must start with a full path (such as `/usr/bin/aws`) and must say `aws`, not `scp`.

> `crontab -` replaces the machine's entire crontab. That is fine here because each machine has only one job.

---

## Verification

1. Wait 1-2 minutes for both cron jobs to run.
2. On the public instance:
   ```bash
   tail -3 /home/ubuntu/s3-upload.log
   ```
   Expect `upload:` lines.
3. **S3 console** → `datacenter-s3-logs-183190420` → `datacenter-priv-vpc/` → `boot/` → `boots.log` must exist.
4. Optional end-to-end test, on the private instance:
   ```bash
   echo "test $(date)" | sudo tee -a /var/log/boots.log
   ```
   After about 2 minutes, the object's **Last modified** time in S3 should update.

> The role only has `s3:PutObject`, so `aws s3 ls` from the public instance is denied (`s3:ListBucket`). That is expected and correct. Verify in the S3 console instead.

---

## Troubleshooting

| Symptom | Cause | Fix |
|---|---|---|
| `Unable to locate credentials` on upload | IAM role not attached to the public instance | EC2 → instance → Actions → Security → Modify IAM role → `datacenter-s3-role` |
| `AccessDenied ... s3:ListBucket` | Role only allows `PutObject` | Expected; check the S3 console instead |
| `scp` or SSH to the private instance hangs | Missing peering route, peering not Active, or SSH (22) blocked | Check both route tables, peering status, and the private security group |
| `Permission denied (publickey)` | Key missing or wrong on the private instance | Redo Step 14, then `chmod 400` the key |
| `/home/ubuntu/boots.log` missing on the public instance | Private-side cron not delivering | On the private instance run `tail /home/ubuntu/scp-upload.log` and test `scp` by hand |
| `bash: syntax error near unexpected token 'newline'` | A `<placeholder>` was pasted literally | Replace the `<...>` with the real value |
| `identity file -o not accessible` or `invalid characters` | Shell variables used on the wrong machine (they exist only on `aws-client`) | Run those commands from `aws-client`, or use literal values |
| `aws: command not found` or silent cron failure | Cron has a minimal `PATH` | Use the full path from `which aws` in the cron line |
| `tail: cannot open s3-upload.log` right after install | Cron hasn't run yet; the log is created on first run | Wait a minute and retry |
| Peering creation fails with a CIDR error | VPC CIDRs overlap | Use a different CIDR for the public VPC |

## Lessons learned

- **Test by hand before cron.** If the manual command fails, the cron job will fail too, only silently.
- **Always use full paths in cron** (`/usr/bin/scp`, `/usr/bin/aws`). Cron does not use your login shell's `PATH`.
- **Log cron output** with `>> file 2>&1`. Without it, errors disappear (no MTA is installed on these instances).
- **Check your prompt after every login.** Several errors came from running commands on the wrong instance.
- **Shell variables do not travel over SSH.** Variables set on `aws-client` do not exist inside the instances.
- **Attach the IAM role at launch**, and give it the narrowest permission that works (`PutObject` on one bucket).
- **`crontab -` overwrites the whole crontab.** Use `crontab -e` if a machine ever needs more than one job.

## Security notes
- The S3 bucket keeps **Block all public access** enabled, and the role grants only `s3:PutObject` on this one bucket.
- For production, replace `0.0.0.0/0` SSH access with a specific IP or use SSM Session Manager, and use a dedicated key pair for the private → public transfer rather than copying the main key.

## Cleanup
Delete in this order to avoid dependency errors: EC2 instances → peering connection → internet gateway (detach, then delete) → subnet → route table → VPC → IAM role → S3 bucket (empty it first).
