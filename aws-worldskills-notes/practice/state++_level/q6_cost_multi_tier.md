# Q6: Cost-Optimized Multi-Tier Architecture Design

## Lab Overview
- **Difficulty:** Intermediate
- **Estimated Time:** 60-75 minutes
- **AWS Services:** EC2, ALB, RDS, ElastiCache, CloudWatch, AWS Budgets
- **Region:** us-east-1
- **Skills Focus:** Architecture design, cost optimization, scaling strategies, Free Tier limits

## Prerequisites Check
- [ ] Completed State-Level Q1-Q8 and State++ Q1-Q5
- [ ] Understanding of cost drivers in each AWS service
- [ ] Ability to read AWS cost calculators

## Learning Objectives
- Design cost-optimized architecture within Free Tier
- Identify cost-saving opportunities
- Calculate ROI of different architectural choices
- Monitor and optimize ongoing costs
- Plan capacity with budget constraints

## Architecture Comparison

### Design A: Minimal Cost (Free Tier)
```
┌─────────────────────────────────┐
│      Internet (HTTP 80)          │
└────────────┬────────────────────┘
             │
      ┌──────▼──────┐
      │ EC2 t2.micro│ (Web + App + Cache)
      │   Public    │ (Single tier)
      └──────┬──────┘
             │
      ┌──────▼──────┐
      │ RDS MySQL   │
      │ t3.micro    │
      └─────────────┘

Cost: ~$0/month (Free Tier)
Availability: Single AZ (not HA)
Scalability: Limited
```

### Design B: Balanced (Recommended)
```
┌──────────────────────────────────────────────┐
│         Internet (HTTP 80)                    │
└──────────────────┬─────────────────────────────┘
                   │
         ┌─────────▼──────────┐
         │   ALB (Paid)       │
         │   Multi-AZ         │
         └────────┬───────────┘
                  │
    ┌─────────────┼────────────┐
    │             │            │
┌───▼──┐     ┌───▼──┐      ┌──▼───┐
│EC2   │     │EC2   │      │Redis │
│1a    │     │1b    │      │Private│
│t2m   │     │t2m   │      │cache  │
└───┬──┘     └───┬──┘      └──▲───┘
    │            │            │
    └────────────┼────────────┘
                 │
           ┌─────▼──────┐
           │ RDS MySQL  │
           │ t3.micro   │
           │ Multi-AZ   │
           └────────────┘

Cost: ~$16/month (ALB)
Availability: Multi-AZ
Scalability: Good
```

### Design C: Enterprise (Outside Free Tier)
```
(Includes NAT, more RDS instances, Multi-Master, etc.)
Cost: $500+/month
```

## Step-by-Step Console Instructions

### Step 1: Access AWS Pricing Calculator
**Web:** https://calculator.aws/

**Create New Estimate:**
1. Click "Create estimate"
2. Add services for the "Balanced" design above
3. Configurations:
   - Quantity: 2× EC2 t2.micro
   - Region: us-east-1
   - Pricing model: On-Demand
   - Estimated usage: 730 hours/month (continuous)
   - Data transfer: 10 GB/month ingress + egress

[SCREENSHOT: AWS Pricing Calculator with estimate]

### Step 2: Compare Costs by Service
**Calculator Estimate Breakdown:**

| Service | Instance | Hours/Month | Cost |
|---------|----------|-------------|------|
| EC2 | t2.micro ×2 | 730×2 = 1460 | Free (750×2 included) |
| RDS | db.t3.micro | 730 | Free (750 included) |
| RDS | Storage 20GB | 1 month | Free (20GB included) |
| ElastiCache | cache.t3.micro | 730 | Free (750 for 12mo) |
| ALB | Load Balancer | 730 hours | $11.70 (~$0.016/hour) |
| ALB | LCU | ~1-10/month | $4.50 (~$0.006/LCU) |
| **Total** | | | **~$16.20/month** |

[SCREENSHOT: Cost breakdown table]

### Step 3: Identify Cost Drivers
**What costs money immediately:**
1. ALB: $16.20/month (largest cost!)
2. Data transfer out: $0.09/GB (after 1GB free)
3. Unassociated Elastic IPs: $0.005/hour each
4. NAT Gateway: $0.045/hour

**What's Free in Year 1:**
1. EC2 750 hours/month (2 × t2.micro = 1500 hours; excess $0.011/hour)
2. RDS 750 hours/month (db.t3.micro)
3. ElastiCache 750 hours/month (cache.t3.micro) - first 12 months only
4. S3 5GB/month
5. Data transfer 1GB/month

[SCREENSHOT: Cost drivers highlighted]

### Step 4: Create AWS Budgets
**Console Navigation:** Budgets → Create budget

**Budget 1: Total Monthly Spending**
1. Budget name: `practice-total-monthly`
2. Budget type: Monthly
3. Budgeted amount: $25/month
4. Alert threshold: 80% ($20) → sends notification at $16
5. Alert recipients: your@email.com
6. Create

**Budget 2: EC2 Spending**
1. Budget name: `practice-ec2-monthly`
2. Budget type: Monthly
3. Service: EC2
4. Budgeted amount: $5
5. Alert threshold: 100% ($5)
6. Create

**Budget 3: ALB Spending**
1. Budget name: `practice-alb-monthly`
2. Budget type: Monthly
3. Service: Elastic Load Balancing
4. Budgeted amount: $20
5. Alert threshold: 100%
6. Create

[SCREENSHOT: Budgets dashboard]

### Step 5: Estimate Costs of Alternative Designs

**Design A (Minimal) Estimate:**
```
1× EC2 t2.micro (shared web+app+cache): Free (750h included)
1× RDS db.t3.micro: Free (750h included)
NO ALB, NO Multi-AZ, NO Redis

Total: $0/month (but no HA, manual scaling)
```

**Design B (Balanced) Estimate:**
```
2× EC2 t2.micro: Free (1500h = 750h×2 included)
1× RDS db.t3.micro Multi-AZ: +$0 (750h included, backup free)
1× ElastiCache cache.t3.micro: Free (750h for 12mo)
1× ALB: $16.20/month
1× NAT Gateway (if used): +$0.045/hour = $32.4/month (NOT recommended)

Total: $16-48/month (highly available, autoscalable)
```

**Design C (Enterprise) Estimate:**
```
2× EC2 t2.small: $18.28/month
2× RDS db.t3.small Multi-AZ: $89.42/month
1× ElastiCache cache.t3.small: $62.49/month
1× ALB: $16.20/month
1× NAT Gateway × 2: +$64.80/month
1× Private subnet with managed NAT

Total: $250+/month (enterprise HA, production-ready)
```

[SCREENSHOT: Three cost estimates side-by-side]

### Step 6: Design Architecture Within Budget

**Given $20/month budget:**
1. ALB: $16.20 (mandatory for HA)
2. EC2: $0 (free tier)
3. RDS: $0 (free tier)
4. Redis: $0 (free tier first 12 months)
5. **Constraint: Cannot afford NAT Gateway** ($32/month)
6. **Design strategy:** Use public subnets + IGW, no private tier

**Budget Plan:**
- Year 1: $16.20/month (ALB only, free tier)
- Year 2: +$0 (ElastiCache no longer free, but can budget for cache.t2 $5-7/month)
- Beyond: Consider switching to RDS Aurora Serverless or DynamoDB

[SCREENSHOT: Budget-constrained architecture design]

### Step 7: Cost Optimization Recommendations

**Immediate Savings (Implement Now):**
1. ✅ Use Free Tier instances (t2.micro, db.t3.micro)
2. ✅ Set up billing alarms ($5, $10, $25 thresholds)
3. ✅ Release unassociated Elastic IPs (costing $0.005/hour each)
4. ✅ Use S3 lifecycle policies (delete old objects after 90 days)
5. ✅ Delete unused RDS snapshots (storage costs $0.095/GB/month)
6. ✅ Stop (don't terminate) instances if pausing work (no storage charges)

**Medium-term Optimizations (6-12 months):**
1. 📊 Review Reserved Instances after usage patterns clear (up to 70% discount)
2. 📊 Migrate to Compute Savings Plans for EC2 (more flexible than RI)
3. 📊 Consider Spot instances for non-critical workloads (90% discount but can be interrupted)
4. 🗄️ Replace EBS with S3 for long-term storage (cheaper for infrequent access)

**Advanced Optimizations (1+ year):**
1. 🚀 Migrate to serverless: Lambda + DynamoDB (pay-per-request)
2. 🚀 Use Aurora Serverless instead of RDS (scales to zero, pay for executions)
3. 🚀 Implement caching layers to reduce database queries (ElastiCache)
4. 🚀 Use CloudFront CDN for static content (reduces data transfer costs)

[SCREENSHOT: Cost optimization checklist]

### Step 8: Set Up Cost Monitoring Dashboards
**Console Navigation:** CloudWatch → Dashboards → Create dashboard

**Dashboard Name:** `practice-cost-monitoring`

**Add Widgets:**

**Widget 1: Estimated Monthly Costs (Cost Explorer)**
1. Metric: Estimated Charges
2. Dimension: Service
3. Time period: Last 30 days
4. Shows cost trend

**Widget 2: Usage by Service (Cost Explorer)**
1. Breakdown by service
2. EC2 hours, RDS hours, ALB hours
3. Shows which services consuming most

**Widget 3: Budget vs Actual (Budgets)**
1. Display active budgets
2. Current spending vs threshold

[SCREENSHOT: Cost monitoring dashboard]

### Step 9: Analyze Architecture Trade-offs
**Console Navigation:** Spreadsheet (or create own table)

| Aspect | Design A (Min) | Design B (Balanced) | Design C (Enterprise) |
|--------|----------------|--------------------|----------------------|
| **Cost/Month** | $0 | $16.20 | $250+ |
| **Availability** | 99.5% (1 AZ) | 99.99% (2 AZ) | 99.99%+ (3 AZ) |
| **Capacity** | Limited | Medium | Large |
| **Scaling** | Manual | Automatic via ALB | Auto-scaling groups |
| **Failover** | Manual restart | Automatic ALB drain | Automatic multi-region |
| **Data redundancy** | Single copy | Replicated (RDS Multi-AZ) | Multi-region replication |
| **Best For** | Learning, dev | Production for startups | Enterprise SLA |

[SCREENSHOT: Architecture comparison table]

### Step 10: Calculate Total Cost of Ownership (TCO)
**For 3-year period (Design B - Balanced):**

```
Year 1:
- EC2 × 2: $0 (Free Tier)
- RDS: $0 (Free Tier)
- ElastiCache: $0 (Free Tier)
- ALB: $16.20 × 12 = $194.40
- Subtotal Year 1: $194.40

Year 2:
- EC2 × 2: $0 → $53.04 (leaving free tier) [10% excess]
- RDS: $0 → $8.30 (Multi-AZ premium)
- ElastiCache: $0 → $84 (no longer free; cache.t3.micro $7/mo)
- ALB: $16.20 × 12 = $194.40
- Subtotal Year 2: $339.74

Year 3 (same as Year 2):
- Subtotal Year 3: $339.74

**Total 3-Year TCO: $194.40 + $339.74 + $339.74 = $873.88**
**Average cost: $24.30/month**
```

[SCREENSHOT: TCO calculation]

### Step 11: Create Cost Optimization Plan Document
**File:** cost-optimization-plan.md

**Template:**
```markdown
# Cost Optimization Plan for Practice Labs

## Current Architecture (Design B - Balanced)
- 2× EC2 t2.micro (multi-AZ)
- 1× RDS db.t3.micro (Multi-AZ)
- 1× ALB
- 1× ElastiCache cache.t3.micro (free for 12 months)

## Estimated Monthly Cost
- Year 1: $16.20/month
- Year 2+: $100-150/month

## Cost Optimization Roadmap

### Month 1-3: Foundation (No additional cost)
- [ ] Set billing alarms ($5, $10, $25)
- [ ] Delete unused resources (old snapshots, EIPs)
- [ ] Enable VPC Flow Logs to identify traffic patterns
- [ ] Document baseline metrics

### Month 4-6: Review & Analyze
- [ ] Analyze usage patterns (Cost Explorer)
- [ ] Identify underutilized resources
- [ ] Calculate Reserved Instance break-even
- [ ] Plan for end of Free Tier (month 12)

### Month 7-12: Optimization Phase 1
- [ ] Downsize dev/test instances to t3.nano if possible
- [ ] Implement auto-scaling for predictable traffic
- [ ] Use S3 for static assets (cheaper than EBS)
- [ ] Schedule development instances to stop at night ($50/month savings)

### Month 13-18: Optimization Phase 2 (Post-Free Tier)
- [ ] Evaluate Reserved Instances (1-year commitment for 30% discount)
- [ ] Consider switching to Aurora Serverless for variable workloads
- [ ] Implement DynamoDB for specific workloads (pay-per-request)
- [ ] Review ALB necessity (consider NLB for cost reduction)

### Month 19+: Long-term Strategy
- [ ] Migrate to serverless architecture (Lambda + managed services)
- [ ] Implement CloudFront CDN for global distribution
- [ ] Use Spot instances for non-critical workloads
- [ ] Evaluate multi-region disaster recovery ROI

## Expected Savings
- Stopping dev instances at night: $50-100/month
- Reserved Instances (1-year): 30% discount = $30/month
- Serverless migration: 70% reduction = $70+/month
```

[SCREENSHOT: Cost optimization plan document]

## CLI Alternative (AWS Cost Explorer)

```bash
REGION=us-east-1

# Get estimated charges
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics "BlendedCost" \
  --group-by Type=DIMENSION,Key=SERVICE \
  --region $REGION \
  --query 'ResultsByTime[*].[TimePeriod,Total]'

# Get cost forecast (next 3 months)
aws ce get-cost-forecast \
  --time-period Start=2024-01-01,End=2024-04-01 \
  --metric BLENDED_COST \
  --granularity MONTHLY \
  --region $REGION

# Get usage (hours) for specific service
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity DAILY \
  --metrics "UsageQuantity" \
  --group-by Type=DIMENSION,Key=PURCHASE_TYPE \
  --region $REGION \
  --filter file://ec2-filter.json

# List budgets
aws budgets describe-budgets --account-id 123456789012 --region $REGION
```

## Verification Checklist

1. **AWS Pricing Calculator**
   - [ ] Estimate created for Design B (Balanced)
   - [ ] ALB cost: $16.20/month calculated
   - [ ] Free Tier services shown as $0
   - [ ] [SCREENSHOT: Pricing estimate]

2. **Cost Breakdown**
   - [ ] Identified ALB as largest cost
   - [ ] Documented Free Tier limits (750h EC2, 750h RDS, 750h ElastiCache Year 1)
   - [ ] Calculated excess usage charges
   - [ ] [SCREENSHOT: Cost breakdown table]

3. **Budgets Configured**
   - [ ] Total monthly budget: $25/month
   - [ ] Service budgets: EC2 $5, ALB $20
   - [ ] Alerts configured for 80-100% threshold
   - [ ] Notifications enabled
   - [ ] [SCREENSHOT: Budgets dashboard]

4. **Architecture Comparison**
   - [ ] Design A: $0/month (minimal, no HA)
   - [ ] Design B: $16/month (balanced, recommended)
   - [ ] Design C: $250+/month (enterprise)
   - [ ] Trade-offs documented
   - [ ] [SCREENSHOT: Comparison table]

5. **Optimization Recommendations**
   - [ ] Immediate actions (billing alarms, delete EIPs)
   - [ ] Medium-term (Reserved Instances, Spot instances)
   - [ ] Long-term (serverless migration, CDN)
   - [ ] [SCREENSHOT: Optimization checklist]

6. **Cost Monitoring Dashboard**
   - [ ] CloudWatch dashboard created
   - [ ] Estimated charges widget
   - [ ] Service breakdown widget
   - [ ] Budget status widget
   - [ ] [SCREENSHOT: Dashboard]

7. **TCO Calculation**
   - [ ] 3-year cost projection completed
   - [ ] Year 1: $194.40 (ALB only)
   - [ ] Year 2-3: $339+/month (post-free tier)
   - [ ] Total TCO: ~$873.88
   - [ ] [SCREENSHOT: TCO spreadsheet]

8. **Cost Optimization Plan**
   - [ ] Document created with 6 phases
   - [ ] Savings estimates for each optimization
   - [ ] Timeline: Month 1-3 foundation, 4-6 analysis, 7-12 optimization
   - [ ] Expected long-term savings: 70%+
   - [ ] [SCREENSHOT: Cost plan document]

## Troubleshooting Guide

- **Pricing calculator gives different numbers**
  - Cause: Different assumptions (region, data transfer, instance size)
  - Fix: Verify region (us-east-1), instance type (t2.micro), usage (730 hours)

- **Budget alerts not triggering**
  - Cause: Charges not yet incurred in current period; email subscriptions not confirmed
  - Fix: Confirm SNS subscription email; budgets evaluate daily at midnight

- **Cost exceeding estimate**
  - Cause: More instances running than planned; data transfer or EBS storage charges
  - Fix: Verify running instances match architecture; check data transfer in Cost Explorer

- **Free Tier limit exceeded**
  - Cause: Used more than 750 hours/month for service
  - Fix: Downsize instances; stop instances when not in use; upgrade in Year 2

- **Snapshot storage charges**
  - Cause: Old RDS snapshots or undeleted EBS snapshots
  - Fix: Delete snapshots; review retention policies in automated backup settings

## Cleanup Instructions

1. Delete CloudWatch dashboard
2. Delete budgets
3. [Keep monitoring for cost trends]

## Mark Mapping (Exam Scoring)

| Task | Marks | Criteria | Your Score |
|------|-------|----------|------------|
| Pricing calculation | 3 | Design B estimated at $16.20/month | [ ] |
| Cost breakdown | 3 | Services identified, Free Tier limits documented | [ ] |
| Budget setup | 3 | Monthly, service-level, alerts configured | [ ] |
| Architecture comparison | 3 | Design A/B/C costs and trade-offs analyzed | [ ] |
| Optimization plan | 3 | 6-phase roadmap with savings targets | [ ] |
| TCO calculation | 2 | 3-year projection calculated accurately | [ ] |
| Monitoring dashboard | 2 | CloudWatch dashboard with cost widgets | [ ] |
| **Total** | **19** | | **[ ]** |

## Key Takeaways
- **ALB is the main cost:** $16.20/month dominates practice lab budget; Free Tier covers compute/DB
- **Free Tier window (12 months):** Critical for cost optimization; plan for post-free-tier expenses
- **Cost increases over time:** Year 1 ($16/mo) to Year 2+ ($100+/mo) when free tier expires
- **Optimization requires monitoring:** Use CloudWatch, Cost Explorer, and budgets to track trends
- **Architectural choices matter:** Design B (ALB + multi-AZ) vs A (single server) difference is mainly ALB

## Next Steps
- Progress to State++ Q7: Bastion host and private access patterns
- Implement recommended optimizations from plan
- Review AWS cost optimization tools and services

## Related Resources
- AWS Pricing Calculator: https://calculator.aws/
- Cost Explorer: https://console.aws.amazon.com/cost-management/
- Budgets: https://console.aws.amazon.com/billing/home#/budgets
- Free Tier details: https://aws.amazon.com/free/
- Cost optimization: https://aws.amazon.com/architecture/cost-optimization/
