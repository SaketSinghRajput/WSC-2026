# Scenario 1: Serverless Web Application (Blog)

## Problem Statement
Build a complete serverless blog that supports authenticated article creation, public article retrieval, and low-latency global delivery using only managed services. Target a 90-minute implementation suitable for WorldSkills state-level competitors while staying within the AWS Free Tier.

## Architecture Components
- S3 static website hosting for HTML/CSS/JS frontend
- API Gateway REST API with methods: GET /articles, POST /articles, GET /articles/{id}
- Lambda functions: ListArticles, CreateArticle, GetArticle (Python)
- DynamoDB table `Articles` with partition key `articleId` and GSI on `authorEmail`
- Optional CloudFront distribution for CDN (recommended for State++ level)
- IAM roles with least-privilege policies for Lambda and deployment automation

## Prerequisites
- Review [aws-worldskills-notes/01_lambda](aws-worldskills-notes/01_lambda)
- Review [aws-worldskills-notes/02_api_gateway](aws-worldskills-notes/02_api_gateway)
- Review [aws-worldskills-notes/04_s3](aws-worldskills-notes/04_s3)

## High-Level Diagram
```mermaid
graph TD
    BROWSER[Browser] -->|HTTPS| CF[CloudFront (optional)]
    CF --> S3[S3 Static Site]
    BROWSER -->|HTTPS /api/*| API[API Gateway]
    API --> L1[Lambda ListArticles]
    API --> L2[Lambda CreateArticle]
    API --> L3[Lambda GetArticle]
    L1 --> DDB[DynamoDB Articles]
    L2 --> DDB
    L3 --> DDB
```

## Time-Boxed Implementation (90 minutes)
1. **Create DynamoDB table with GSI (10 min)**
   - Table name `Articles`, partition key `articleId` (String), on-demand capacity, point-in-time recovery off.
   - Add GSI `AuthorEmailIndex` with partition key `authorEmail` (String), projection: ALL.
2. **Create IAM execution role for Lambda (8 min)**
   - Trust policy: lambda.amazonaws.com.
   - Permissions: `AmazonDynamoDBFullAccess` (scope down to table if time), `CloudWatchLogsFullAccess`.
3. **Deploy three Lambda functions (20 min)**
   - Runtime: Python 3.11, memory 256 MB, timeout 10s.
   - Environment: `TABLE_NAME=Articles`, `GSI_NAME=AuthorEmailIndex`.
   - Sample handler sketch:
```python
import boto3, json, os, uuid
from decimal import Decimal

ddb = boto3.resource("dynamodb")
table = ddb.Table(os.environ["TABLE_NAME"])

def create_article(event, context):
    body = json.loads(event.get("body", "{}"))
    item = {
        "articleId": str(uuid.uuid4()),
        "title": body.get("title", "Untitled"),
        "content": body.get("content", ""),
        "authorEmail": body.get("authorEmail", "unknown"),
    }
    table.put_item(Item=item)
    return {"statusCode": 201, "headers": {"Content-Type": "application/json"}, "body": json.dumps(item)}
```
   - Repeat pattern for list/get; return CORS headers: `Access-Control-Allow-Origin: *` and `Access-Control-Allow-Headers: Content-Type`.
4. **Create API Gateway REST API with 3 methods + CORS (25 min)**
   - Resources: `/articles` (GET, POST) and `/articles/{id}` (GET).
   - Integration: Lambda proxy for each function.
   - Enable CORS on both resources; deploy to stage `prod`; note invoke URL.
5. **Create S3 bucket and upload static site (15 min)**
   - Unique bucket name, region-aligned, Block Public Access: off for this lab only.
   - Enable static website hosting; index document `index.html`.
   - Upload minimal frontend that calls API Gateway endpoints.
6. **Configure bucket policy for public read (5 min)**
   - Policy allowing `s3:GetObject` on `arn:aws:s3:::<bucket>/*` for `Principal: *`.
7. **Test end-to-end (7 min)**
   - Visit S3 website endpoint, create article, list articles, fetch by id.
   - Confirm CloudWatch Logs show Lambda invocations.

## Verification Checklist
- S3 website endpoint loads the static site.
- `GET /articles` returns list; `POST /articles` stores items in DynamoDB; `GET /articles/{id}` returns the created item.
- CORS headers are present in API responses.
- DynamoDB table shows new items; GSI present.
- CloudWatch Logs contain Lambda execution entries without errors.
- (Optional) CloudFront domain serves the site via HTTPS.

## Common Mistakes
- Static website hosting not enabled on S3 (403 errors).
- CORS preflight not configured on API Gateway, causing browser blocks.
- Forgot to deploy API changes to stage `prod`.
- Lambda timeout/memory too low for cold-start + DynamoDB call.
- Missing execution role permissions for DynamoDB or CloudWatch Logs.

## Cost Breakdown (Free Tier aligned)
- S3: $0.023/GB storage + $0.09/GB transfer (Free Tier: 5GB storage, 15GB transfer).
- Lambda: $0.20 per 1M requests (Free Tier: 1M requests/month).
- API Gateway REST: ~$3.50 per 1M requests (Free Tier: 1M req for 12 months).
- DynamoDB: On-demand pricing (Free Tier: 25GB storage, 25 WCUs/RCUs equivalent).
- Expected total within Free Tier for lab traffic: ~$0.

## WorldSkills Marking Criteria (weights)
- Architecture design: 20%
- Implementation accuracy: 30%
- Security best practices (least privilege, CORS, public access scope): 20%
- Cost optimization (Free Tier alignment): 15%
- Documentation and verification evidence: 15%

## Time Management Tips
- Parallelize: create DynamoDB + IAM role while zipping Lambda code.
- Critical path: API Gateway deployment → static site testing; do this early.
- Checkpoint every 20 minutes: DynamoDB+IAM (T+20), Lambdas (T+40), API (T+65), S3 site (T+80).

## Exam Simulation Mode (2-hour cap)
- Must-have: DynamoDB table, 3 Lambdas, API Gateway methods with CORS, S3 static site, public bucket policy.
- Nice-to-have: CloudFront CDN, IAM policy scope-down, GSI queries from UI.
- If behind schedule: skip CloudFront; use one Lambda for GET/POST via routing; minimal frontend without styling.

## Cleanup
- Delete S3 bucket (objects first), API Gateway, Lambda functions, DynamoDB table, CloudWatch log groups, optional CloudFront distribution and ACM certs.

## Related Scenarios
- Alternative with EC2 and ALB: see [aws-worldskills-notes/09_full_server_scenarios/scenario_2_ec2_web_hosting.md](aws-worldskills-notes/09_full_server_scenarios/scenario_2_ec2_web_hosting.md).
- Security-focused VPC pattern: see [aws-worldskills-notes/09_full_server_scenarios/scenario_3_secure_private_vpc.md](aws-worldskills-notes/09_full_server_scenarios/scenario_3_secure_private_vpc.md).
- Cost-first serverless with monitoring: see [aws-worldskills-notes/09_full_server_scenarios/scenario_4_cost_optimized_architecture.md](aws-worldskills-notes/09_full_server_scenarios/scenario_4_cost_optimized_architecture.md).

## Next Steps
- Continue to IndiaSkills prep materials (placeholder): [aws-worldskills-notes/10_indskills](aws-worldskills-notes/10_indskills).
