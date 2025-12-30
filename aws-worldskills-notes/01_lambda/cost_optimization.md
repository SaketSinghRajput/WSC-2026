# Lambda Cost Optimization - Free Tier Maximization

## Lambda Free Tier Details

AWS Lambda provides a permanent Free Tier that does not expire after 12 months, making it ideal for learning, development, and low-traffic production applications.

### Monthly Free Tier Allowances

**Compute Time**: 400,000 GB-seconds per month
- GB-seconds = (Memory allocation in GB) × (Execution duration in seconds)
- Example: 128 MB (0.125 GB) function running 200ms (0.2s) = 0.025 GB-seconds per invocation

**Requests**: 1,000,000 requests per month
- Includes all invocation types (synchronous, asynchronous, event source mapping)

**Duration Calculation**:
- Measured from start of execution to return or termination
- Rounded up to nearest 1 millisecond
- Minimum charge: 1ms

**Memory Allocation**:
- Range: 128 MB to 10,240 MB (10 GB)
- Increments: 1 MB
- CPU and network allocated proportionally to memory

### Free Tier Coverage Examples

| Memory | Duration | GB-Seconds | Max Free Invocations | Typical Use Case |
|--------|----------|------------|---------------------|------------------|
| 128 MB | 100ms | 0.0125 | 32,000,000 | Simple API responses |
| 128 MB | 200ms | 0.025 | 16,000,000 | API with DynamoDB access |
| 256 MB | 500ms | 0.125 | 3,200,000 | Image resizing |
| 512 MB | 1s | 0.5 | 800,000 | Data processing |
| 1024 MB | 3s | 3.0 | 133,333 | Video transcoding |
| 2048 MB | 5s | 10.0 | 40,000 | ML inference |

**Key Insight**: Lower memory and shorter duration allow significantly more free invocations.

### Practical Free Tier Capacity

For WorldSkills practice and typical competition scenarios:

**API Backend** (128 MB, 200ms average):
- Free Tier supports: 16 million requests/month
- Daily practice: 500,000 requests/day available
- Competition simulation: Unlimited within Free Tier

**File Processing** (512 MB, 5 seconds):
- Free Tier supports: 80,000 files/month
- Daily practice: 2,600 files/day
- More than sufficient for competition preparation

**Scheduled Tasks** (256 MB, 1 second, hourly):
- Monthly executions: 720 (24 hours × 30 days)
- GB-seconds used: 180 GB-seconds
- Free Tier remaining: 399,820 GB-seconds (99.95%)

## Pricing Beyond Free Tier

Understanding pricing helps optimize even within Free Tier limits.

### Request Pricing

**$0.20 per 1 million requests**
- After first 1 million free requests
- Applies to all invocation types

**Example Cost**:
- 2 million requests/month = 1 million free + 1 million paid
- Cost: $0.20

### Compute Pricing

**$0.0000166667 per GB-second**
- After first 400,000 free GB-seconds

**Example Cost Calculation**:

**Scenario**: 128 MB function, 300ms duration, 5 million requests/month
- GB-seconds per request: 0.128 GB × 0.3s = 0.0384 GB-seconds
- Total GB-seconds: 5,000,000 × 0.0384 = 192,000 GB-seconds
- Free Tier: 192,000 GB-seconds (fully covered)
- Compute cost: $0.00
- Request cost: (5,000,000 - 1,000,000) × $0.20 / 1,000,000 = $0.80
- **Total cost**: $0.80/month

### Cost Comparison: Lambda vs EC2

**Scenario**: REST API serving 100,000 requests/month

**Lambda** (128 MB, 200ms average):
- Requests: 100,000 (free)
- GB-seconds: 100,000 × 0.025 = 2,500 (free)
- **Cost**: $0.00/month

**EC2 t2.micro** (1 vCPU, 1 GB RAM):
- Running 24/7: 730 hours/month
- First 750 hours free (first 12 months only)
- After Free Tier: 730 × $0.0116 = $8.47/month
- **Cost**: $0.00/month (first year), $8.47/month thereafter

**Lambda Advantages**:
- Permanent Free Tier (no 12-month limit)
- Zero cost when not invoked
- Automatic scaling for traffic spikes
- No server maintenance

## Memory Allocation Impact

Memory allocation affects both cost and performance. Finding the optimal setting is critical for cost-efficiency.

### How Memory Affects Cost

Higher memory allocation increases GB-seconds consumed:

| Memory | Cost per Second | 100ms Execution | 1s Execution |
|--------|----------------|-----------------|--------------|
| 128 MB | $0.000002083 | $0.0000002083 | $0.000002083 |
| 256 MB | $0.000004167 | $0.0000004167 | $0.000004167 |
| 512 MB | $0.000008333 | $0.0000008333 | $0.000008333 |
| 1024 MB | $0.000016667 | $0.0000016667 | $0.000016667 |

**Example**: 1 million invocations, 500ms execution
- 128 MB: 62.5 GB-seconds (free)
- 512 MB: 250 GB-seconds (free)
- 1024 MB: 500 GB-seconds (free)

All scenarios stay within Free Tier, but higher memory consumes more of allowance.

### How Memory Affects Performance

AWS allocates CPU power proportionally to memory:
- 128 MB: ~0.083 vCPU
- 1024 MB: ~0.66 vCPU
- 1769 MB: 1 full vCPU
- 10,240 MB: ~6 vCPUs

**Performance Impact**:

**CPU-Bound Task** (image processing):
- 128 MB: 5000ms execution
- 512 MB: 1500ms execution
- 1024 MB: 800ms execution

**I/O-Bound Task** (DynamoDB query):
- 128 MB: 200ms execution
- 512 MB: 190ms execution (minimal improvement)

### Finding Optimal Memory

**Strategy 1: Start Small**
1. Begin with 128 MB
2. Test execution duration
3. If timeout occurs or duration >5s, double memory
4. Repeat until acceptable performance

**Strategy 2: Use Lambda Power Tuning**

AWS Lambda Power Tuning is an open-source tool that tests your function at different memory settings and identifies cost-optimal configuration.

**Installation**:
1. Deploy via AWS Serverless Application Repository
2. Search for "aws-lambda-power-tuning"
3. Deploy stack
4. Execute state machine with function ARN

**Interpretation**:
- Tool generates cost vs performance graph
- Identifies "sweet spot" where cost is minimized for acceptable performance
- Typical result: 256-512 MB for most use cases

### Recommended Memory by Use Case

| Use Case | Recommended Memory | Reasoning |
|----------|-------------------|-----------|
| Simple API (JSON response) | 128 MB | Minimal processing, I/O bound |
| API with DynamoDB | 128-256 MB | I/O bound, minimal computation |
| Image thumbnail (small) | 512 MB | CPU bound, faster execution offsets higher memory cost |
| Data transformation | 256-512 MB | Moderate computation |
| Machine learning inference | 1024-3008 MB | CPU intensive, requires more memory for models |

**WorldSkills Recommendation**: Use 128 MB for all competition tasks unless timeout occurs. Judges favor cost optimization.

## Execution Time Optimization

Reducing execution duration lowers costs and improves user experience.

### Code Optimization Techniques

**1. Minimize Cold Starts**

Cold starts add 100ms-2s latency. Optimize by:

**Initialize Globally**:
```python
import boto3

# Good: Initialize once per container
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    # Reuse table client
    table.put_item(Item={...})
```

**Avoid In-Handler Initialization**:
```python
# Bad: Initialize per invocation
def lambda_handler(event, context):
    dynamodb = boto3.resource('dynamodb')  # Recreated every time
    table = dynamodb.Table('Users')
```

**2. Connection Pooling**

Database connections are expensive to create. Reuse across invocations:

```python
import pymysql

# Global scope
connection = None

def lambda_handler(event, context):
    global connection
    
    # Reuse existing connection if available
    if connection is None or not connection.open:
        connection = pymysql.connect(
            host='mydb.example.com',
            user='admin',
            password='password',
            database='mydb'
        )
    
    # Use connection
    with connection.cursor() as cursor:
        cursor.execute("SELECT * FROM users")
        results = cursor.fetchall()
    
    return results
```

**3. Minimize Dependencies**

Fewer imports reduce cold start time:

**Before** (500ms cold start):
```python
import pandas as pd
import numpy as np
import matplotlib
from sklearn import preprocessing

def lambda_handler(event, context):
    # Simple data processing
    data = [1, 2, 3, 4, 5]
    return sum(data) / len(data)
```

**After** (100ms cold start):
```python
def lambda_handler(event, context):
    # Simple data processing without heavy libraries
    data = [1, 2, 3, 4, 5]
    return sum(data) / len(data)
```

**4. Use Lambda Layers for Shared Dependencies**

Lambda Layers reduce deployment package size and share dependencies across functions:

**Benefits**:
- Faster deployment updates (update code without redeploying dependencies)
- Smaller deployment packages
- Shared code across multiple functions

**Example**:
- Layer: boto3, requests, custom utilities
- Function code: Only application logic

**5. Optimize Algorithms**

Choose efficient algorithms for data processing:

**Inefficient** (1000ms):
```python
def lambda_handler(event, context):
    numbers = list(range(10000))
    results = []
    
    # O(n²) complexity
    for i in numbers:
        for j in numbers:
            if i * j > 100:
                results.append(i * j)
    
    return len(results)
```

**Efficient** (50ms):
```python
def lambda_handler(event, context):
    numbers = list(range(10000))
    
    # O(n) complexity
    results = [i * j for i in numbers for j in numbers if i * j > 100]
    
    return len(results)
```

### Provisioned Concurrency

Provisioned Concurrency keeps functions initialized and ready, eliminating cold starts.

**Cost**: Additional charge for provisioned capacity
- $0.0000041667 per GB-second (10x regular pricing)
- Not recommended for Free Tier optimization

**When to Use**:
- Production applications requiring <100ms latency
- Predictable traffic patterns
- Not for competition practice (adds cost)

## Concurrent Execution Limits

Understanding concurrency helps predict costs and avoid throttling.

### Account-Level Concurrency

**Default Limit**: 1,000 concurrent executions per region

**Calculation**:
Concurrency = (Requests per second) × (Average duration in seconds)

**Example**:
- 100 requests/second
- 500ms average duration
- Concurrency: 100 × 0.5 = 50

**Free Tier Impact**: All concurrent executions count toward Free Tier, regardless of concurrency level.

### Reserved Concurrency

Reserve dedicated capacity for critical functions.

**Purpose**:
- Guarantee capacity for function
- Prevent other functions from consuming all concurrency
- Not needed for competition practice (low traffic)

**Cost**: No additional charge for reserved concurrency itself, but ensures function can run when account reaches concurrency limit.

## Cost Monitoring

Track Lambda usage to avoid unexpected charges and optimize resource allocation.

### CloudWatch Metrics

Lambda automatically publishes metrics to CloudWatch:

**Key Metrics**:
- **Invocations**: Total number of function executions
- **Duration**: Average, minimum, maximum execution time
- **Errors**: Failed invocations (exceptions not caught)
- **Throttles**: Rejected invocations due to concurrency limits
- **ConcurrentExecutions**: Number of simultaneous executions

**Viewing Metrics (AWS Console)**:
1. Open Lambda function
2. Click "Monitor" tab
3. View graphs for invocations, duration, errors
4. Time range: 1 hour to 1 week

**Cost Estimation from Metrics**:

Example metrics:
- Invocations: 500,000 (last 30 days)
- Average duration: 250ms
- Memory: 128 MB

GB-seconds calculation:
- Per request: 0.128 GB × 0.25s = 0.032 GB-seconds
- Total: 500,000 × 0.032 = 16,000 GB-seconds
- Free Tier: 400,000 GB-seconds
- **Result**: Fully covered by Free Tier

### Billing Alarms

Set up CloudWatch billing alarms to receive email notifications when costs exceed thresholds.

**Creating Billing Alarm**:

1. **Enable Billing Alerts**:
   - Open AWS Console
   - Navigate to Billing Dashboard
   - Preferences > Billing Alerts
   - Enable "Receive Billing Alerts"

2. **Create Alarm**:
   - Navigate to CloudWatch
   - Alarms > Create Alarm
   - Select Metric: Billing > Total Estimated Charge
   - Condition: Greater than $5 (or desired threshold)
   - Configure SNS notification (email)
   - Create alarm

**Recommended Thresholds for Practice**:
- $5: Early warning
- $10: Review usage immediately
- $20: Critical alert

### AWS Cost Explorer

AWS Cost Explorer provides detailed cost breakdowns and forecasting.

**Accessing Cost Explorer**:
1. Navigate to Billing Dashboard
2. Click "Cost Explorer"
3. Enable Cost Explorer (first-time setup)
4. View cost graphs and reports

**Useful Reports**:
- **Service Costs**: Filter by "Lambda" to see Lambda-specific charges
- **Usage Type**: Distinguish between request charges and compute charges
- **Daily Costs**: Track costs over time to identify spikes
- **Forecast**: Predict month-end costs based on current usage

**Filtering for Lambda**:
- Service: AWS Lambda
- Usage Type: Request (request charges) or Lambda-GB-Second (compute charges)
- Region: us-east-1 (or your region)

### AWS CLI Cost Monitoring

**Get Lambda Metrics**:
```bash
# Get invocation count (last 7 days)
aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Invocations \
    --dimensions Name=FunctionName,Value=MyFunction \
    --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 86400 \
    --statistics Sum

# Get average duration
aws cloudwatch get-metric-statistics \
    --namespace AWS/Lambda \
    --metric-name Duration \
    --dimensions Name=FunctionName,Value=MyFunction \
    --start-time $(date -u -d '7 days ago' +%Y-%m-%dT%H:%M:%S) \
    --end-time $(date -u +%Y-%m-%dT%H:%M:%S) \
    --period 86400 \
    --statistics Average
```

## Free Tier Maximization Strategies

### Strategy 1: Use Minimum Memory for I/O-Bound Tasks

Most competition tasks involve API calls and database queries (I/O-bound). These benefit minimally from higher memory.

**Pattern**:
- API Gateway + Lambda + DynamoDB: 128 MB
- S3 event processing (read/write): 128 MB
- SNS/SQS message processing: 128 MB

**Exception**: CPU-intensive tasks (image processing, data transformation) may benefit from 256-512 MB.

### Strategy 2: Optimize Code to Reduce Execution Time

Faster execution = lower costs:

**Code Review Checklist**:
- [ ] Dependencies initialized in global scope
- [ ] Database connections pooled
- [ ] Minimal imports (remove unused libraries)
- [ ] Efficient algorithms (avoid nested loops)
- [ ] Early returns for validation errors

**Example Optimization**:

**Before** (300ms, 0.0375 GB-seconds):
```python
import pandas as pd

def lambda_handler(event, context):
    df = pd.DataFrame(event['records'])
    result = df.sum()
    return result.to_dict()
```

**After** (50ms, 0.00625 GB-seconds):
```python
def lambda_handler(event, context):
    records = event['records']
    totals = {}
    for record in records:
        for key, value in record.items():
            totals[key] = totals.get(key, 0) + value
    return totals
```

**Savings**: 83% reduction in GB-seconds

### Strategy 3: Batch Operations

Process multiple items per invocation instead of one-at-a-time:

**Inefficient** (1000 invocations):
```python
# Invoked 1000 times for 1000 records
def lambda_handler(event, context):
    record = event['record']
    process_single_record(record)
```

**Efficient** (10 invocations):
```python
# Invoked 10 times for batches of 100 records
def lambda_handler(event, context):
    batch = event['Records']  # 100 records per batch
    for record in batch:
        process_single_record(record)
```

**Savings**: 90% reduction in invocations and cold starts

### Strategy 4: Delete Unused Functions

Functions don't incur charges when not invoked, but good practice is to delete test functions to avoid accidental invocations.

**Cleanup Checklist**:
- [ ] Delete all Lambda functions after practice session
- [ ] Remove associated IAM roles
- [ ] Delete CloudWatch Log groups (optional, minimal cost)
- [ ] Verify AWS Cost Explorer shows $0.00 Lambda charges

### Strategy 5: Monitor Free Tier Usage

Check Free Tier usage regularly to ensure staying within limits.

**AWS Free Tier Dashboard**:
1. Navigate to Billing Dashboard
2. Click "Free Tier" in left sidebar
3. View Lambda usage:
   - Requests: X of 1,000,000 used
   - Compute: X of 400,000 GB-seconds used
4. Review usage percentage to plan practice schedule

## Best Practices for WorldSkills Competition

### Before Competition

**Practice Efficiency**:
- Complete labs 3-5 times to optimize implementation speed
- Pre-write Lambda function templates
- Memorize IAM policy structures
- Practice with 128 MB memory setting

**Cost Preparation**:
- Confirm Free Tier usage is near zero before competition week
- Set billing alarms at $5, $10, $20
- Enable Free Tier usage tracking emails

### During Competition

**Memory Selection**:
- Default to 128 MB for all functions
- Only increase if timeout occurs during testing

**Timeout Configuration**:
- API endpoints: 5-10 seconds (balance between cost and reliability)
- File processing: 30-60 seconds
- Scheduled tasks: 60-300 seconds

**Testing Strategy**:
- Test functions incrementally to minimize invocations
- Use Lambda console test events (free) before API Gateway testing
- Limit browser testing to 2-3 attempts (each API call counts)

### After Competition

**Immediate Cleanup**:
- Delete all Lambda functions
- Delete API Gateway APIs
- Delete DynamoDB tables
- Remove IAM roles
- Verify $0.00 charges in Cost Explorer

## Cleanup Checklist

Complete this checklist after every practice session:

- [ ] **Lambda Functions**: Delete all test functions
- [ ] **IAM Roles**: Delete Lambda execution roles
- [ ] **CloudWatch Logs**: Delete log groups (optional, <$1/month for logs)
- [ ] **API Gateway**: Delete all APIs
- [ ] **DynamoDB**: Delete all tables
- [ ] **S3 Buckets**: Empty and delete all buckets used for Lambda deployments
- [ ] **CloudWatch Alarms**: Keep billing alarms active
- [ ] **Cost Explorer**: Verify Lambda charges are $0.00
- [ ] **Free Tier Dashboard**: Verify usage percentages

## Cost Calculation Examples

### Example 1: Simple API (Low Traffic)

**Specifications**:
- Memory: 128 MB
- Average duration: 200ms
- Monthly requests: 10,000

**Calculations**:
- GB-seconds per request: 0.128 × 0.2 = 0.0256
- Total GB-seconds: 10,000 × 0.0256 = 256
- Request charges: 10,000 requests (within Free Tier)
- Compute charges: 256 GB-seconds (within Free Tier)

**Cost**: $0.00/month

### Example 2: File Processing (Moderate Traffic)

**Specifications**:
- Memory: 512 MB
- Average duration: 3 seconds
- Monthly requests: 50,000

**Calculations**:
- GB-seconds per request: 0.512 × 3 = 1.536
- Total GB-seconds: 50,000 × 1.536 = 76,800
- Request charges: 50,000 requests (within Free Tier)
- Compute charges: 76,800 GB-seconds (within Free Tier)

**Cost**: $0.00/month

### Example 3: High-Volume API (Exceeding Free Tier)

**Specifications**:
- Memory: 128 MB
- Average duration: 150ms
- Monthly requests: 5,000,000

**Calculations**:
- GB-seconds per request: 0.128 × 0.15 = 0.0192
- Total GB-seconds: 5,000,000 × 0.0192 = 96,000 (within Free Tier)
- Request charges: (5,000,000 - 1,000,000) = 4,000,000 paid requests
- Cost: 4,000,000 × $0.20 / 1,000,000 = $0.80

**Cost**: $0.80/month (compute free, minimal request charge)

## Key Takeaways

1. **Free Tier is Generous**: 1 million requests and 400,000 GB-seconds cover extensive practice and competition usage
2. **Memory Matters**: 128 MB is optimal for most competition tasks, maximizing free invocations
3. **Code Optimization**: Reducing execution time by 50% doubles Free Tier capacity
4. **Monitoring is Critical**: Set billing alarms and check Free Tier dashboard weekly
5. **Cleanup is Essential**: Delete all resources after practice to avoid accidental charges
6. **Lambda vs EC2**: Lambda's permanent Free Tier and pay-per-use model make it more cost-effective for learning and low-traffic applications

## Next Steps

After mastering Lambda cost optimization:
- [02_api_gateway/cost_optimization.md](../02_api_gateway/cost_optimization.md): Optimize API Gateway costs
- [04_s3/cost_optimization.md](../04_s3/cost_optimization.md): S3 storage cost management
- [09_full_server_scenarios/](../09_full_server_scenarios/): Apply cost optimization to complete architectures
