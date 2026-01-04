# Q2: EC2 SSH Connectivity

## Lab Overview
- **Difficulty:** Beginner
- **Estimated Time:** 30-35 minutes
- **AWS Services:** EC2, VPC, Security Groups, Key Pairs
- **Region:** us-east-1
- **Cost:** Free Tier (t2.micro 750 hours)

## Prerequisites Check
- [ ] Completed Q1 VPC setup
- [ ] SSH client installed (PuTTY on Windows, OpenSSH on Linux/Mac)
- [ ] AWS CLI configured (optional)
- [ ] Billing alarm set at $5

## Learning Objectives
- Create and manage EC2 key pairs
- Configure least-privilege security groups for SSH
- Launch EC2 in public subnet and connect via SSH
- Validate internet connectivity from the instance

## Architecture Overview
```mermaid
flowchart LR
    Admin[Admin IP /32] -->|SSH 22| SG[Security Group]
    SG --> EC2[EC2 (Public IP)]
```

## Step-by-Step Console Instructions

### Step 1: Create Key Pair
**Console Navigation:** EC2 Dashboard → Key Pairs → Create key pair

**Detailed Steps:**
1. Name: `practice-key-q2`
2. Key pair type: RSA
3. Private key file format: .pem (OpenSSH) or .ppk (PuTTY)
4. Click "Create key pair"
5. Browser downloads key file automatically

[SCREENSHOT: Key pair creation confirmation]

- Save key file securely; cannot be downloaded again
- Linux/Mac: `chmod 400 practice-key-q2.pem`
- Windows PuTTY: convert .pem to .ppk using PuTTYgen if needed

### Step 2: Get Your Public IP Address
1. Open https://checkip.amazonaws.com
2. Note IP (e.g., 203.0.113.45) → add /32 (203.0.113.45/32)

[SCREENSHOT: Your public IP]

Tip: Use "My IP" in console to auto-detect.

### Step 3: Create Security Group
**Console Navigation:** EC2 Dashboard → Security Groups → Create security group

**Detailed Steps:**
1. Name: `practice-ssh-sg`
2. Description: "SSH access from admin IP only"
3. VPC: `practice-vpc-q1`
4. Inbound rule:
   - Type: SSH, Port: 22
   - Source: Custom → your IP/32
   - Description: "Admin SSH access"
5. Outbound: default (all traffic)
6. Tags: Name = `practice-ssh-sg`
7. Create security group

[SCREENSHOT: Security group with SSH rule]

### Step 4: Launch EC2 Instance
**Console Navigation:** EC2 Dashboard → Instances → Launch instances

**Detailed Steps:**
1. Name: `practice-ec2-q2`
2. AMI: Amazon Linux 2023 (Free Tier), 64-bit (x86)
3. Instance type: `t2.micro`
4. Key pair: select `practice-key-q2`
5. Network settings (Edit):
   - VPC: `practice-vpc-q1`
   - Subnet: `practice-public-1a`
   - Auto-assign public IP: Enable
   - Firewall: Select existing SG → `practice-ssh-sg`
6. Storage: 8 GiB gp3 (default)
7. Advanced: leave defaults
8. Launch instance

[SCREENSHOT: Launch success message with instance ID]

### Step 5: Wait for Instance Running State
1. Instances list → wait for "Running" and "2/2 checks passed"
2. Note Public IPv4 address

[SCREENSHOT: Instance running with public IP]

### Step 6: Connect via SSH
- **Linux/Mac:**
  - `ssh -i practice-key-q2.pem ec2-user@<public-ip>`
  - Accept host prompt
- **Windows (PuTTY):**
  - Host: `ec2-user@<public-ip>` Port 22
  - Auth: load .ppk file
  - Open session

**Expected Prompt:** `[ec2-user@ip-10-0-1-x ~]`

[SCREENSHOT: Successful SSH session]

## CLI Alternative (Copy-Paste Ready)
```bash
REGION=us-east-1

# Get your public IP
ADMIN_IP=$(curl -s https://checkip.amazonaws.com)/32
echo "Your IP: $ADMIN_IP"

# Create key pair
KEY_NAME=practice-key-q2
aws ec2 create-key-pair --key-name $KEY_NAME --region $REGION \
  --query 'KeyMaterial' --output text > ${KEY_NAME}.pem
chmod 400 ${KEY_NAME}.pem

echo "Key pair created: ${KEY_NAME}.pem"

# Get VPC ID from Q1
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=practice-vpc-q1" \
  --query 'Vpcs[0].VpcId' --output text --region $REGION)
echo "VPC ID: $VPC_ID"

# Create security group
SG_ID=$(aws ec2 create-security-group \
  --group-name practice-ssh-sg \
  --description "SSH from admin IP only" \
  --vpc-id $VPC_ID \
  --region $REGION \
  --query 'GroupId' --output text)
echo "Security Group ID: $SG_ID"

# Add SSH rule for your IP
aws ec2 authorize-security-group-ingress \
  --group-id $SG_ID \
  --protocol tcp \
  --port 22 \
  --cidr $ADMIN_IP \
  --region $REGION
echo "SSH rule added for $ADMIN_IP"

# Get latest Amazon Linux 2023 AMI
AL2023_AMI=$(aws ssm get-parameter \
  --name /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64 \
  --region $REGION \
  --query 'Parameter.Value' --output text)
echo "AMI ID: $AL2023_AMI"

# Get subnet ID from Q1
PUB_SUBNET=$(aws ec2 describe-subnets \
  --filters "Name=tag:Name,Values=practice-public-1a" \
  --query 'Subnets[0].SubnetId' --output text --region $REGION)
echo "Subnet ID: $PUB_SUBNET"

# Launch EC2 instance
INSTANCE_ID=$(aws ec2 run-instances \
  --image-id $AL2023_AMI \
  --instance-type t2.micro \
  --key-name $KEY_NAME \
  --security-group-ids $SG_ID \
  --subnet-id $PUB_SUBNET \
  --associate-public-ip-address \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=practice-ec2-q2}]' \
  --region $REGION \
  --query 'Instances[0].InstanceId' --output text)
echo "Instance ID: $INSTANCE_ID"

echo "Waiting for instance to be running..."
aws ec2 wait instance-running --instance-ids $INSTANCE_ID --region $REGION

echo "Waiting for status checks..."
aws ec2 wait instance-status-ok --instance-ids $INSTANCE_ID --region $REGION

# Get public IP
PUBLIC_IP=$(aws ec2 describe-instances \
  --instance-ids $INSTANCE_ID \
  --region $REGION \
  --query 'Reservations[0].Instances[0].PublicIpAddress' --output text)
echo "Public IP: $PUBLIC_IP"

echo "Connect with: ssh -i ${KEY_NAME}.pem ec2-user@${PUBLIC_IP}"
```

## Verification Checklist

1. **Key Pair Verification**
   - Console: EC2 → Key Pairs shows `practice-key-q2`
   - File permissions: `ls -l practice-key-q2.pem` → `-r--------`
   - [SCREENSHOT: Key pair list]

2. **Security Group Verification**
   - Console: EC2 → Security Groups → `practice-ssh-sg`
   - Inbound: SSH 22 from your IP/32 only; outbound allow all
   - [SCREENSHOT: Security group rules]

3. **Instance State Verification**
   - Console: EC2 → Instances → state Running, status 2/2 passed, Public IP present
   - [SCREENSHOT: Instance details]

4. **SSH Connection Test**
  - `ssh -i practice-key-q2.pem ec2-user@<public-ip>`
  - Expected prompt: `[ec2-user@ip-10-0-1-x ~]`
   - Run `whoami` → `ec2-user`
   - Run `hostname` → `ip-10-0-1-x`
   - [SCREENSHOT: SSH session with commands]

5. **Internet Connectivity Test**
   - Run `curl -s ifconfig.me` → returns instance public IP
   - Run `ping -c 3 8.8.8.8` → replies
   - Run `curl -I http://example.com` → HTTP/1.1 200 OK
   - [SCREENSHOT: Connectivity test outputs]

6. **Security Verification (Negative Test)**
   - Attempt SSH from different IP (hotspot/VPN) → expect timeout/refused
   - [SCREENSHOT: Connection failure from unauthorized IP]

## Troubleshooting Guide

- **Permission Denied (publickey)**
  - Fix: `chmod 400 practice-key-q2.pem`; ensure username `ec2-user`; ensure key matches instance

- **Connection Timeout**
  - Fix: Verify current IP via `curl https://checkip.amazonaws.com`; update SG rule; ensure public IP assigned; ensure route to IGW exists

- **Host Key Verification Failed**
  - Fix: `ssh-keygen -R <public-ip>` then reconnect and accept new key

- **Instance Has No Public IP**
  - Fix: Terminate; confirm subnet auto-assign public IP enabled; relaunch with "Enable" selected

- **Wrong AMI Username**
  - Fix: Amazon Linux uses `ec2-user`; Ubuntu uses `ubuntu`; use correct default user

- **Security Group Too Permissive (0.0.0.0/0)**
  - Fix: Edit inbound rule to your IP/32; avoid exam penalties

## Cleanup Instructions

**Console Cleanup:**
1. Terminate EC2 `practice-ec2-q2`
2. Delete security group `practice-ssh-sg`
3. Delete key pair `practice-key-q2` (optional if not reused); remove local .pem

**CLI Cleanup:**
```bash
REGION=us-east-1

# Terminate instance
aws ec2 terminate-instances --instance-ids $INSTANCE_ID --region $REGION
aws ec2 wait instance-terminated --instance-ids $INSTANCE_ID --region $REGION

# Delete security group
aws ec2 delete-security-group --group-id $SG_ID --region $REGION

# Delete key pair and local file
aws ec2 delete-key-pair --key-name $KEY_NAME --region $REGION
rm ${KEY_NAME}.pem
```

**Verification:** EC2 console shows no running instances for this lab

## Mark Mapping (Exam Scoring)

| Task | Marks | Criteria | Your Score |
|------|-------|----------|------------|
| Key pair creation | 2 | Key pair created and downloaded, correct permissions set | [ ] |
| Security group rule | 4 | SSH (22) allowed from admin IP only, not 0.0.0.0/0 | [ ] |
| EC2 launch | 4 | t2.micro, correct subnet, correct AMI | [ ] |
| Public IP assignment | 2 | Public IP assigned and visible | [ ] |
| SSH connection | 5 | Successful login with screenshot proof | [ ] |
| Internet connectivity test | 3 | curl/ping outputs showing internet access | [ ] |
| Security group verification | 3 | Console screenshot showing scoped rule | [ ] |
| Documentation | 2 | Steps and outputs recorded clearly | [ ] |
| **Total** | **25** | | **[ ]** |

## Key Takeaways
- Key pairs are region-specific and unrecoverable if lost
- Security groups are stateful; restrict SSH to IP/32 for least privilege
- Default username varies by AMI; Amazon Linux uses `ec2-user`
- Key file permissions must be 400 on Linux/macOS
- Public IP is required for direct internet SSH access

## Next Steps
- Complete Q7: EC2 User Data automation
- Complete Q5: ALB Basics for load balancing
- Review EC2 concepts in 03_ec2/overview.md

## Related Resources
- Main practice file: 10_indskills/state_level_practice.md (Q2)
- EC2 service guide: 03_ec2/
- Security Groups guide: 03_ec2/security_groups.md
- SSH guide: 03_ec2/keypairs_ssh.md
