# Amazon S3 Overview

## What is Amazon S3?

Amazon Simple Storage Service (S3) is **object storage** for the cloud. It stores data as objects inside buckets with virtually unlimited scale, 11 nines durability, and simple HTTP-based access.

- **Object storage vs block**: Objects (data + metadata) addressed by key; no block volumes or mounting.
- **Global namespace**: Bucket names are globally unique; data itself is stored in a specific AWS region you choose.
- **Flat structure**: Folders are a UI convenience based on key prefixes (e.g., `photos/2025/img1.jpg`).
- **Access via HTTPS**: REST API, AWS CLI, SDKs (boto3), and console.

## Core Concepts

| Concept | Description |
|---------|-------------|
| Bucket | Top-level container for objects; unique name; lives in one region |
| Object | Data + metadata + key; can be up to 5 TB |
| Key | Full path of the object (e.g., `site/index.html`) |
| Region | Physical location where data is stored |
| Storage Class | Pricing/performance profile for objects |
| Versioning | Keeps multiple versions of objects for rollback |
| Encryption | Server-side (SSE-S3, SSE-KMS) or client-side |
| Access Control | IAM, bucket policies, ACLs, Block Public Access |
| Events | Notifications to Lambda, SQS, SNS when objects change |

## Storage Classes (Summary)

| Class | Durability | Availability | Retrieval | Typical Use | Notes |
|-------|------------|--------------|-----------|-------------|-------|
| Standard | 11 9s | 99.99% | Milliseconds | Frequently accessed data | Default |
| Standard-IA | 11 9s | 99.9% | Milliseconds | Infrequent access, rapid retrieval | 30-day minimum, retrieval fee |
| One Zone-IA | 11 9s | 99.5% | Milliseconds | Non-critical, easily reproducible data | Single AZ, lower cost |
| Intelligent-Tiering | 11 9s | 99.9% | Milliseconds | Unknown/changing access | Auto-tiering across frequent/IA/archive tiers |
| Glacier Instant Retrieval | 11 9s | 99.9% | Milliseconds | Archives needing instant access | Retrieval fee |
| Glacier Flexible Retrieval | 11 9s | 99.9% | Minutes | Long-term archives | Bulk/standard/expedited options |
| Glacier Deep Archive | 11 9s | 99.9% | Hours (up to 12) | Long retention (years) | Lowest cost |

## Key Features

- **Durability**: 99.999999999% (11 9s) across devices and facilities.
- **Scalability**: Virtually unlimited objects and size (5 TB per object).
- **Versioning**: Protects against accidental deletes/overwrites.
- **Security**: IAM policies, bucket policies, ACLs, Block Public Access, encryption (SSE-S3, SSE-KMS).
- **Event Notifications**: Trigger Lambda/SQS/SNS on object events.
- **Data Management**: Lifecycle policies, replication (CRR/SRR), S3 Inventory, S3 Object Lock.
- **Performance**: Parallel uploads (multipart), transfer acceleration, VPC endpoints.

## Use Cases for Competitions

- Static website hosting (landing pages, portfolios).
- Backups and logs (CloudFront, ALB, application logs).
- Application assets (images, CSS/JS bundles).
- Data lakes and CSV/Parquet staging.
- Artifact storage for CI/CD outputs.

## Architecture Diagram

```mermaid
graph LR
    User[User/Browser] -->|HTTPS| S3[S3 Bucket (Region)]
    S3 -->|Object PUT/GET| Obj[(Objects with Keys)]
    Obj -->|Events| Lambda[Lambda]
    Obj -->|Metrics| CW[CloudWatch]
```

## AWS Console Steps (Create and Upload)

1. Open AWS Console → S3.
2. Click **Create bucket**.
3. Bucket name: `my-unique-bucket-2025` (lowercase, hyphens).
4. Region: choose closest (e.g., us-east-1).
5. **Block Public Access**: leave ON unless hosting a public site.
6. Create bucket.
7. Open bucket → **Upload** → add files → **Upload**.

## AWS CLI Quick Commands

```bash
# Create bucket (region-specific endpoint)
aws s3 mb s3://my-unique-bucket-2025 --region us-east-1

# List buckets
aws s3 ls

# Upload a file
aws s3 cp ./file.txt s3://my-unique-bucket-2025/file.txt

# Sync a folder
aws s3 sync ./site s3://my-unique-bucket-2025/site

# List objects
aws s3 ls s3://my-unique-bucket-2025 --recursive --human-readable
```

## Python boto3 Basics

```python
import boto3
s3 = boto3.client('s3')

# Upload
s3.upload_file('file.txt', 'my-unique-bucket-2025', 'file.txt')

# List objects
resp = s3.list_objects_v2(Bucket='my-unique-bucket-2025', Prefix='')
for obj in resp.get('Contents', []):
    print(obj['Key'], obj['Size'])
```

## Free Tier (12 Months)

- 5 GB of Standard storage
- 20,000 GET requests
- 2,000 PUT, COPY, POST, LIST requests
- 15 GB data transfer out aggregated across AWS each month

## Competition Tips

- Bucket names: lowercase, numbers, hyphens; no uppercase or underscores.
- Pick region close to judges/users to reduce latency.
- Default to **private**; open only what must be public.
- For static sites: disable Block Public Access, add bucket policy for public read, and confirm index/error docs.
- Use versioning when updates are frequent; pair with lifecycle rules to control cost.

## Common Mistakes

- Using uppercase or spaces in bucket names.
- Forgetting region selection during creation (affects latency and pricing).
- Making buckets public without need; missing Block Public Access toggle.
- Uploading with same key and no versioning (overwrites silently).

## Verification Checklist

- [ ] Bucket created in correct region
- [ ] Naming follows S3 rules
- [ ] Objects uploaded and listed
- [ ] Permissions set appropriately (private vs public)
- [ ] Free Tier limits understood for workload

## Next Steps

- [bucket_policies.md](bucket_policies.md) for access control
- [versioning_lifecycle.md](versioning_lifecycle.md) for protection and automation
- [static_website.md](static_website.md) for website hosting
- [server_lab.md](server_lab.md) for a full hands-on lab
- [cost_optimization.md](cost_optimization.md) for pricing and savings
