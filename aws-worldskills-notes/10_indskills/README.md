# IndiaSkills State-Level Cloud Computing Exam Preparation

## Introduction and Scope

This module is designed specifically for **IndiaSkills State-Level and State++ Level** cloud computing competitions. It does NOT cover National or WorldSkills International-level complexity.

**Reference Baseline:** Sikkim 2025 State-Level Examination Paper  
**Time Constraint:** 2 hours for complete exam  
**Standard Region:** `us-east-1` (all implementations)  
**Focus:** Practical implementation with clear verification steps

This module prepares candidates to handle real exam questions under time pressure, focusing on services commonly tested at the state level with explicit marking criteria in mind.

---

## Level Differentiation

Understanding the difference between State and State++ levels is critical for focused preparation:

| Criteria | State Level | State++ Level |
|----------|-------------|---------------|
| **Complexity** | Single VPC, straightforward networking | Multi-subnet VPC (public + private), NAT Gateway |
| **VPC Setup** | One public subnet, basic routing | Public and private subnets across AZs, route table management |
| **Networking** | Internet Gateway only, direct EC2 internet access | NAT Gateway for private subnet egress, ALB in public subnets |
| **Services** | EC2, S3, RDS, basic ElastiCache, single-instance patterns | ALB with 2+ instances, path-based routing, advanced caching patterns |
| **Troubleshooting** | Direct verification (ping, curl, browser) | Security group chaining, route table debugging, multi-layer validation |
| **Time Pressure** | 2 hours (manageable pace) | 2 hours (requires efficiency and parallel tasking) |

---

## Topics Coverage

### Core Topics for State-Level Exams

- **VPC & CIDR**: Custom VPC creation with appropriate CIDR block planning (e.g., 10.0.0.0/16)
- **Subnets**: Public vs Private subnet configuration, CIDR subdivision, AZ distribution
- **Gateways**: Internet Gateway (IGW) for public internet access vs NAT Gateway for private subnet egress
- **Routing**: Route tables, subnet associations, default routes (0.0.0.0/0)
- **Security**: Security Groups (SG) configuration, inbound/outbound rules, source-based access control
- **Compute**: EC2 instance deployment, key pairs, SSH connectivity, user data scripts
- **Caching**: ElastiCache Redis for database performance optimization, cache hit/miss patterns
- **Database**: RDS backup strategies (snapshots vs automated backups), point-in-time recovery (PITR)
- **Load Balancing**: Application Load Balancer (ALB) for traffic distribution across multiple instances
- **Path Routing**: Path-based routing patterns (`/api/*`, `/static/*`) on ALB
- **Verification**: DNS validation, curl testing, redis-cli connectivity checks, screenshot documentation
- **Cost**: Free Tier optimization (t2.micro, db.t2.micro, cache.t2.micro), budget awareness

---

## Module Structure

This module is organized into three parts for progressive skill development:

### Part 1: Solved Past Questions (3 files)
- **Q1: Custom VPC Implementation** - Complete VPC setup with subnets, routing, and EC2 connectivity
- **Q2: Database Performance** - ElastiCache Redis integration with RDS for query optimization
- **Q3: Traffic Distribution** - ALB with path-based routing to multiple backend instances

These are fully solved questions from previous exams (Sikkim 2025 pattern) with detailed explanations, marking schemes, and evaluator expectations.

### Part 2: State-Level Practice (8 questions with full solutions)
Focused on single-subnet VPCs, basic networking, and individual service implementations. Ideal for building confidence before attempting complex scenarios.

### Part 3: State++ Practice (7 questions with full solutions)
Advanced scenarios requiring multi-subnet architectures, NAT Gateways, ALB configurations, and integrated service deployments.

---

## What Each Solution Includes

Every practice question follows this structured format:

1. **Scenario Understanding**: Problem statement breakdown and key requirements
2. **Architecture Explanation**: Marks-oriented diagram and component justification
3. **Step-by-Step Implementation**: Console instructions AND CLI commands for each task
4. **Verification/Proof**: Exact commands and expected outputs for evaluator validation
5. **Common Mistakes**: Pitfalls that cost marks (e.g., wrong security group rules)
6. **Mark Mapping**: How marks are distributed across functional, verification, and documentation tasks

---

## Exam Tips and Best Practices

### Time Management
- **Allocate 40 minutes per major question** (3 questions in 2 hours)
- Spend first 5 minutes reading all questions and identifying easier sections
- Complete verification steps immediately after implementation (don't defer)
- Reserve 10 minutes at end for screenshots and documentation

### Region Consistency
- **Always use `us-east-1`** to match exam environment and evaluator expectations
- Verify region selection in Console (top-right dropdown) before starting ANY task
- CLI commands should include `--region us-east-1` explicitly

### Verification First
- Test connectivity after EACH major step (don't wait until end)
- Use curl, ping, redis-cli, mysql client for validation
- Screenshot successful outputs immediately (browser dev tools, terminal)

### Documentation
- Use clear, consistent naming: `exam-vpc`, `exam-web-sg`, `exam-db-instance`
- Add Name tags to ALL resources (evaluators appreciate organized consoles)
- Keep a notepad with resource IDs, endpoints, and DNS names

### Common Pitfalls
- **Security Groups**: Forgetting to allow return traffic (stateful but source-specific)
- **Route Tables**: Not associating subnets with correct route table
- **IAM Permissions**: Missing role attachments for EC2 to access S3/RDS
- **CIDR Conflicts**: Overlapping subnet ranges causing routing failures
- **Health Checks**: ALB targets failing due to wrong path or port configuration

### Free Tier Awareness
- **EC2**: Use `t2.micro` (750 hours/month Free Tier)
- **RDS**: Use `db.t2.micro` or `db.t3.micro` (750 hours/month)
- **ElastiCache**: Use `cache.t2.micro` (750 hours/month for 12 months)
- **ALB**: Not Free Tier eligible (~$16/month, but often exam-provided)
- Stop instances after exam/practice to avoid charges

### Naming Conventions
Use structured names for evaluator clarity:
```
VPC: indskills-exam-vpc
Subnets: indskills-public-1a, indskills-private-1b
Security Groups: indskills-web-sg, indskills-db-sg
Instances: indskills-web-server-01
RDS: indskills-mysql-db
ElastiCache: indskills-redis-cache
ALB: indskills-web-alb
```

---

## Services NOT in Scope

The following services are beyond State-Level and State++ complexity. **Do not study these for state exams:**

- EKS (Elastic Kubernetes Service)
- Transit Gateway
- AWS Network Firewall
- Karpenter (Kubernetes autoscaling)
- CloudWatch Insights (basic CloudWatch only)
- Multi-region architectures
- VPC peering or VPN connections
- Advanced IAM policies (basic roles/policies only)
- Lambda@Edge
- AWS Organizations
- Step Functions
- AWS Glue or data analytics services

**State-Level Scope:** EC2, VPC, S3, RDS, ElastiCache, ALB, basic IAM roles, CloudWatch basic metrics only.

---

## Evaluation Criteria

Evaluators assess submissions based on these weighted criteria:

| Criteria | Weight | What Evaluators Look For |
|----------|--------|--------------------------|
| **Functionality** | 40% | Does the solution work as specified? Can evaluator replicate the result? |
| **Security** | 20% | Proper security group configuration, no overly permissive rules (0.0.0.0/0 for SSH) |
| **Verification** | 20% | Clear proof of working solution (screenshots, curl outputs, redis-cli responses) |
| **Documentation** | 10% | Clean, organized resource naming; evaluator can navigate Console easily |
| **Time Completion** | 5% | Finished within 2 hours; incomplete solutions lose marks |
| **Cost Awareness** | 5% | Free Tier usage where possible; unnecessary services avoided |

**Key Insight:** Verification carries 20% of marks. Always include proof outputs (terminal screenshots, browser screenshots, CLI command results).

---

## How to Use This Module

### Recommended Learning Path

1. **Start with Part 1: Solved Questions** (3 questions)
   - Read each solution top to bottom
   - Understand the marking scheme
   - Note evaluator expectations in verification steps

2. **Practice State-Level Questions** (8 questions)
   - Attempt each question without looking at solution
   - Time yourself (40 minutes per question)
   - Compare your approach with provided solution
   - Review "Common Mistakes" section after each attempt

3. **Progress to State++ Questions** (7 questions)
   - Build on State-Level skills with multi-subnet scenarios
   - Focus on NAT Gateway and ALB configurations
   - Practice security group chaining (ALB SG → EC2 SG → RDS SG)

4. **Time-Bound Mock Exams**
   - Select 3 random questions (1 State + 2 State++)
   - Set 2-hour timer
   - Complete all three with verification
   - Review what slowed you down

5. **Focus on Verification Steps**
   - Practice curl commands for HTTP testing
   - Master redis-cli for ElastiCache validation
   - Learn mysql client for RDS connectivity checks
   - Take clear screenshots (evaluators need to see success)

### Study Schedule Suggestion

- **Week 1-2**: Part 1 Solved Questions + first 4 State-Level questions
- **Week 3**: Remaining 4 State-Level questions + mock exam (3 State-Level questions, 2 hours)
- **Week 4**: All 7 State++ questions
- **Week 5**: Mixed mock exams (2 State + 1 State++ or 1 State + 2 State++)
- **Week 6**: Revision of common mistakes, speed optimization, verification practice

---

## Prerequisites

Before starting this module, ensure you have:

- AWS Free Tier account (or exam-provided account)
- Basic understanding of Linux command line
- Familiarity with networking concepts (IP addresses, CIDR, routing)
- Completed modules 03 (EC2), 05 (VPC), 06 (ALB), 07 (RDS), 08 (Caching) from this repository

---

## Next Steps

- **Part 1**: [Solved Past Questions](solved_questions/) - Start here to understand exam format
- **Part 2**: [State-Level Practice](state_level_practice/) - Build foundational skills
- **Part 3**: [State++ Practice](state_plus_practice/) - Master advanced scenarios
- **Practice Labs**: Detailed click-by-click walkthroughs for all 15 questions are in [../practice/README.md](../practice/README.md) with level-specific indexes in [../practice/state_level/](../practice/state_level/) and [../practice/state++_level/](../practice/state++_level/)

---

## Additional Resources

- **AWS Free Tier Details**: [https://aws.amazon.com/free/](https://aws.amazon.com/free/)
- **VPC Documentation**: Review [aws-worldskills-notes/05_vpc](../05_vpc)
- **ALB Documentation**: Review [aws-worldskills-notes/06_alb](../06_alb)
- **Caching Patterns**: Review [aws-worldskills-notes/08_caching](../08_caching)

---

**Good luck with your IndiaSkills State-Level preparation!** Remember: verification is 20% of your marks, so always prove your solution works with clear outputs and screenshots.
