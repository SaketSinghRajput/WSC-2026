# Q4: RDS MySQL Backup & Restore

## Lab Overview
- **Difficulty:** Beginner
- **Estimated Time:** 40-45 minutes
- **AWS Services:** RDS MySQL, VPC, Security Groups
- **Region:** us-east-1
- **Cost:** Free Tier (db.t3.micro 750 hours, 20GB storage)

## Prerequisites Check
- [ ] Completed Q1 (VPC) and Q2 (EC2 client)
- [ ] MySQL client available (on EC2 or local)
- [ ] AWS CLI configured (optional)
- [ ] Understanding of backups and snapshots

## Learning Objectives
- Create RDS MySQL instance with Free Tier settings
- Configure automated backups with retention
- Create manual snapshots
- Connect from EC2 using scoped security groups
- Differentiate automated vs manual backups

## Architecture Overview
```mermaid
flowchart LR
    EC2[EC2 (MySQL Client)] -->|3306| SG[Security Group]
    SG --> RDS[RDS MySQL]
    RDS --> Auto[Automated Backups (7 days)]
    RDS --> Snap[Manual Snapshot]
```

## Step-by-Step Console Instructions

### Step 1: Create Database Security Group
**Console Navigation:** EC2 → Security Groups → Create security group

**Detailed Steps:**
1. Name: `practice-rds-sg`
2. Description: "MySQL access from EC2 instances"
3. VPC: `practice-vpc-q1`
4. Inbound rule: MySQL/Aurora TCP 3306, Source: `practice-ssh-sg`
5. Outbound: default
6. Create security group

[SCREENSHOT: RDS security group with MySQL rule]

### Step 2: Create RDS Subnet Group
**Console Navigation:** RDS → Subnet groups → Create DB subnet group

**Detailed Steps:**
1. Name: `practice-rds-subnet-group`
2. Description: "Subnet group for practice RDS"
3. VPC: `practice-vpc-q1`
4. AZs: select us-east-1a and us-east-1b
5. Subnets: select `practice-public-1a` and `practice-public-1b`
6. Create

[SCREENSHOT: Subnet group created]

### Step 3: Create RDS MySQL Database
**Console Navigation:** RDS → Databases → Create database

**Detailed Steps:**
1. Creation method: Standard create
2. Engine: MySQL 8.0.x (Community)
3. Templates: Free tier
4. Settings:
   - DB instance id: `practice-rds-q4`
   - Master username: `admin`
   - Master password: `TempPass123!` (strong password) confirm same
5. Instance class: db.t3.micro (burstable)
6. Storage: gp2, 20 GiB, autoscaling disabled
7. Connectivity:
   - VPC: `practice-vpc-q1`
   - DB subnet group: `practice-rds-subnet-group`
   - Public access: No
   - VPC security group: existing → `practice-rds-sg`
   - AZ: No preference
8. Authentication: Password
9. Additional configuration:
   - Initial DB name: `practicedb`
   - Backup: enable automated backups, retention 7 days, copy tags to snapshots
   - Encryption: Enable (default)
   - Auto minor version upgrade: Yes
   - Deletion protection: Disable
10. Create database

[SCREENSHOT: Database creation in progress]

### Step 4: Wait for Database Availability
1. RDS → Databases → wait until status "Available"
2. Open `practice-rds-q4`
3. Note Endpoint and Port 3306

[SCREENSHOT: Database available with endpoint]

### Step 5: Install MySQL Client on EC2 (from Q2)
SSH to EC2 from Q2:
```bash
# Amazon Linux 2023
sudo dnf install -y mariadb105
mysql --version
```

[SCREENSHOT: MySQL client installed]

### Step 6: Connect to RDS from EC2
```bash
RDS_ENDPOINT="practice-rds-q4.c9akciq32.us-east-1.rds.amazonaws.com"
DB_USER="admin"
DB_PASS="TempPass123!"

mysql -h $RDS_ENDPOINT -u $DB_USER -p
# Enter password when prompted
```

[SCREENSHOT: MySQL connection successful]

### Step 7: Create Test Data
In MySQL prompt:
```sql
SHOW DATABASES;
USE practicedb;

CREATE TABLE students (
    id INT AUTO_INCREMENT PRIMARY KEY,
    name VARCHAR(100),
    score INT,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);

INSERT INTO students (name, score) VALUES 
    ('Alice', 95),
    ('Bob', 87),
    ('Charlie', 92);

SELECT * FROM students;
SELECT NOW();
EXIT;
```

[SCREENSHOT: Test data created]

### Step 8: Create Manual Snapshot
1. RDS → Databases → select `practice-rds-q4`
2. Actions → Take snapshot
3. Name: `practice-rds-snap-q4`
4. Create; go to Snapshots and wait for "Available"

[SCREENSHOT: Manual snapshot available]

### Step 9: Verify Automated Backups
1. RDS → Databases → `practice-rds-q4`
2. Maintenance & backups tab
3. Verify retention 7 days, latest restore time, backup window

[SCREENSHOT: Automated backups configuration]

## CLI Alternative (Copy-Paste Ready)
```bash
REGION=us-east-1

# IDs from Q1/Q2
VPC_ID=$(aws ec2 describe-vpcs --filters "Name=tag:Name,Values=practice-vpc-q1" \
  --query 'Vpcs[0].VpcId' --output text --region $REGION)
SUBNET_A=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=practice-public-1a" \
  --query 'Subnets[0].SubnetId' --output text --region $REGION)
SUBNET_B=$(aws ec2 describe-subnets --filters "Name=tag:Name,Values=practice-public-1b" \
  --query 'Subnets[0].SubnetId' --output text --region $REGION)
EC2_SG=$(aws ec2 describe-security-groups --filters "Name=group-name,Values=practice-ssh-sg" \
  --query 'SecurityGroups[0].GroupId' --output text --region $REGION)

# RDS security group
aws ec2 create-security-group \
  --group-name practice-rds-sg \
  --description "MySQL access from EC2" \
  --vpc-id $VPC_ID \
  --region $REGION \
  --query 'GroupId' --output text > /tmp/rds_sg_id
RDS_SG=$(cat /tmp/rds_sg_id)
echo "RDS Security Group: $RDS_SG"

aws ec2 authorize-security-group-ingress \
  --group-id $RDS_SG \
  --protocol tcp \
  --port 3306 \
  --source-group $EC2_SG \
  --region $REGION

# Subnet group
aws rds create-db-subnet-group \
  --db-subnet-group-name practice-rds-subnet-group \
  --db-subnet-group-description "Subnet group for practice RDS" \
  --subnet-ids $SUBNET_A $SUBNET_B \
  --region $REGION

# RDS instance
aws rds create-db-instance \
  --db-instance-identifier practice-rds-q4 \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password TempPass123! \
  --allocated-storage 20 \
  --storage-type gp2 \
  --db-subnet-group-name practice-rds-subnet-group \
  --vpc-security-group-ids $RDS_SG \
  --backup-retention-period 7 \
  --no-publicly-accessible \
  --db-name practicedb \
  --region $REGION

echo "Creating RDS instance..."
aws rds wait db-instance-available --db-instance-identifier practice-rds-q4 --region $REGION

RDS_ENDPOINT=$(aws rds describe-db-instances \
  --db-instance-identifier practice-rds-q4 \
  --region $REGION \
  --query 'DBInstances[0].Endpoint.Address' --output text)
echo "RDS Endpoint: $RDS_ENDPOINT"

# Manual snapshot
aws rds create-db-snapshot \
  --db-instance-identifier practice-rds-q4 \
  --db-snapshot-identifier practice-rds-snap-q4 \
  --region $REGION

echo "Creating snapshot..."
aws rds wait db-snapshot-available \
  --db-snapshot-identifier practice-rds-snap-q4 \
  --region $REGION
echo "Snapshot ready"
```

## Verification Checklist

1. **RDS Instance**
   - Console: RDS → Databases → status Available, class db.t3.micro, engine MySQL 8.0.x, storage 20 GiB
   - [SCREENSHOT: Database details page]

2. **Connectivity Settings**
   - RDS → Connectivity & security: endpoint present, VPC `practice-vpc-q1`, SG `practice-rds-sg`, Public access No
   - [SCREENSHOT: Connectivity settings]

3. **Security Group**
   - EC2 → Security Groups → `practice-rds-sg` inbound MySQL 3306 from `practice-ssh-sg`
   - [SCREENSHOT: Security group rules]

4. **Database Connection Test**
   - From EC2: `mysql -h <endpoint> -u admin -p`
   - `SHOW DATABASES;` → includes practicedb
   - `SHOW TABLES;` → students table
   - `SELECT * FROM students;` → 3 rows
   - [SCREENSHOT: MySQL queries]

5. **Automated Backups**
   - Database → Maintenance & backups: retention 7 days, latest restore time set
   - [SCREENSHOT: Automated backups section]

6. **Manual Snapshot**
   - RDS → Snapshots: `practice-rds-snap-q4` status Available, type Manual
   - [SCREENSHOT: Snapshot list]

7. **Backup Window**
   - Maintenance & backups: earliest/latest restore time span visible (after backups accumulate)
   - [SCREENSHOT: Restore time window]

## Troubleshooting Guide

- **Cannot Connect from EC2**
  - Ensure RDS SG allows 3306 from `practice-ssh-sg`; endpoint correct; DB status Available; test `telnet <endpoint> 3306`

- **Access Denied for User 'admin'**
  - Password wrong; reset via Modify DB instance → new master password → apply immediately

- **Insufficient Capacity**
  - Retry with AZ preference "No preference" or different AZ; delete failed instance first

- **Backup Retention Not 7 Days**
  - Modify DB → Backup retention period 7 → apply immediately; wait for modification

- **Manual Snapshot Stuck**
  - Wait for DB to be Available; retry snapshot; ensure storage not full

- **MySQL Client Missing**
  - Amazon Linux 2023: `sudo dnf install -y mariadb105`; Amazon Linux 2: `sudo yum install -y mysql`; Ubuntu: `sudo apt-get install -y mysql-client`

## Cleanup Instructions

**Console Cleanup:**
1. Delete manual snapshot `practice-rds-snap-q4`
2. Delete DB `practice-rds-q4` (no final snapshot, delete automated backups)
3. Delete DB subnet group `practice-rds-subnet-group`
4. Delete security group `practice-rds-sg`

**CLI Cleanup:**
```bash
REGION=us-east-1

aws rds delete-db-snapshot \
  --db-snapshot-identifier practice-rds-snap-q4 \
  --region $REGION

aws rds delete-db-instance \
  --db-instance-identifier practice-rds-q4 \
  --skip-final-snapshot \
  --delete-automated-backups \
  --region $REGION

aws rds wait db-instance-deleted \
  --db-instance-identifier practice-rds-q4 \
  --region $REGION

aws rds delete-db-subnet-group \
  --db-subnet-group-name practice-rds-subnet-group \
  --region $REGION

aws ec2 delete-security-group \
  --group-id $RDS_SG \
  --region $REGION
```

**Verification:** RDS console shows no databases or snapshots for this lab

## Mark Mapping (Exam Scoring)

| Task | Marks | Criteria | Your Score |
|------|-------|----------|------------|
| DB instance creation | 4 | MySQL 8.0, db.t3.micro, correct VPC/subnet group | [ ] |
| Engine configuration | 2 | Correct version and settings | [ ] |
| Automated backups | 4 | Enabled with 7-day retention period | [ ] |
| Retention period | 2 | Correctly set to 7 days | [ ] |
| Manual snapshot | 4 | Created and available | [ ] |
| Backup verification | 4 | Console shows automated backups and snapshot | [ ] |
| Security settings | 3 | Public access disabled, SG scoped to EC2 SG | [ ] |
| Documentation | 2 | Screenshots and evidence captured | [ ] |
| **Total** | **25** | | **[ ]** |

## Key Takeaways
- Automated backups enable PITR within retention window
- Manual snapshots persist until deleted; not governed by retention
- Backup retention: 0 disables; 1-35 days enables
- Private subnets are best practice; public used here for simplicity
- SGs control DB access; keep scope tight
- Free Tier covers db.t3.micro 750 hours and 20GB storage; backups share storage
- Always test connectivity before declaring success

## Next Steps
- Complete Q6: ElastiCache Redis for performance
- Explore Multi-AZ and Read Replicas in State++ Q3
- Review RDS concepts in 07_rds/overview.md

## Related Resources
- Main practice file: 10_indskills/state_level_practice.md (Q4)
- RDS service guide: 07_rds/
- Backup guide: 07_rds/security_backup.md
- Multi-AZ guide: 07_rds/multi_az_read_replica.md
