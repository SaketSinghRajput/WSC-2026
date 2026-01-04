# Q3: S3 Static Website Hosting

## Lab Overview
- **Difficulty:** Beginner
- **Estimated Time:** 35-40 minutes
- **AWS Services:** S3, Bucket Policies
- **Region:** us-east-1
- **Cost:** Free Tier (5GB storage, 20,000 GET requests)

## Prerequisites Check
- [ ] AWS account and S3 permissions
- [ ] Text editor for HTML
- [ ] Basic HTML knowledge

## Learning Objectives
- Create unique S3 bucket for static hosting
- Configure static website hosting with index and error documents
- Apply public-read bucket policy safely
- Upload and validate website content

## Architecture Overview
```mermaid
flowchart LR
    Browser --> Endpoint[S3 Website Endpoint]
    Endpoint --> Policy[Bucket Policy (Public Read)]
    Policy --> Objects[index.html / error.html]
```

## Step-by-Step Console Instructions

### Step 1: Create HTML Files Locally
Create `index.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>State Practice Q3</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #f0f0f0; }
        h1 { color: #232f3e; }
        p { color: #666; }
    </style>
</head>
<body>
    <h1>Welcome to State Practice Q3</h1>
    <p>This is a static website hosted on Amazon S3</p>
    <p>IndiaSkills State-Level Practice</p>
</body>
</html>
```

Create `error.html`:
```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>404 Error</title>
    <style>
        body { font-family: Arial, sans-serif; text-align: center; padding: 50px; background: #ffebee; }
        h1 { color: #c62828; }
    </style>
</head>
<body>
    <h1>404 - Page Not Found</h1>
    <p>The page you are looking for does not exist.</p>
</body>
</html>
```

Save both in a folder (e.g., `q3-website`).

[SCREENSHOT: HTML files in folder]

### Step 2: Create S3 Bucket
**Console Navigation:** S3 → Buckets → Create bucket

**Detailed Steps:**
1. Bucket name: `state-practice-site-q3-<your-initials>-<random>` (globally unique)
2. Region: us-east-1
3. Object Ownership: ACLs disabled
4. Block Public Access: **Uncheck** "Block all public access" and acknowledge
5. Versioning: Disable
6. Tags: Name = `state-practice-q3`
7. Default encryption: SSE-S3
8. Click "Create bucket"

[SCREENSHOT: Bucket creation success]

### Step 3: Upload HTML Files
1. Open bucket → "Upload"
2. Add files: select `index.html` and `error.html`
3. Leave permissions/properties default
4. Click "Upload" → wait for success

[SCREENSHOT: Upload success with 2 objects]

### Step 4: Enable Static Website Hosting
1. Bucket → Properties tab → Static website hosting → Edit
2. Enable hosting: Host a static website
3. Index document: `index.html`
4. Error document: `error.html`
5. Save changes
6. Copy the bucket website endpoint URL (e.g., `http://state-practice-site-q3-xyz.s3-website-us-east-1.amazonaws.com`)

[SCREENSHOT: Static website hosting enabled with endpoint]

### Step 5: Create Bucket Policy for Public Read
1. Bucket → Permissions tab → Bucket policy → Edit
2. Paste policy (replace `YOUR-BUCKET-NAME`):
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
    }
  ]
}
```
3. Save changes

[SCREENSHOT: Bucket policy saved]

## CLI Alternative (Copy-Paste Ready)
```bash
REGION=us-east-1
BUCKET_NAME="state-practice-site-q3-$(whoami)-$(date +%s)"
echo "Bucket name: $BUCKET_NAME"

# Create bucket
aws s3api create-bucket --bucket $BUCKET_NAME --region $REGION
echo "Bucket created"

# Disable block public access
aws s3api put-public-access-block \
  --bucket $BUCKET_NAME \
  --region $REGION \
  --public-access-block-configuration \
    'BlockPublicAcls=false,IgnorePublicAcls=false,BlockPublicPolicy=false,RestrictPublicBuckets=false'
echo "Public access enabled"

# Create bucket policy file
cat > bucket-policy.json <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublicReadGetObject",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:GetObject",
      "Resource": "arn:aws:s3:::${BUCKET_NAME}/*"
    }
  ]
}
EOF

# Apply bucket policy
aws s3api put-bucket-policy \
  --bucket $BUCKET_NAME \
  --policy file://bucket-policy.json
echo "Bucket policy applied"

# Enable static website hosting
aws s3api put-bucket-website \
  --bucket $BUCKET_NAME \
  --website-configuration '{
    "IndexDocument": {"Suffix": "index.html"},
    "ErrorDocument": {"Key": "error.html"}
  }'
echo "Static website hosting enabled"

# Upload files
aws s3 cp index.html s3://${BUCKET_NAME}/
aws s3 cp error.html s3://${BUCKET_NAME}/
echo "Files uploaded"

# Get website endpoint
WEBSITE_URL="http://${BUCKET_NAME}.s3-website-${REGION}.amazonaws.com"
echo "Website URL: $WEBSITE_URL"
```

## Verification Checklist

1. **Bucket Configuration**
   - S3 → Buckets: bucket exists in us-east-1, public access off
   - [SCREENSHOT: Bucket list and settings]

2. **Objects Upload**
   - Objects tab shows `index.html` and `error.html`
   - [SCREENSHOT: Objects list]

3. **Static Website Hosting**
   - Properties → Static website hosting: Enabled, index/error set
   - [SCREENSHOT: Static website hosting configuration]

4. **Bucket Policy**
   - Permissions → Bucket policy shows JSON with `/*` suffix
   - [SCREENSHOT: Bucket policy JSON]

5. **Website Access Test (Index)**
   - Browser: open website endpoint → homepage loads
   - [SCREENSHOT: Website homepage]

6. **Website Access Test (Error Page)**
   - Browser: `<endpoint>/nonexistent.html` → custom 404 page
   - [SCREENSHOT: Error page]

7. **CLI Access Test**
   - `curl -I http://<endpoint>` → HTTP/1.1 200 OK
   - `curl http://<endpoint>` → HTML content
   - `curl -I http://<endpoint>/missing` → HTTP/1.1 404 Not Found
   - [SCREENSHOT: curl outputs]

## Troubleshooting Guide

- **403 Forbidden**
  - Check block public access is off; bucket policy applied; Resource ARN ends with `/*`; allow `s3:GetObject`

- **404 on Index**
  - Ensure `index.html` uploaded and index document set to `index.html`

- **Bucket Name Exists**
  - Use unique suffix (initials + timestamp); lowercase and hyphens only

- **Cannot Disable Block Public Access**
  - Check account-level block public access settings; uncheck at account level then retry

- **Endpoint Not Working**
  - Use website endpoint format `http://<bucket>.s3-website-us-east-1.amazonaws.com`; do not use https or s3.amazonaws.com object URL

- **Policy JSON Error**
  - Validate JSON; ensure double quotes; correct bucket name; include `/*`

## Cleanup Instructions

**Console Cleanup:**
1. Empty bucket: S3 → Bucket → Empty → confirm
2. Delete bucket: select bucket → Delete → type name to confirm

**CLI Cleanup:**
```bash
REGION=us-east-1
aws s3 rm s3://${BUCKET_NAME} --recursive --region $REGION
aws s3api delete-bucket --bucket $BUCKET_NAME --region $REGION
rm bucket-policy.json
```

**Verification:** S3 console shows no bucket for this lab

## Mark Mapping (Exam Scoring)

| Task | Marks | Criteria | Your Score |
|------|-------|----------|------------|
| Bucket creation | 3 | us-east-1, unique name, block public access disabled | [ ] |
| Public access configuration | 3 | Block public access disabled with acknowledgment | [ ] |
| File upload | 2 | index.html and error.html uploaded | [ ] |
| Static hosting enable | 3 | Hosting enabled with correct index/error docs | [ ] |
| Bucket policy | 5 | Valid public-read policy with correct ARN and /* | [ ] |
| Index/error config | 2 | Document names set correctly | [ ] |
| Verification | 5 | Website accessible, index and 404 pages working | [ ] |
| Documentation | 2 | Screenshots and evidence captured | [ ] |
| **Total** | **25** | | **[ ]** |

## Key Takeaways
- S3 bucket names are globally unique; use lowercase and hyphens
- Static website hosting is HTTP-only; use correct website endpoint
- Public-read policy must include object ARN with `/*`
- Block public access must be disabled for public sites
- Error document improves UX for missing pages

## Next Steps
- Complete Q4: RDS Backup to practice database management
- Explore S3 versioning in 04_s3/versioning_lifecycle.md
- Learn CloudFront for HTTPS/CDN in advanced scenarios

## Related Resources
- Main practice file: 10_indskills/state_level_practice.md (Q3)
- S3 service guide: 04_s3/
- Static website guide: 04_s3/static_website.md
- Bucket policies guide: 04_s3/bucket_policies.md
