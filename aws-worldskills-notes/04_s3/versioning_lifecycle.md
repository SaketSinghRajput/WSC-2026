# S3 Versioning and Lifecycle Management

## Versioning Overview

- **What**: Keeps multiple versions of an object (with unique Version IDs).
- **Why**: Protects against accidental deletes/overwrites; enables rollbacks and audits.
- **Behavior**: Cannot be disabled once enabled; you can **suspend** to stop creating new versions.
- **Deletes**: A DELETE creates a delete marker; previous versions remain unless explicitly removed.

## Enable Versioning

### Console
1. S3 → Buckets → select bucket → **Properties** tab.
2. **Versioning** → Edit → Enable → Save.

### CLI

```bash
aws s3api put-bucket-versioning \
  --bucket bucket-name \
  --versioning-configuration Status=Enabled
```

## Working with Versions

- **List versions**:
```bash
aws s3api list-object-versions --bucket bucket-name
```
- **Restore previous version**: Re-upload that version or copy it over the current key.
- **Delete a specific version**:
```bash
aws s3api delete-object \
  --bucket bucket-name \
  --key path/to/file.txt \
  --version-id <version-id>
```
- **Remove delete marker** (to undelete): delete the delete marker version.

## Lifecycle Rules Overview

Lifecycle rules automate transitions and expirations to control cost:
- Transition objects between storage classes (Standard → IA → Glacier).
- Expire current/non-current versions after a period.
- Abort incomplete multipart uploads.

## Example Lifecycle Rules

### 1) Transition to IA then Glacier

- After 30 days: Standard → Standard-IA.
- After 90 days: Standard-IA → Glacier Flexible Retrieval.

### 2) Expire Old Versions

- Delete non-current versions after 60 days.

### 3) Clean Up Incomplete Multipart Uploads

- Abort incomplete multipart uploads after 7 days.

## Lifecycle JSON Example

```json
{
  "Rules": [
    {
      "ID": "TransitionAndArchive",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER"}
      ],
      "NoncurrentVersionTransitions": [
        {"NoncurrentDays": 30, "StorageClass": "STANDARD_IA"},
        {"NoncurrentDays": 90, "StorageClass": "GLACIER"}
      ]
    },
    {
      "ID": "ExpireOldVersions",
      "Status": "Enabled",
      "Filter": {"Prefix": ""},
      "NoncurrentVersionExpiration": {"NoncurrentDays": 60}
    },
    {
      "ID": "AbortMultipart",
      "Status": "Enabled",
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }
  ]
}
```

## AWS Console Steps (Lifecycle)

1. Bucket → **Management** tab → **Lifecycle rules** → Create rule.
2. Name rule (e.g., "Transition-Archive").
3. Scope: choose whole bucket or prefix.
4. Add transitions: 30 days to Standard-IA; 90 days to Glacier.
5. Add non-current transitions: 30 days to Standard-IA; 90 days to Glacier.
6. Add expiration for non-current versions after 60 days.
7. Add abort incomplete multipart after 7 days.
8. Save rule and verify status = Enabled.

## AWS CLI Command

```bash
aws s3api put-bucket-lifecycle-configuration \
  --bucket bucket-name \
  --lifecycle-configuration file://lifecycle.json
```

## Python boto3 Example

```python
import json
import boto3

s3 = boto3.client('s3')
with open('lifecycle.json') as f:
    config = json.load(f)

s3.put_bucket_lifecycle_configuration(
    Bucket='bucket-name',
    LifecycleConfiguration=config
)
```

## Cost Optimization Strategy

- Enable versioning only when rollback is needed.
- Pair versioning with lifecycle transitions to IA/Glacier for older versions.
- Set non-current expiration to avoid unbounded cost growth.
- Clean up incomplete multipart uploads to stop stray charges.

## Competition Scenario

**Scenario**: Bucket has 100 GB growing by 10 GB/month. Goal: keep storage under $5/month.
- Strategy: Versioning ON. Transition non-current to Standard-IA at 30 days; to Glacier at 90 days. Expire non-current after 180 days. Abort incomplete uploads after 7 days. Monitor with S3 Storage Lens.

## Verification Checklist

- [ ] Versioning status shows **Enabled** (or **Suspended** if intentionally paused)
- [ ] `list-object-versions` shows version IDs
- [ ] Lifecycle rule status **Enabled** and targets correct prefixes
- [ ] Transition and expiration timings match requirements
- [ ] Incomplete multipart cleanup rule present

## Common Mistakes

- Forgetting that delete without version ID creates delete marker (object not actually removed).
- Missing `NoncurrentVersionExpiration` leading to accumulating cost.
- Not accounting for minimum storage duration (30 days IA, 90 days Glacier).
- Uploading many small (<128 KB) objects to IA classes (minimum billable object size).

## Troubleshooting

- Rule not running: ensure Status=Enabled and filter/prefix correct.
- Objects not transitioning: check object age and minimum storage duration.
- Access denied on lifecycle API: requires `s3:PutLifecycleConfiguration` permission.

## Next Steps

- Apply lifecycle rules to buckets with versioning to keep Free Tier usage in check.
- Move to [static_website.md](static_website.md) to host content publicly.
