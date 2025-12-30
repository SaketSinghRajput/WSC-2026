# S3 Bucket Policies and Access Control

## Concept Overview

- **Bucket policies**: JSON documents attached to a bucket to control access at bucket/object level.
- **IAM policies**: Attach to users/roles; control what a principal can do across services.
- **ACLs**: Legacy, per-object grants; avoid unless required.
- **Block Public Access**: Four account/bucket-level settings to prevent public exposure; defaults should stay ON unless hosting a public website.

### Policy Elements

| Element | Purpose |
|---------|---------|
| Version | Policy language version (use 2012-10-17) |
| Statement | Array of permission statements |
| Effect | Allow or Deny |
| Principal | Who the policy applies to (account, user, role, or *) |
| Action | API actions (e.g., s3:GetObject) |
| Resource | Bucket or object ARNs (e.g., arn:aws:s3:::my-bucket/*) |
| Condition | Optional constraints (IP, SSL, VPC, etc.) |

## Block Public Access (BPA)

Four toggles (bucket or account level):
- Block public ACLs
- Ignore public ACLs
- Block public bucket policies
- Restrict public buckets (s3:PutBucketPolicy only for bucket owner)

Keep BPA **ON** for private data. Turn OFF only when intentionally hosting a public website, and then add a scoped bucket policy.

## Sample Policies

### Public Read (Website Hosting)

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "PublicReadGetObject",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::bucket-name/*"
  }]
}
```

### Restrict by IP Range

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "IPAllow",
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::bucket-name/*",
    "Condition": {"IpAddress": {"aws:SourceIp": "203.0.113.0/24"}}
  }]
}
```

### Enforce HTTPS Only

```json
{
  "Version": "2012-10-17",
  "Statement": [{
    "Sid": "DenyInsecureTransport",
    "Effect": "Deny",
    "Principal": "*",
    "Action": "s3:*",
    "Resource": [
      "arn:aws:s3:::bucket-name",
      "arn:aws:s3:::bucket-name/*"
    ],
    "Condition": {"Bool": {"aws:SecureTransport": "false"}}
  }]
}
```

## AWS Console Steps (Apply Policy)

1. S3 → Buckets → select bucket → **Permissions** tab.
2. **Block Public Access**: turn off only if you need public read; acknowledge warning.
3. **Bucket policy**: Edit → paste JSON → Save.
4. Re-open to confirm the policy saved without errors.

## AWS CLI Commands

```bash
# Apply policy from file
aws s3api put-bucket-policy \
  --bucket bucket-name \
  --policy file://policy.json

# Get current policy
aws s3api get-bucket-policy --bucket bucket-name

# Delete policy
aws s3api delete-bucket-policy --bucket bucket-name
```

## Python boto3 Example

```python
import json
import boto3

s3 = boto3.client('s3')
policy = {
    "Version": "2012-10-17",
    "Statement": [{
        "Sid": "PublicReadGetObject",
        "Effect": "Allow",
        "Principal": "*",
        "Action": "s3:GetObject",
        "Resource": "arn:aws:s3:::bucket-name/*"
    }]
}

s3.put_bucket_policy(
    Bucket="bucket-name",
    Policy=json.dumps(policy)
)
```

## Security Best Practices

- Default to private; enable BPA by default.
- Use IAM roles for EC2/Lambda access instead of long-lived keys.
- When making public, scope to read-only objects and specific bucket.
- Add HTTPS-only deny to prevent plaintext access.
- Log access (S3 server access logging or CloudTrail data events).

## Common Mistakes

- Missing `/*` in Resource when intending object-level access.
- BPA left on while applying public policy → access still blocked.
- Invalid JSON (trailing commas) causing policy rejection.
- Not specifying Principal correctly (e.g., account root ARN when needed).

## Verification Steps

- Fetch an object with curl: `curl -I https://bucket-name.s3.amazonaws.com/file.txt` (expect 200 or 403 depending on intent).
- For IP-restricted policies, test from allowed and disallowed IP ranges.
- Use AWS Policy Simulator for IAM principal testing.
- Check CloudTrail data events for denies.

## Competition Tips

- For static sites: disable BPA, apply public read policy, confirm index/error objects exist.
- For private data: keep BPA on, grant access via IAM roles or specific principals.
- Include the ARN with `/*` for object access; bucket ARN alone is insufficient for reads.
- Document your policy Sid and intent for clarity in submissions.
