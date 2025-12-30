# S3 Cost Optimization

## Cost Components

- **Storage**: Per GB-month by storage class.
- **Requests**: GET, PUT, COPY, POST, LIST priced per 1,000 requests.
- **Data Transfer Out**: To internet charged; within same region often free; cross-region charged.
- **Management**: Versioning (extra copies), replication, Inventory, Object Lock.

## Free Tier (12 Months)

- 5 GB S3 Standard storage
- 20,000 GET requests
- 2,000 PUT/COPY/POST/LIST requests
- 15 GB data transfer out (shared with other services)

## Storage Class Selection

| Class | When to Use | Retrieval | Notes |
|-------|-------------|-----------|-------|
| Standard | Frequent access (websites, active data) | ms | Default |
| Standard-IA | Infrequent, needs rapid access | ms | 30-day min, retrieval fee |
| One Zone-IA | Re-creatable, non-critical | ms | Single AZ, cheaper |
| Intelligent-Tiering | Unknown/changing patterns | ms | Auto-moves; small monthly monitoring per object |
| Glacier Instant Retrieval | Archives with instant needs | ms | Retrieval fee |
| Glacier Flexible Retrieval | Long-term, minutes retrieval | 1-5 min | Bulk/standard/expedited tiers |
| Glacier Deep Archive | Long-term, hours retrieval | up to 12h | Lowest cost |

## Lifecycle Best Practices

- Respect minimum storage durations: 30 days (IA), 90 days (Glacier tiers).
- Avoid many tiny objects in IA/Glacier (128 KB minimum billable for IA classes).
- Use prefixes/tags to target only the data that should transition.
- Always add **AbortIncompleteMultipartUpload** (e.g., 7 days).

## Request Optimization

- Use `aws s3 sync` to batch updates instead of many single PUTs.
- Enable CloudFront caching for read-heavy sites to reduce GETs.
- Use S3 Select to retrieve subsets of large objects (cuts transfer and compute).
- Compress assets (gzip/brotli) before upload.

## Data Transfer Optimization

- Keep compute in the same region as the bucket.
- Use S3 VPC endpoints for private traffic (no NAT data processing charges).
- Limit cross-region replication unless required.
- For public sites, keep assets small; consider CloudFront for global cache.

## Versioning Cost Management

- Enable versioning only when needed for rollback/compliance.
- Add lifecycle rules to transition non-current versions to IA/Glacier and expire after N days.
- Monitor version counts with S3 Storage Lens.

## Monitoring and Alerts

- **S3 Storage Lens**: Usage and cost visibility.
- **CloudWatch Metrics**: BucketSizeBytes, NumberOfObjects (per storage class).
- **Billing Alerts/Budgets**: Set monthly budgets (e.g., $5) with email alerts.
- **Cost Explorer**: Review monthly; filter by service=S3.

## Competition Cost Scenarios

### Scenario 1: 100 GB Website, 50k Visitors
- Assume Standard class, mostly GET.
- 100 GB storage ~ $2.30/month (us-east-1).
- 50k GET (0.05 million) ~ $0.02.
- Data transfer: dominant factor; 100 GB out at $0.09/GB = $9 (watch Free Tier 15 GB).

### Scenario 2: Backups 500 GB + 50 GB/month Growth
- Strategy: Store new data in Standard for 30 days, then Standard-IA; after 180 days move to Glacier Flexible Retrieval.
- Lifecycle: Current to IA at 30d; to Glacier at 180d; expire after 365d if allowed.
- Keeps monthly cost under typical $5-$10 targets for competition labs (depending on retention).

### Scenario 3: Unpredictable Access Patterns
- Use Intelligent-Tiering; lets S3 move objects between frequent/IA/archive tiers automatically.
- Good default when patterns are unknown and you cannot manage lifecycle manually.

## Cost Calculation Example (us-east-1)

- 10 GB in Standard: 10 × $0.023 = $0.23/month (likely fully covered by Free Tier 5 GB + light use).
- 50 GB in Standard-IA: 50 × $0.0125 = $0.63/month + retrieval per GB when accessed.
- 200 GB in Glacier Deep Archive: 200 × $0.00099 ≈ $0.20/month (retrieval extra, 12h delay).

## AWS CLI Examples

```bash
# Lifecycle: transitions, expiration, and abort incomplete multipart uploads
cat > lifecycle.json <<'EOF'
{
  "Rules": [{
    "ID": "CostControls",
    "Status": "Enabled",
    "Filter": {"Prefix": ""},
    "NoncurrentVersionTransitions": [
      {"NoncurrentDays": 30, "StorageClass": "STANDARD_IA"},
      {"NoncurrentDays": 90, "StorageClass": "GLACIER"}
    ],
    "NoncurrentVersionExpiration": {"NoncurrentDays": 180},
    "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
  }]
}
EOF

aws s3api put-bucket-lifecycle-configuration \
  --bucket my-bucket \
  --lifecycle-configuration file://lifecycle.json

# Verify lifecycle
aws s3api get-bucket-lifecycle-configuration --bucket my-bucket

# Optional: create a simple monthly budget with 80% alert
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{"BudgetName":"S3-Monthly","BudgetLimit":{"Amount":"5","Unit":"USD"},"TimeUnit":"MONTHLY","BudgetType":"COST"}' \
  --notifications-with-subscribers '[{"Notification":{"NotificationType":"ACTUAL","ComparisonOperator":"GREATER_THAN","Threshold":80},"Subscribers":[{"SubscriptionType":"EMAIL","Address":"you@example.com"}]}]'
```

## Python boto3 Example

```python
import boto3
import json

bucket = "my-bucket"

lifecycle = {
    "Rules": [{
        "ID": "CostControls",
        "Status": "Enabled",
        "Filter": {"Prefix": ""},
        "NoncurrentVersionTransitions": [
            {"NoncurrentDays": 30, "StorageClass": "STANDARD_IA"},
            {"NoncurrentDays": 90, "StorageClass": "GLACIER"}
        ],
        "NoncurrentVersionExpiration": {"NoncurrentDays": 180},
        "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }]
}

s3 = boto3.client("s3")
s3.put_bucket_lifecycle_configuration(
    Bucket=bucket,
    LifecycleConfiguration=lifecycle
)

print(json.dumps(
    s3.get_bucket_lifecycle_configuration(Bucket=bucket),
    indent=2
))
```

## Shutdown and Cleanup

- Empty buckets (all versions and delete markers) before deletion.
- `aws s3 rb s3://bucket-name --force` to remove bucket and contents.
- Release cross-region replication if configured.
- Re-check billing dashboard after 24 hours.

## Best Practices Summary

| Practice | Cost Impact | When to Use |
|----------|-------------|-------------|
| Use Standard only when needed | Saves on storage | Default, short-lived data |
| Lifecycle to IA/Glacier | Reduces long-term storage | Backups, logs, old versions |
| Abort incomplete multipart | Prevents stray charges | Always |
| Intelligent-Tiering for unknown patterns | Automates savings | Mixed/unknown workloads |
| Keep data in-region with compute | Avoids data transfer out | Apps and analytics |
| CloudFront caching | Reduces GET and transfer | Public static sites |
| Monitor with Storage Lens | Early detection of growth | Always |

## Verification Checklist

- [ ] Lifecycle rules in place for versioned buckets
- [ ] Storage class choices documented
- [ ] Budgets/alerts configured
- [ ] Multipart abort rule enabled
- [ ] Free Tier limits considered for current workload

## Next Steps

- Ensure public access is deliberate: see [bucket_policies.md](bucket_policies.md).
- For websites, follow [static_website.md](static_website.md) and pair with lifecycle rules to control cost.
