# Complete EC2 Web Server Lab - Apache Deployment

## Lab Overview

**Objective**: Deploy a fully functional Apache web server on EC2 with custom HTML, accessible from the internet via HTTP.

**What You'll Build**:
- t2.micro EC2 instance (Free Tier)
- Amazon Linux 2023 operating system
- Apache HTTP Server (httpd)
- Custom security group with SSH and HTTP access
- Custom HTML page displaying your name
- SSH access for administration

**Time Estimate**: 45-60 minutes  
**Cost**: Free Tier eligible ($0 expected)  
**Prerequisites**: AWS account, SSH client, text editor

## Problem Statement

> **WorldSkills Competition Task**:  
> Deploy a web server accessible from the internet displaying a custom HTML page.
>
> **Requirements**:
> 1. EC2 instance type: t2.micro (Free Tier)
> 2. Operating system: Amazon Linux 2023
> 3. Web server: Apache HTTP Server
> 4. HTTP access on port 80 from anywhere
> 5. SSH access on port 22 from your IP only
> 6. Custom index.html displaying: "Welcome! Server managed by [Your Name]"
> 7. Web server must start automatically on boot
> 8. Verify HTTP access via browser and curl
> 9. Document public IP address
> 10. Complete deployment in under 30 minutes

## Architecture Diagram

```mermaid
graph TB
    Internet[Internet Users] -->|HTTP Port 80| IGW[Internet Gateway]
    Admin[Administrator] -->|SSH Port 22| IGW
    
    IGW --> RT[Route Table<br/>0.0.0.0/0 → IGW]
    RT --> Subnet[Public Subnet<br/>10.0.1.0/24]
    
    subgraph "VPC 10.0.0.0/16"
        Subnet --> SG[Security Group]
        SG -->|Allow 80 from 0.0.0.0/0| EC2[EC2 Instance<br/>t2.micro]
        SG -->|Allow 22 from Your IP| EC2
        EC2 --> EBS[EBS Volume<br/>8 GB gp3]
        EC2 --> Apache[Apache httpd]
        Apache --> HTML[/var/www/html/index.html]
    end
```

## Prerequisites Checklist

Before starting:

- [ ] AWS account created and verified
- [ ] AWS CLI installed and configured (optional but recommended)
- [ ] SSH client available (Linux/Mac: built-in, Windows: OpenSSH or PuTTY)
- [ ] Text editor for editing code
- [ ] Know your public IP: `curl ifconfig.me`
- [ ] Browser for testing

## Step 1: Create Key Pair (5 minutes)

Key pair required for SSH access to instance.

### AWS Console Steps

1. **Navigate to Key Pairs**:
   - EC2 Dashboard → Network & Security → Key Pairs
   - Click "Create key pair"

2. **Configure Key Pair**:
   - **Name**: WebServerKey
   - **Key pair type**: RSA
   - **Private key file format**: .pem (OpenSSH)
   - Click "Create key pair"

3. **Download Private Key**:
   - Browser downloads `WebServerKey.pem`
   - Move to secure location: `~/.ssh/` (Linux/Mac)

4. **Set Permissions** (Linux/Mac):
```bash
chmod 400 ~/.ssh/WebServerKey.pem
```

**Windows** (PowerShell):
```powershell
icacls WebServerKey.pem /inheritance:r
icacls WebServerKey.pem /grant:r "$env:USERNAME:R"
```

### AWS CLI Alternative

```bash
aws ec2 create-key-pair \
    --key-name WebServerKey \
    --query 'KeyMaterial' \
    --output text > ~/.ssh/WebServerKey.pem

chmod 400 ~/.ssh/WebServerKey.pem
```

## Step 2: Create Security Group (10 minutes)

Security group controls firewall rules for instance.

### AWS Console Steps

1. **Navigate to Security Groups**:
   - EC2 Dashboard → Network & Security → Security Groups
   - Click "Create security group"

2. **Basic Details**:
   - **Security group name**: WebServerSG
   - **Description**: Allow HTTP from anywhere and SSH from my IP
   - **VPC**: Default VPC (or select your VPC)

3. **Inbound Rules**:

**Rule 1: SSH**
   - Click "Add rule"
   - **Type**: SSH (auto-fills TCP port 22)
   - **Source type**: My IP
   - **Description**: SSH from my workstation

**Rule 2: HTTP**
   - Click "Add rule"
   - **Type**: HTTP (auto-fills TCP port 80)
   - **Source type**: Anywhere-IPv4 (0.0.0.0/0)
   - **Description**: HTTP from internet

4. **Outbound Rules**:
   - Leave default: All traffic to 0.0.0.0/0

5. **Create Security Group**:
   - Click "Create security group"
   - Note security group ID: sg-0123456789abcdef

### AWS CLI Alternative

```bash
# Get your public IP
MY_IP=$(curl -s ifconfig.me)

# Create security group
SG_ID=$(aws ec2 create-security-group \
    --group-name WebServerSG \
    --description "Allow HTTP and SSH" \
    --query 'GroupId' \
    --output text)

echo "Security Group ID: $SG_ID"

# Add SSH rule
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 22 \
    --cidr $MY_IP/32

# Add HTTP rule
aws ec2 authorize-security-group-ingress \
    --group-id $SG_ID \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```

## Step 3: Launch EC2 Instance (15 minutes)

### AWS Console Steps

1. **Navigate to Instances**:
   - EC2 Dashboard → Instances → Instances
   - Click "Launch instances"

2. **Name and Tags**:
   - **Name**: WebServer
   - Add tag: Environment = Production

3. **Application and OS Images (AMI)**:
   - **Quick Start**: Amazon Linux
   - **Amazon Machine Image (AMI)**: Amazon Linux 2023 AMI
   - **Architecture**: 64-bit (x86)
   - Verify "Free tier eligible" badge

4. **Instance Type**:
   - **Instance type**: t2.micro
   - Verify "Free tier eligible" label
   - vCPU: 1, Memory: 1 GiB

5. **Key Pair (login)**:
   - **Key pair name**: Select WebServerKey
   - If forgotten key, create new one

6. **Network Settings**:
   - **VPC**: Default VPC
   - **Subnet**: No preference (auto-assign)
   - **Auto-assign public IP**: Enable
   - **Firewall (security groups)**: Select existing security group
   - **Select**: WebServerSG

7. **Configure Storage**:
   - **Root volume**: 8 GiB gp3
   - Free Tier: 30 GB per month
   - Leave default settings

8. **Advanced Details** (scroll down):
   - **User data** (paste script below):

```bash
#!/bin/bash
# Update system packages
yum update -y

# Install Apache HTTP Server
yum install httpd -y

# Fetch instance metadata (IMDSv2)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/instance-id)
AVAILABILITY_ZONE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/placement/availability-zone)

# Create custom index.html with server-side metadata injection
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome to My Web Server</title>
    <style>
        body {
            font-family: Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            display: flex;
            justify-content: center;
            align-items: center;
            height: 100vh;
            margin: 0;
        }
        .container {
            text-align: center;
            background: rgba(255, 255, 255, 0.1);
            padding: 50px;
            border-radius: 15px;
            box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
        }
        h1 { font-size: 3em; margin-bottom: 20px; }
        p { font-size: 1.2em; }
        .info { margin-top: 30px; font-size: 0.9em; opacity: 0.8; }
    </style>
</head>
<body>
    <div class="container">
        <h1>Welcome!</h1>
        <p>Server managed by <strong>Your Name</strong></p>
        <div class="info">
            <p>Instance ID: <span>${INSTANCE_ID}</span></p>
            <p>Availability Zone: <span>${AVAILABILITY_ZONE}</span></p>
            <p>Server: Apache HTTP Server on Amazon Linux 2023</p>
        </div>
    </div>
</body>
</html>
EOF

# Replace "Your Name" with actual name
sed -i 's/Your Name/John Doe/g' /var/www/html/index.html

# Start Apache and enable on boot
systemctl start httpd
systemctl enable httpd

# Create log file indicating completion
echo "User data script completed at $(date)" > /var/log/userdata.log
```

**Important**: Replace "John Doe" with your actual name in the sed command.

9. **Summary**:
   - Review configuration
   - Verify Free Tier selections
   - **Number of instances**: 1

10. **Launch Instance**:
    - Click "Launch instance"
    - Wait for "Successfully initiated launch of instance i-..."
    - Click "View all instances"

### AWS CLI Alternative

```bash
# Get latest Amazon Linux 2023 AMI
AMI_ID=$(aws ec2 describe-images \
    --owners amazon \
    --filters "Name=name,Values=al2023-ami-2023.*-x86_64" \
    --query 'Images | sort_by(@, &CreationDate) | [-1].ImageId' \
    --output text)

# Encode user data script
USER_DATA=$(cat <<'EOF' | base64 -w 0
#!/bin/bash
yum update -y
yum install httpd -y
cat > /var/www/html/index.html <<'HTML'
<html><head><title>Web Server</title></head>
<body><h1>Welcome! Server managed by John Doe</h1></body></html>
HTML
systemctl start httpd
systemctl enable httpd
EOF
)

# Launch instance
aws ec2 run-instances \
    --image-id $AMI_ID \
    --instance-type t2.micro \
    --key-name WebServerKey \
    --security-group-ids $SG_ID \
    --user-data $USER_DATA \
    --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=WebServer}]'
```

## Step 4: Verify Instance Running (5 minutes)

### AWS Console

1. **View Instances**:
   - EC2 Dashboard → Instances → Instances
   - Find "WebServer" instance

2. **Check Status**:
   - **Instance state**: Should show "Running" (green)
   - **Status checks**: Wait for "2/2 checks passed" (takes ~2 minutes)

3. **Note Public IP**:
   - Select instance
   - **Details tab** → Public IPv4 address
   - Example: 54.123.45.67
   - **Save this IP** for testing

### AWS CLI

```bash
# Get instance details
aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=WebServer" \
    --query 'Reservations[0].Instances[0].[InstanceId,State.Name,PublicIpAddress]' \
    --output table

# Wait for running state
aws ec2 wait instance-running \
    --instance-ids i-0123456789abcdef

# Wait for status checks
aws ec2 wait instance-status-ok \
    --instance-ids i-0123456789abcdef
```

## Step 5: Test SSH Access (5 minutes)

Verify you can connect to instance via SSH.

### Linux/Mac

```bash
ssh -i ~/.ssh/WebServerKey.pem ec2-user@54.123.45.67
```

**First connection prompts**:
```
The authenticity of host '54.123.45.67 (54.123.45.67)' can't be established.
ECDSA key fingerprint is SHA256:...
Are you sure you want to continue connecting (yes/no)?
```

Type: `yes`

**Successful connection**:
```
   ,     #_
   ~\_  ####_        Amazon Linux 2023
  ~~  \_#####\
  ~~     \###|
  ~~       \#/ ___   https://aws.amazon.com/linux/amazon-linux-2023
   ~~       V~' '->
    ~~~         /
      ~~._.   _/
         _/ _/
       _/m/'

[ec2-user@ip-10-0-1-123 ~]$
```

### Windows (OpenSSH)

```powershell
ssh -i C:\Users\YourName\.ssh\WebServerKey.pem ec2-user@54.123.45.67
```

### Windows (PuTTY)

1. Convert .pem to .ppk using PuTTYgen
2. Open PuTTY
3. Host Name: ec2-user@54.123.45.67
4. Connection → SSH → Auth → Credentials → Private key file: Browse to .ppk
5. Click "Open"

### Verify Apache Running

Once connected via SSH:

```bash
# Check Apache status
sudo systemctl status httpd

# Expected output:
# ● httpd.service - The Apache HTTP Server
#    Loaded: loaded (/usr/lib/systemd/system/httpd.service; enabled)
#    Active: active (running) since ...

# Check if listening on port 80
sudo netstat -tlnp | grep :80

# Expected output:
# tcp6  0  0 :::80  :::*  LISTEN  1234/httpd

# Exit SSH
exit
```

## Step 6: Test HTTP Access (10 minutes)

### Browser Test

1. Open web browser
2. Navigate to: `http://54.123.45.67` (use your public IP)
3. **Expected**: Custom HTML page with purple gradient background
4. **Verify**:
   - Title: "Welcome to My Web Server"
   - Your name displayed
   - Instance ID shown
   - Availability Zone shown

### curl Test

```bash
curl http://54.123.45.67
```

**Expected output**: Full HTML source

**Check HTTP status**:
```bash
curl -I http://54.123.45.67
```

**Expected**:
```
HTTP/1.1 200 OK
Server: Apache/2.4.58 ()
Content-Type: text/html; charset=UTF-8
```

### Verify Auto-Start on Boot

Test that web server starts automatically after reboot:

```bash
# Get instance ID
INSTANCE_ID=$(aws ec2 describe-instances \
    --filters "Name=tag:Name,Values=WebServer" \
    --query 'Reservations[0].Instances[0].InstanceId' \
    --output text)

# Reboot instance
aws ec2 reboot-instances --instance-ids $INSTANCE_ID

# Wait 2 minutes
sleep 120

# Test HTTP again
curl -I http://54.123.45.67
```

**Expected**: Still returns 200 OK (Apache started automatically)

## Step 7: Customize Web Page (Optional, 5 minutes)

SSH into instance and modify the HTML:

```bash
ssh -i ~/.ssh/WebServerKey.pem ec2-user@54.123.45.67

# Edit index.html
sudo vi /var/www/html/index.html

# Or replace entirely
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/instance-id)
AVAILABILITY_ZONE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" http://169.254.169.254/latest/meta-data/placement/availability-zone)

sudo cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html>
<head><title>Custom Page</title></head>
<body>
<h1>My Custom Web Server</h1>
<p>Deployed for WorldSkills 2026 Competition</p>
<p>Instance ID: ${INSTANCE_ID}</p>
<p>Availability Zone: ${AVAILABILITY_ZONE}</p>
</body>
</html>
EOF

# Verify changes
curl http://localhost
```

Refresh browser to see changes.

## Installation Commands Reference

### Apache Installation (Amazon Linux 2023)

```bash
# Update package repository
sudo yum update -y

# Install Apache HTTP Server
sudo yum install httpd -y

# Start Apache
sudo systemctl start httpd

# Check status
sudo systemctl status httpd

# Enable auto-start on boot
sudo systemctl enable httpd

# Stop Apache (if needed)
sudo systemctl stop httpd

# Restart Apache
sudo systemctl restart httpd
```

### Nginx Alternative

If using Nginx instead of Apache:

```bash
# Install Nginx
sudo yum install nginx -y

# Start Nginx
sudo systemctl start nginx

# Enable on boot
sudo systemctl enable nginx

# Document root
# /usr/share/nginx/html/index.html
```

## Complete User Data Script

For automated deployment without SSH:

```bash
#!/bin/bash
# Complete web server deployment script

# Update system
yum update -y

# Install Apache
yum install httpd -y

# Get instance metadata (IMDSv2)
TOKEN=$(curl -s -X PUT "http://169.254.169.254/latest/api/token" \
    -H "X-aws-ec2-metadata-token-ttl-seconds: 21600")
INSTANCE_ID=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/instance-id)
AVAILABILITY_ZONE=$(curl -s -H "X-aws-ec2-metadata-token: $TOKEN" \
    http://169.254.169.254/latest/meta-data/placement/availability-zone)

# Create custom index.html with dynamic data
cat > /var/www/html/index.html <<EOF
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>WorldSkills Web Server</title>
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body {
            font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            justify-content: center;
            align-items: center;
            color: white;
        }
        .container {
            background: rgba(255, 255, 255, 0.1);
            backdrop-filter: blur(10px);
            padding: 60px;
            border-radius: 20px;
            box-shadow: 0 8px 32px 0 rgba(31, 38, 135, 0.37);
            text-align: center;
            max-width: 600px;
        }
        h1 { font-size: 3em; margin-bottom: 20px; text-shadow: 2px 2px 4px rgba(0,0,0,0.3); }
        .tagline { font-size: 1.3em; margin-bottom: 40px; opacity: 0.9; }
        .info-box {
            background: rgba(255, 255, 255, 0.2);
            padding: 20px;
            border-radius: 10px;
            margin-top: 30px;
        }
        .info-box p { margin: 10px 0; font-size: 1.1em; }
        .info-box strong { color: #ffd700; }
        .footer { margin-top: 40px; font-size: 0.9em; opacity: 0.7; }
    </style>
</head>
<body>
    <div class="container">
        <h1>🚀 Welcome!</h1>
        <p class="tagline">Server managed by <strong>Your Name Here</strong></p>
        
        <div class="info-box">
            <p><strong>Instance ID:</strong> ${INSTANCE_ID}</p>
            <p><strong>Availability Zone:</strong> ${AVAILABILITY_ZONE}</p>
            <p><strong>Server:</strong> Apache HTTP Server</p>
            <p><strong>OS:</strong> Amazon Linux 2023</p>
            <p><strong>Status:</strong> <span style="color: #00ff00;">●</span> Running</p>
        </div>
        
        <div class="footer">
            <p>WorldSkills 2026 Competition Project</p>
            <p>Deployed: $(date)</p>
        </div>
    </div>
</body>
</html>
EOF

# Start and enable Apache
systemctl start httpd
systemctl enable httpd

# Create log
echo "Web server deployment completed at $(date)" > /var/log/deployment.log
```

## Verification Checklist

Complete deployment verification:

- [ ] Key pair created and downloaded
- [ ] Private key permissions set (400)
- [ ] Security group created with SSH and HTTP rules
- [ ] EC2 instance launched with t2.micro
- [ ] Instance state: Running
- [ ] Status checks: 2/2 passed
- [ ] Public IP address noted
- [ ] SSH connection successful
- [ ] Apache service running (`systemctl status httpd`)
- [ ] Apache listening on port 80 (`netstat -tlnp | grep :80`)
- [ ] HTTP accessible via browser (http://public-ip)
- [ ] Custom HTML page displays
- [ ] Your name shown on page
- [ ] Instance ID displayed
- [ ] Apache starts on boot (verified via reboot test)
- [ ] curl returns 200 OK

## Common Mistakes and Solutions

### Mistake 1: Cannot SSH - Connection Timeout

**Cause**: Security group doesn't allow port 22 from your IP

**Solution**:
```bash
# Add SSH rule
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxx \
    --protocol tcp \
    --port 22 \
    --cidr YOUR_IP/32
```

### Mistake 2: Cannot Access Website

**Cause 1**: Security group doesn't allow port 80

**Solution**:
```bash
aws ec2 authorize-security-group-ingress \
    --group-id sg-xxxxx \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```

**Cause 2**: Apache not running

**Solution**:
```bash
ssh -i key.pem ec2-user@ip
sudo systemctl start httpd
sudo systemctl enable httpd
```

### Mistake 3: User Data Script Didn't Run

**Cause**: Script had errors or didn't execute

**Check logs**:
```bash
ssh -i key.pem ec2-user@ip
sudo cat /var/log/cloud-init-output.log
```

**Re-run manually**:
```bash
sudo yum install httpd -y
sudo systemctl start httpd
sudo systemctl enable httpd
```

### Mistake 4: Wrong Username for SSH

**Cause**: Used wrong default username

**Amazon Linux**: `ec2-user`  
**Ubuntu**: `ubuntu`  
**Debian**: `admin`

### Mistake 5: Instance Has No Public IP

**Cause**: Auto-assign public IP not enabled

**Solution**: Allocate Elastic IP and associate with instance

## Troubleshooting Guide

### Apache Not Starting

```bash
# Check service status
sudo systemctl status httpd

# View error logs
sudo cat /var/log/httpd/error_log

# Check configuration
sudo httpd -t

# Common fixes
sudo yum reinstall httpd -y
sudo systemctl start httpd
```

### Page Not Loading

```bash
# Test from instance itself
curl http://localhost

# If works locally but not externally:
# - Check security group (port 80)
# - Verify public IP correct
# - Check Network ACL
```

### Permission Denied for Key

```bash
# Fix permissions
chmod 400 WebServerKey.pem

# Verify
ls -l WebServerKey.pem
# Should show: -r--------
```

## Cleanup Steps

After completing lab, clean up to avoid charges:

### Option 1: Stop Instance (Preserve Data)

```bash
aws ec2 stop-instances --instance-ids i-0123456789abcdef
```

**Charges**: EBS storage only (~$0.80/month for 8 GB)

### Option 2: Terminate Instance (Delete Everything)

**Console**:
1. EC2 Dashboard → Instances
2. Select WebServer instance
3. Instance state → Terminate instance
4. Confirm

**CLI**:
```bash
aws ec2 terminate-instances --instance-ids i-0123456789abcdef
```

### Delete Security Group

```bash
aws ec2 delete-security-group --group-id sg-0123456789abcdef
```

### Delete Key Pair

```bash
aws ec2 delete-key-pair --key-name WebServerKey
rm ~/.ssh/WebServerKey.pem
```

## Expected Output Screenshots

**Placeholder for screenshots** (add your own):

1. **AWS Console - Instance Running**:
   - Instance state: Running
   - Status checks: 2/2 passed

2. **SSH Session**:
   - Successful login as ec2-user
   - Apache status showing active

3. **Browser**:
   - Custom HTML page with purple gradient
   - Your name displayed
   - Instance metadata shown

4. **curl Output**:
   - HTTP/1.1 200 OK
   - HTML content

## Time Breakdown

| Step | Task | Time (minutes) |
|------|------|----------------|
| 1 | Create key pair | 5 |
| 2 | Create security group | 10 |
| 3 | Launch EC2 instance | 15 |
| 4 | Verify running | 5 |
| 5 | Test SSH | 5 |
| 6 | Test HTTP | 10 |
| 7 | Customize (optional) | 5 |
| **Total** | | **55 minutes** |

## Bonus: Auto Scaling Version

Deploy same web server with Auto Scaling:

1. Create Launch Template with user data script
2. Create Auto Scaling Group (min=1, desired=2, max=3)
3. Create Application Load Balancer
4. Attach ASG to ALB target group
5. Access via ALB DNS name

See [autoscaling_basics.md](autoscaling_basics.md) for details.

## Summary

You've successfully:
- ✅ Created EC2 key pair for SSH access
- ✅ Configured security group with proper firewall rules
- ✅ Launched t2.micro instance (Free Tier)
- ✅ Installed and configured Apache HTTP Server
- ✅ Deployed custom HTML with dynamic metadata
- ✅ Verified SSH and HTTP access
- ✅ Ensured auto-start on boot

**Next Steps**:
- [cost_optimization.md](cost_optimization.md): Minimize costs
- [autoscaling_basics.md](autoscaling_basics.md): Add Auto Scaling
- Deploy application (Node.js, Python, PHP)
