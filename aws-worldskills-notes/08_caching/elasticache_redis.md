# ElastiCache Redis

**Duration**: 30 minutes  
**Difficulty**: Intermediate  
**Skills**: Redis commands, cluster configuration, VPC integration

## Redis vs Memcached Detailed Comparison

| Feature | Redis | Memcached | WorldSkills Recommendation |
|---------|-------|-----------|----------------------------|
| **Data Structures** | Strings, lists, sets, sorted sets, hashes | Strings only | Redis (versatile) |
| **Persistence** | Optional disk persistence | In-memory only | Redis (data durability) |
| **Replication** | Master-replica support | None | Redis (high availability) |
| **Pub/Sub Messaging** | Yes (real-time notifications) | No | Redis (event-driven apps) |
| **Transactions** | Yes (MULTI/EXEC) | No | Redis (atomic operations) |
| **Lua Scripting** | Yes (server-side logic) | No | Redis (complex operations) |
| **Multi-threading** | Single-threaded | Multi-threaded | Depends on workload |
| **Memory Efficiency** | Good | Excellent | Memcached for simple K-V |
| **Use Case Fit** | Complex data, persistence needed | Simple key-value, high throughput | **Redis for competitions** |

**Verdict**: Use Redis for WorldSkills competitions unless specifically required otherwise.

## Redis Data Structures

### 1. Strings (Simple Key-Value)

**Use Cases**: Caching HTML fragments, API responses, counters

```bash
# Set string
SET user:1000:name "John Doe"

# Get string
GET user:1000:name
# Output: "John Doe"

# Set with expiration (TTL)
SETEX session:abc123 3600 "user_id:1000"
# Expires in 3600 seconds (1 hour)

# Increment counter
SET page:views 0
INCR page:views
INCR page:views
GET page:views
# Output: "2"
```

### 2. Lists (Ordered Collections)

**Use Cases**: Message queues, recent items, activity feeds

```bash
# Push to list
LPUSH queue:emails "email1@example.com"
LPUSH queue:emails "email2@example.com"

# Pop from list (FIFO queue)
RPOP queue:emails
# Output: "email1@example.com"

# Get list length
LLEN queue:emails

# Get range
LRANGE recent:products 0 9
# Get first 10 recent products
```

### 3. Sets (Unique Items)

**Use Cases**: Tags, unique visitors, membership checks

```bash
# Add to set
SADD tags:product:123 "electronics"
SADD tags:product:123 "laptop"
SADD tags:product:123 "laptop"  # Duplicate ignored

# Get all members
SMEMBERS tags:product:123
# Output: "electronics", "laptop"

# Check membership
SISMEMBER tags:product:123 "laptop"
# Output: 1 (true)

# Set operations
SADD set1 "a" "b" "c"
SADD set2 "b" "c" "d"
SINTER set1 set2  # Intersection: b, c
SUNION set1 set2  # Union: a, b, c, d
```

### 4. Sorted Sets (Ranked Collections)

**Use Cases**: Leaderboards, priority queues, time-series data

```bash
# Add to sorted set (score, member)
ZADD leaderboard:game1 1000 "player1"
ZADD leaderboard:game1 1500 "player2"
ZADD leaderboard:game1 800 "player3"

# Get top 10 (highest scores)
ZREVRANGE leaderboard:game1 0 9 WITHSCORES
# Output: player2 1500, player1 1000, player3 800

# Get rank
ZRANK leaderboard:game1 "player1"
# Output: 1 (0-indexed)

# Increment score
ZINCRBY leaderboard:game1 100 "player3"
```

### 5. Hashes (Object Storage)

**Use Cases**: User profiles, product details, configuration

```bash
# Set hash fields
HSET user:1000 name "John Doe"
HSET user:1000 email "john@example.com"
HSET user:1000 age 30

# Get single field
HGET user:1000 name
# Output: "John Doe"

# Get all fields
HGETALL user:1000
# Output: name "John Doe", email "john@example.com", age 30

# Set multiple fields
HMSET product:500 name "Laptop" price 999.99 stock 50

# Increment numeric field
HINCRBY product:500 stock -1  # Decrement stock
```

## Redis Commands for Competitions

### Essential Commands

| Command | Purpose | Example |
|---------|---------|---------|
| SET | Store string value | `SET key value` |
| GET | Retrieve string value | `GET key` |
| SETEX | Set with expiration | `SETEX key 3600 value` |
| DEL | Delete key | `DEL key` |
| EXISTS | Check if key exists | `EXISTS key` |
| EXPIRE | Set expiration on existing key | `EXPIRE key 3600` |
| TTL | Get remaining TTL | `TTL key` |
| KEYS | Find keys by pattern | `KEYS user:*` |
| INCR/DECR | Increment/decrement | `INCR counter` |
| LPUSH/RPUSH | Push to list | `LPUSH queue item` |
| LPOP/RPOP | Pop from list | `RPOP queue` |
| SADD | Add to set | `SADD tags item` |
| SMEMBERS | Get set members | `SMEMBERS tags` |
| ZADD | Add to sorted set | `ZADD leaderboard 100 player` |
| HSET | Set hash field | `HSET user:1 name John` |
| HGETALL | Get all hash fields | `HGETALL user:1` |

## ElastiCache for Redis Configuration

### Node Type Selection

**Free Tier Options**:

| Node Type | vCPU | Memory | Network | Cost (beyond Free Tier) |
|-----------|------|--------|---------|-------------------------|
| cache.t2.micro | 1 | 555 MB | Low | $0.017/hour |
| cache.t3.micro | 2 | 512 MB | Up to 5 Gbps | $0.017/hour |

**Recommendation**: Use **cache.t3.micro** (better performance, same cost)

### Engine Version

- **Latest stable**: Redis 7.0.x (recommended)
- Older versions: 6.2.x, 6.0.x (maintained)

**Best Practice**: Select latest 7.0.x for competitions

### Port Configuration

- **Default port**: 6379
- **Custom port**: Possible but unnecessary for competitions
- **Security**: Never expose to internet (private subnet only)

### Parameter Groups

Parameter groups define configuration settings for Redis.

**Default parameter group**: Works for most competition scenarios

**Custom parameters** (optional):
- `maxmemory-policy`: `allkeys-lru` (evict least recently used when full)
- `timeout`: `300` (close idle connections after 5 minutes)
- `tcp-keepalive`: `300` (keep connections alive)

### Subnet Groups

Subnet groups define VPC subnets where ElastiCache nodes are placed.

**Best Practice**:
- Use **private subnets** (no internet gateway route)
- Span multiple AZs (high availability)
- Example: `10.0.2.0/24` (AZ-1a), `10.0.3.0/24` (AZ-1b)

### Security Groups

**Inbound Rules**:

| Type | Protocol | Port | Source | Description |
|------|----------|------|--------|-------------|
| Custom TCP | TCP | 6379 | sg-application | Allow from app tier |

**Outbound Rules**: Default (allow all) is fine

## AWS Console Steps: Create Redis Cluster

### Step-by-Step Process

1. **Navigate to ElastiCache**:
   - AWS Console → Search "ElastiCache" → Redis clusters

2. **Create Redis Cluster**:
   - Click **Create Redis cluster**

3. **Cluster Creation Method**:
   - **Easy create** (recommended for labs): Pre-configured defaults
   - **Standard create**: Full control over settings

4. **Cluster Settings** (Easy Create):
   - **Name**: `lab-redis-cluster`
   - **Description**: `Redis cluster for caching lab`
   - **Engine version**: Redis 7.0 (latest)
   - **Port**: 6379 (default)
   - **Node type**: cache.t2.micro
   - **Number of replicas**: 0 (single node for Free Tier)

5. **Location** (Standard Create):
   - **AWS Cloud** (default)
   - **Multi-AZ**: Disabled (Free Tier)

6. **Subnet Group**:
   - **Create new**:
     - Name: `redis-subnet-group`
     - VPC: Select your VPC
     - Subnets: Select 2+ private subnets in different AZs
   - **Use existing**: Select pre-created subnet group

7. **Security**:
   - **Security groups**: Create new or select existing
   - **Encryption**: Disabled for Free Tier (optional for production)

8. **Backup** (Optional, costs extra):
   - **Enable automatic backups**: No (Free Tier)
   - **Retention period**: 1-35 days

9. **Maintenance**:
   - **Maintenance window**: No preference (or specify off-peak hours)

10. **Create**:
    - Review settings
    - Click **Create**
    - Wait 5-10 minutes for "Available" status

11. **Retrieve Endpoint**:
    - Select cluster → **Details** → **Primary Endpoint**
    - Example: `lab-redis-cluster.abc123.0001.use1.cache.amazonaws.com:6379`

## AWS CLI Commands

### Create Subnet Group

```bash
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name redis-subnet-group \
  --cache-subnet-group-description "Redis subnet group" \
  --subnet-ids subnet-private1a subnet-private1b \
  --region us-east-1
```

### Create Security Group

```bash
# Create security group
SG_REDIS=$(aws ec2 create-security-group \
  --group-name sg-redis \
  --description "Redis cache security group" \
  --vpc-id vpc-xxxxx \
  --region us-east-1 \
  --query 'GroupId' \
  --output text)

# Add inbound rule (allow from application SG)
aws ec2 authorize-security-group-ingress \
  --group-id $SG_REDIS \
  --protocol tcp \
  --port 6379 \
  --source-group sg-application \
  --region us-east-1
```

### Create Redis Cluster

```bash
aws elasticache create-cache-cluster \
  --cache-cluster-id lab-redis-cluster \
  --engine redis \
  --engine-version 7.0 \
  --cache-node-type cache.t2.micro \
  --num-cache-nodes 1 \
  --cache-subnet-group-name redis-subnet-group \
  --security-group-ids $SG_REDIS \
  --region us-east-1
```

### Describe Cluster

```bash
aws elasticache describe-cache-clusters \
  --cache-cluster-id lab-redis-cluster \
  --show-cache-node-info \
  --region us-east-1
```

### Get Endpoint

```bash
aws elasticache describe-cache-clusters \
  --cache-cluster-id lab-redis-cluster \
  --show-cache-node-info \
  --query 'CacheClusters[0].CacheNodes[0].Endpoint' \
  --region us-east-1
```

## Connection from EC2/Lambda

### Retrieve Endpoint

**Console**: ElastiCache → Redis clusters → Select cluster → Primary Endpoint

**CLI**: See command above

**Format**: `cluster-name.xxxxx.0001.region.cache.amazonaws.com:6379`

### Security Group Configuration

1. **Redis security group**: Allow inbound port 6379 from application SG
2. **Application security group**: No changes needed (outbound allows all by default)

### Test Connectivity

```bash
# Install redis-cli on EC2
sudo yum install -y redis6

# Test connection
redis-cli -h lab-redis-cluster.abc123.0001.use1.cache.amazonaws.com ping
# Expected output: PONG

# Set and get a test value
redis-cli -h <endpoint> SET testkey "Hello Redis"
redis-cli -h <endpoint> GET testkey
# Output: "Hello Redis"
```

## Python boto3 Example

### Create Redis Cluster

```python
import boto3

# Create ElastiCache client
client = boto3.client('elasticache', region_name='us-east-1')

# Create Redis cluster
response = client.create_cache_cluster(
    CacheClusterId='lab-redis-cluster',
    Engine='redis',
    EngineVersion='7.0',
    CacheNodeType='cache.t2.micro',
    NumCacheNodes=1,
    CacheSubnetGroupName='redis-subnet-group',
    SecurityGroupIds=['sg-12345678'],
    Port=6379
)

print(f"Cluster created: {response['CacheCluster']['CacheClusterId']}")
```

### Get Cluster Endpoint

```python
import boto3

client = boto3.client('elasticache', region_name='us-east-1')

response = client.describe_cache_clusters(
    CacheClusterId='lab-redis-cluster',
    ShowCacheNodeInfo=True
)

endpoint = response['CacheClusters'][0]['CacheNodes'][0]['Endpoint']
print(f"Endpoint: {endpoint['Address']}:{endpoint['Port']}")
```

## Python redis-py Library

### Installation

```bash
pip3 install redis
```

### Basic Connection

```python
import redis

# Connect to Redis
r = redis.Redis(
    host='lab-redis-cluster.abc123.0001.use1.cache.amazonaws.com',
    port=6379,
    decode_responses=True  # Return strings instead of bytes
)

# Test connection
if r.ping():
    print("Connected to Redis!")
```

### String Operations

```python
import redis

r = redis.Redis(host='<endpoint>', port=6379, decode_responses=True)

# Set and get
r.set('user:1000:name', 'John Doe')
name = r.get('user:1000:name')
print(name)  # Output: John Doe

# Set with expiration (TTL)
r.setex('session:abc123', 3600, 'user_id:1000')
ttl = r.ttl('session:abc123')
print(f"Expires in {ttl} seconds")

# Increment counter
r.set('page:views', 0)
r.incr('page:views')
r.incr('page:views')
views = r.get('page:views')
print(f"Views: {views}")  # Output: 2
```

### Hash Operations

```python
# Store user profile as hash
r.hset('user:1000', 'name', 'John Doe')
r.hset('user:1000', 'email', 'john@example.com')
r.hset('user:1000', 'age', 30)

# Get single field
name = r.hget('user:1000', 'name')

# Get all fields
user = r.hgetall('user:1000')
print(user)
# Output: {'name': 'John Doe', 'email': 'john@example.com', 'age': '30'}

# Set multiple fields
r.hmset('product:500', {'name': 'Laptop', 'price': 999.99, 'stock': 50})
```

### List Operations

```python
# Push to queue
r.lpush('queue:emails', 'email1@example.com')
r.lpush('queue:emails', 'email2@example.com')

# Pop from queue (FIFO)
email = r.rpop('queue:emails')
print(email)  # Output: email1@example.com

# Get list length
length = r.llen('queue:emails')
```

### Set Operations

```python
# Add tags to product
r.sadd('tags:product:123', 'electronics', 'laptop', 'new')

# Get all tags
tags = r.smembers('tags:product:123')
print(tags)  # Output: {'electronics', 'laptop', 'new'}

# Check membership
is_member = r.sismember('tags:product:123', 'laptop')
print(is_member)  # Output: True
```

### Sorted Set Operations

```python
# Leaderboard
r.zadd('leaderboard:game1', {'player1': 1000, 'player2': 1500, 'player3': 800})

# Get top 10 with scores
top_players = r.zrevrange('leaderboard:game1', 0, 9, withscores=True)
print(top_players)
# Output: [('player2', 1500.0), ('player1', 1000.0), ('player3', 800.0)]

# Increment score
r.zincrby('leaderboard:game1', 100, 'player3')
```

## Monitoring and Troubleshooting

### CloudWatch Metrics

| Metric | Description | Target |
|--------|-------------|--------|
| CPUUtilization | CPU usage percentage | <75% |
| DatabaseMemoryUsagePercentage | Memory used | <80% |
| CacheHits | Successful cache retrievals | Maximize |
| CacheMisses | Failed cache retrievals | Minimize |
| Evictions | Items removed due to memory | <1% |
| NetworkBytesIn/Out | Data transfer | Monitor trends |

### Cache Hit Ratio

**Formula**: `CacheHits / (CacheHits + CacheMisses) × 100`

**Target**: >80% for effective caching

**View in Console**:
1. Select cluster → **Metrics** tab
2. View CacheHits and CacheMisses graphs
3. Calculate ratio manually or use CloudWatch Math

### Common Issues

#### Issue 1: Connection Timeout

**Symptoms**: Application cannot connect to Redis

**Causes**:
- Security group blocks port 6379
- EC2 and Redis in different VPCs
- Incorrect endpoint

**Fix**:
```bash
# Verify security group
aws ec2 describe-security-groups --group-ids sg-redis

# Test from EC2
redis-cli -h <endpoint> ping
```

#### Issue 2: Memory Eviction

**Symptoms**: Evictions metric increasing, cache hit ratio dropping

**Causes**:
- Node too small for workload
- No TTL set on keys

**Fix**:
- Upgrade to larger node type (cache.t2.small)
- Set TTL on all keys: `SETEX key 3600 value`

#### Issue 3: High CPU Usage

**Symptoms**: CPUUtilization >75%

**Causes**:
- Expensive operations (KEYS * on large dataset)
- Too many concurrent connections

**Fix**:
- Avoid `KEYS` command in production (use `SCAN`)
- Scale to larger node or add read replicas

## Competition Tips

- **Always use private subnets** for Redis (security best practice)
- **Set TTL on all cached data** to prevent memory exhaustion
- **Monitor cache hit ratio** (>80% proves caching is effective)
- **Use descriptive key names**: `user:1000:profile`, not `u1000p`
- **Test connectivity** from application tier before implementing caching
- **Document configuration**: node type, subnet group, security group

## Common Mistakes

- Placing Redis in public subnet (security risk)
- Not setting TTL (memory fills up, evictions occur)
- Using `KEYS *` in application code (performance issue)
- Security group allows 0.0.0.0/0 (should restrict to application SG)
- Forgetting to install redis-py library (`pip install redis`)
- Using wrong endpoint (writer vs reader)

## Cross-References

- Caching overview: [aws-worldskills-notes/08_caching/overview.md](aws-worldskills-notes/08_caching/overview.md)
- Caching patterns: [aws-worldskills-notes/08_caching/caching_patterns.md](aws-worldskills-notes/08_caching/caching_patterns.md)
- Hands-on lab: [aws-worldskills-notes/08_caching/server_lab.md](aws-worldskills-notes/08_caching/server_lab.md)
- VPC networking: [aws-worldskills-notes/05_vpc/overview.md](aws-worldskills-notes/05_vpc/overview.md)
- Security groups: [aws-worldskills-notes/03_ec2/security_groups.md](aws-worldskills-notes/03_ec2/security_groups.md)
