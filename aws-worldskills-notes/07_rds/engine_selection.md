# RDS Database Engine Selection

**Duration**: 15 minutes  
**Difficulty**: Intermediate  
**Skills**: Engine comparison, Free Tier optimization, performance tuning

## MySQL vs PostgreSQL vs MariaDB

| Feature | MySQL | PostgreSQL | MariaDB |
|---------|-------|------------|---------|
| **Performance** | High read/write | Complex queries excel | Faster than MySQL |
| **ACID Compliance** | Yes (InnoDB) | Yes (native) | Yes (InnoDB) |
| **JSON Support** | Basic (5.7+) | Advanced (JSONB) | Basic |
| **Full-Text Search** | Yes | Advanced (GIN, GiST) | Yes |
| **Replication** | Async/Semi-sync | Async/Sync | Async/Semi-sync |
| **Concurrency** | Row-level locking | MVCC (better) | Row-level locking |
| **Extensions** | Limited | Rich (PostGIS, etc.) | MySQL compatible |
| **Learning Curve** | Easy | Moderate | Easy |
| **Community** | Large | Large | Growing |
| **Default Port** | 3306 | 5432 | 3306 |
| **Free Tier** | ✅ Yes | ✅ Yes | ✅ Yes |

## Free Tier Instance Classes

| Instance Class | vCPU | RAM | Network | Storage | Use Case |
|----------------|------|-----|---------|---------|----------|
| db.t2.micro | 1 | 1GB | Low | EBS only | Legacy, minimal workloads |
| db.t3.micro | 2 | 1GB | Up to 5 Gbps | EBS only | **Recommended for labs** |

**Free Tier Limit**: 750 instance hours/month (run 1 instance 24/7 for entire month)

## Storage Options

| Type | IOPS | Throughput | Use Case | Free Tier |
|------|------|------------|----------|-----------|
| General Purpose SSD (gp2) | 3 IOPS/GB (100-16k) | 250 MB/s | Balanced workloads | ✅ 20GB |
| General Purpose SSD (gp3) | 3000 baseline (up to 16k) | 125 MB/s baseline | Better than gp2 | ✅ 20GB |
| Provisioned IOPS (io1) | 1000-64k (configurable) | 1000 MB/s | High I/O apps | ❌ No |
| Magnetic (deprecated) | 100 | Varies | Legacy only | ❌ No |

**Recommendation**: Use **gp3** for new deployments (better performance at same cost)

## MySQL Specifics

**Versions**: 5.7 (maintenance mode), 8.0 (current, recommended)

**Key Features**:
- InnoDB storage engine (default, ACID-compliant)
- Connection limit: ~150 concurrent connections on t3.micro
- Charset: utf8mb4 recommended for full Unicode support
- Replication: Binary log (binlog) based
- Backup: Physical snapshots + binlog for PITR

**Best For**:
- Web applications (WordPress, Joomla, Drupal)
- E-commerce platforms
- General-purpose OLTP workloads
- Applications requiring MySQL compatibility

**Console Steps**:
1. Create database → **MySQL**
2. Version: **MySQL 8.0.35** (latest)
3. Templates: **Free tier**
4. DB instance identifier: `web-mysql-db`
5. Master username: `admin`
6. Instance configuration: **db.t3.micro**

**CLI Example**:
```bash
aws rds create-db-instance \
  --db-instance-identifier web-mysql-db \
  --db-instance-class db.t3.micro \
  --engine mysql \
  --engine-version 8.0.35 \
  --master-username admin \
  --master-user-password MySecurePass123 \
  --allocated-storage 20 \
  --storage-type gp3 \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-xxxxx \
  --db-subnet-group-name mydb-subnet-group \
  --backup-retention-period 7 \
  --region us-east-1
```

## PostgreSQL Specifics

**Versions**: 12.x, 13.x, 14.x, 15.x (recommended: 15.x)

**Key Features**:
- Advanced data types: JSONB, arrays, hstore, UUID
- Full-text search with ranking
- Geospatial support via PostGIS extension
- Connection limit: ~100 concurrent connections on t3.micro
- MVCC for better concurrency
- Common Table Expressions (CTEs) and window functions

**Best For**:
- Applications requiring complex queries
- JSON data storage and querying
- Geospatial applications (GIS)
- Data analytics and reporting
- Applications needing advanced SQL features

**Console Steps**:
1. Create database → **PostgreSQL**
2. Version: **PostgreSQL 15.4** (latest)
3. Templates: **Free tier**
4. DB instance identifier: `app-postgres-db`
5. Master username: `postgres`
6. Instance configuration: **db.t3.micro**

**CLI Example**:
```bash
aws rds create-db-instance \
  --db-instance-identifier app-postgres-db \
  --db-instance-class db.t3.micro \
  --engine postgres \
  --engine-version 15.4 \
  --master-username postgres \
  --master-user-password MySecurePass123 \
  --allocated-storage 20 \
  --storage-type gp3 \
  --no-publicly-accessible \
  --vpc-security-group-ids sg-xxxxx \
  --db-subnet-group-name mydb-subnet-group \
  --backup-retention-period 7 \
  --region us-east-1
```

## MariaDB Specifics

**Versions**: 10.6.x, 10.11.x (recommended: 10.11.x)

**Key Features**:
- Drop-in MySQL replacement with performance improvements
- Better query optimizer
- Additional storage engines (Aria, ColumnStore)
- Virtual columns and temporal tables
- Thread pool for better concurrency

**Best For**:
- MySQL migrations seeking better performance
- Applications requiring MySQL compatibility
- High-concurrency workloads
- Cost-sensitive deployments

**Console Steps**:
1. Create database → **MariaDB**
2. Version: **MariaDB 10.11.6**
3. Templates: **Free tier**
4. DB instance identifier: `cms-mariadb`
5. Instance configuration: **db.t3.micro**

## Engine Selection Decision Tree

```
Start
  ├─ Need advanced JSON/geospatial?
  │    └─ YES → PostgreSQL
  │    └─ NO → Continue
  ├─ Existing MySQL codebase?
  │    └─ YES → MySQL or MariaDB
  │    └─ NO → Continue
  ├─ Need maximum performance for standard SQL?
  │    └─ YES → MariaDB
  │    └─ NO → MySQL (most common choice)
  └─ Complex analytical queries?
       └─ YES → PostgreSQL
       └─ NO → MySQL
```

## Performance Comparison (db.t3.micro, 20GB gp3)

| Workload | MySQL | PostgreSQL | MariaDB |
|----------|-------|------------|---------|
| Simple reads (SELECT) | ~1200 qps | ~1000 qps | ~1300 qps |
| Simple writes (INSERT) | ~800 qps | ~700 qps | ~850 qps |
| Complex joins | ~400 qps | ~500 qps | ~450 qps |
| JSON queries | ~300 qps | ~600 qps | ~300 qps |
| Concurrent connections | 150 | 100 | 160 |

*Note: Benchmarks vary by workload and configuration*

## Migration Considerations

**MySQL → PostgreSQL**:
- Rewrite SQL: LIMIT → LIMIT/OFFSET, backticks → double quotes
- Data types: TINYINT → SMALLINT, DATETIME → TIMESTAMP
- Tools: AWS Database Migration Service (DMS)

**MySQL → MariaDB**:
- Minimal changes (drop-in replacement)
- Test application compatibility
- Update connection strings if needed

**PostgreSQL → MySQL**:
- Remove PostgreSQL-specific features (JSONB, arrays)
- Simplify complex queries
- Not recommended unless required

## License and Cost

| Engine | License | Free Tier | Beyond Free Tier |
|--------|---------|-----------|------------------|
| MySQL | Open Source (GPL) | ✅ Yes | ~$0.017/hour (t3.micro) |
| PostgreSQL | Open Source (PostgreSQL) | ✅ Yes | ~$0.017/hour (t3.micro) |
| MariaDB | Open Source (GPL) | ✅ Yes | ~$0.017/hour (t3.micro) |
| Oracle | Commercial (BYOL/included) | ❌ No | $0.140/hour+ |
| SQL Server | Commercial (BYOL/included) | ❌ No | $0.100/hour+ |

## Competition Tips

- **Default choice**: MySQL 8.0 (most familiar to evaluators)
- **For JSON workloads**: PostgreSQL 15
- **For performance edge**: MariaDB 10.11
- Always select **latest minor version** within major version
- Use **gp3 storage** for better performance
- Keep **db.t3.micro** for Free Tier compliance

## Common Mistakes

- Selecting Oracle/SQL Server (not Free Tier eligible)
- Choosing db.t2.small or larger (exceeds Free Tier)
- Using old engine versions (security concerns)
- Allocating >20GB storage (exceeds Free Tier)
- Mixing engines in multi-instance setups (confusion)

## Cross-References

- RDS overview: [aws-worldskills-notes/07_rds/overview.md](aws-worldskills-notes/07_rds/overview.md)
- Multi-AZ and Read Replicas: [aws-worldskills-notes/07_rds/multi_az_read_replica.md](aws-worldskills-notes/07_rds/multi_az_read_replica.md)
- Security and backups: [aws-worldskills-notes/07_rds/security_backup.md](aws-worldskills-notes/07_rds/security_backup.md)
- Hands-on lab: [aws-worldskills-notes/07_rds/server_lab.md](aws-worldskills-notes/07_rds/server_lab.md)
