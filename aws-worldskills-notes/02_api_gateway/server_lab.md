# Complete REST API Lab - Task Management System

## Lab Overview

In this comprehensive lab, you will build a complete task management REST API using AWS services:

**What You'll Build**:
- Task Management API with full CRUD operations
- DynamoDB tables with Global Secondary Index
- Lambda functions with router pattern
- HTTP API (not REST API for lower cost)
- IAM roles and policies
- CloudWatch monitoring
- Complete testing suite

**Time Estimate**: 60-90 minutes  
**Cost**: Free tier eligible (no charges expected)  
**Prerequisites**: AWS account, AWS CLI configured, basic Python knowledge

**Architecture Diagram**:
```
Client (curl/Postman)
    ↓
HTTP API Gateway
    ↓
Lambda (TaskHandler)
    ↓
DynamoDB Tables
    ├── Tasks (Primary: taskId)
    └── TaskIndex (GSI: userId)
```

## Step 1: Create DynamoDB Tables (10 minutes)

### Create Tasks Table

**AWS Console Steps**:
1. Go to DynamoDB dashboard
2. Click "Create table"
3. Configure:
   - **Table name**: `Tasks`
   - **Partition key**: `taskId` (String)
   - **Billing mode**: On-demand
   - Click "Create table"

**AWS CLI Command**:
```bash
aws dynamodb create-table \
    --table-name Tasks \
    --attribute-definitions \
        AttributeName=taskId,AttributeType=S \
        AttributeName=userId,AttributeType=S \
    --key-schema \
        AttributeName=taskId,KeyType=HASH \
    --billing-mode PAY_PER_REQUEST \
    --global-secondary-indexes '[{
        "IndexName": "userId-index",
        "KeySchema": [{"AttributeName": "userId", "KeyType": "HASH"}],
        "Projection": {"ProjectionType": "ALL"}
    }]' \
    --region us-east-1
```

### Verify Table Creation

**AWS Console**:
1. In DynamoDB dashboard, see "Tasks" table
2. View "Overview" tab
3. Confirm status: "Active"

**AWS CLI**:
```bash
aws dynamodb describe-table --table-name Tasks --region us-east-1
```

**Expected Output**:
```
{
  "Table": {
    "TableName": "Tasks",
    "TableStatus": "ACTIVE",
    "ItemCount": 0,
    "BillingModeSummary": {
      "BillingMode": "PAY_PER_REQUEST"
    }
  }
}
```

## Step 2: Create IAM Role (10 minutes)

### Create Execution Role

**AWS Console Steps**:
1. Go to IAM console
2. Click "Roles" → "Create role"
3. Select "Lambda" as trusted entity
4. Attach policies:
   - `AWSLambdaBasicExecutionRole` (for CloudWatch logs)
5. Create inline policy:
   - **Name**: `TasksDynamoDBAccess`
   - **JSON Policy**:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query",
        "dynamodb:Scan"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:123456789012:table/Tasks",
        "arn:aws:dynamodb:us-east-1:123456789012:table/Tasks/index/*"
      ]
    },
    {
      "Effect": "Allow",
      "Action": ["logs:CreateLogGroup", "logs:CreateLogStream", "logs:PutLogEvents"],
      "Resource": "arn:aws:logs:us-east-1:123456789012:log-group:/aws/lambda/*"
    }
  ]
}
```

**Replace `123456789012` with your actual AWS account ID**:
```bash
aws sts get-caller-identity --query Account
```

**AWS CLI Commands**:
```bash
# Create role
aws iam create-role \
    --role-name TasksLambdaRole \
    --assume-role-policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Principal": {"Service": "lambda.amazonaws.com"},
        "Action": "sts:AssumeRole"
      }]
    }'

# Attach basic execution policy
aws iam attach-role-policy \
    --role-name TasksLambdaRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Create inline policy (replace 123456789012 with your account ID)
aws iam put-role-policy \
    --role-name TasksLambdaRole \
    --policy-name TasksDynamoDBAccess \
    --policy-document '{
      "Version": "2012-10-17",
      "Statement": [{
        "Effect": "Allow",
        "Action": [
          "dynamodb:GetItem",
          "dynamodb:PutItem",
          "dynamodb:UpdateItem",
          "dynamodb:DeleteItem",
          "dynamodb:Query",
          "dynamodb:Scan"
        ],
        "Resource": [
          "arn:aws:dynamodb:us-east-1:123456789012:table/Tasks",
          "arn:aws:dynamodb:us-east-1:123456789012:table/Tasks/index/*"
        ]
      }]
    }'
```

### Get Role ARN

```bash
ROLE_ARN=$(aws iam get-role --role-name TasksLambdaRole --query 'Role.Arn' --output text)
echo $ROLE_ARN
```

**Save this for next step**: You'll need it when creating Lambda.

## Step 3: Create Lambda Function (15 minutes)

### Create Function

**AWS Console Steps**:
1. Go to Lambda dashboard
2. Click "Create function"
3. Configure:
   - **Function name**: `TaskHandler`
   - **Runtime**: Python 3.11
   - **Execution role**: Select `TasksLambdaRole`
4. Click "Create function"

### Add Code

Create `handler.py`:

```python
import json
import boto3
import uuid
from datetime import datetime

dynamodb = boto3.resource('dynamodb', region_name='us-east-1')
table = dynamodb.Table('Tasks')

def lambda_handler(event, context):
    """Main router for Task Management API"""
    
    method = event.get('httpMethod', 'GET')
    path = event.get('rawPath', '/')
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type'
    }
    
    try:
        # Handle OPTIONS preflight
        if method == 'OPTIONS':
            return {
                'statusCode': 200,
                'headers': headers,
                'body': ''
            }
        
        # Route based on path and method
        if path == '/tasks' or path.endswith('/tasks'):
            if method == 'GET':
                return get_tasks(event, headers)
            elif method == 'POST':
                return create_task(event, headers)
        
        elif '/tasks/' in path:
            if method == 'GET':
                return get_task(event, headers)
            elif method == 'PUT':
                return update_task(event, headers)
            elif method == 'DELETE':
                return delete_task(event, headers)
        
        return error_response(404, 'Not found', headers)
    
    except Exception as e:
        print(f"Error: {str(e)}")
        import traceback
        traceback.print_exc()
        return error_response(500, 'Server error', headers)

def get_tasks(event, headers):
    """GET /tasks - List all tasks with pagination"""
    
    query_params = event.get('queryStringParameters', {}) or {}
    user_id = query_params.get('userId')
    limit = min(int(query_params.get('limit', 10)), 100)
    
    try:
        if user_id:
            # Query by userId using GSI
            response = table.query(
                IndexName='userId-index',
                KeyConditionExpression='userId = :userId',
                ExpressionAttributeValues={':userId': user_id},
                Limit=limit
            )
        else:
            # Scan all tasks
            response = table.scan(Limit=limit)
        
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps({
                'items': response.get('Items', []),
                'count': response['Count'],
                'scannedCount': response.get('ScannedCount', 0)
            })
        }
    except Exception as e:
        print(f"Error in get_tasks: {str(e)}")
        return error_response(500, 'Failed to retrieve tasks', headers)

def get_task(event, headers):
    """GET /tasks/{taskId} - Get single task"""
    
    task_id = event.get('pathParameters', {}).get('taskId')
    
    if not task_id:
        return error_response(400, 'taskId required', headers)
    
    try:
        response = table.get_item(Key={'taskId': task_id})
        
        if 'Item' not in response:
            return error_response(404, 'Task not found', headers)
        
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps(response['Item'])
        }
    except Exception as e:
        print(f"Error in get_task: {str(e)}")
        return error_response(500, 'Failed to retrieve task', headers)

def create_task(event, headers):
    """POST /tasks - Create new task"""
    
    try:
        body = json.loads(event.get('body', '{}'))
    except json.JSONDecodeError:
        return error_response(400, 'Invalid JSON', headers)
    
    # Validate inputs
    title = body.get('title', '').strip()
    user_id = body.get('userId', '').strip()
    description = body.get('description', '').strip()
    due_date = body.get('dueDate', '')
    
    if not title:
        return error_response(400, 'Title required', headers)
    
    if not user_id:
        return error_response(400, 'userId required', headers)
    
    if len(title) > 200:
        return error_response(400, 'Title too long (max 200)', headers)
    
    try:
        task_id = str(uuid.uuid4())
        
        task_item = {
            'taskId': task_id,
            'userId': user_id,
            'title': title,
            'description': description,
            'status': 'open',
            'dueDate': due_date,
            'createdAt': datetime.now().isoformat(),
            'updatedAt': datetime.now().isoformat()
        }
        
        table.put_item(Item=task_item)
        
        return {
            'statusCode': 201,
            'headers': {**headers, 'Location': f'/tasks/{task_id}'},
            'body': json.dumps(task_item)
        }
    except Exception as e:
        print(f"Error in create_task: {str(e)}")
        return error_response(500, 'Failed to create task', headers)

def update_task(event, headers):
    """PUT /tasks/{taskId} - Update task"""
    
    task_id = event.get('pathParameters', {}).get('taskId')
    
    if not task_id:
        return error_response(400, 'taskId required', headers)
    
    try:
        body = json.loads(event.get('body', '{}'))
    except json.JSONDecodeError:
        return error_response(400, 'Invalid JSON', headers)
    
    # Check if task exists
    response = table.get_item(Key={'taskId': task_id})
    if 'Item' not in response:
        return error_response(404, 'Task not found', headers)
    
    # Build update expression
    updates = []
    expr_values = {}
    expr_names = {}
    
    if 'title' in body:
        title = body['title'].strip()
        if len(title) > 200:
            return error_response(400, 'Title too long', headers)
        updates.append('#title = :title')
        expr_values[':title'] = title
        expr_names['#title'] = 'title'
    
    if 'description' in body:
        updates.append('description = :desc')
        expr_values[':desc'] = body['description']
    
    if 'status' in body:
        if body['status'] not in ['open', 'in-progress', 'completed', 'cancelled']:
            return error_response(400, 'Invalid status', headers)
        updates.append('#status = :status')
        expr_values[':status'] = body['status']
        expr_names['#status'] = 'status'
    
    if 'dueDate' in body:
        updates.append('dueDate = :due')
        expr_values[':due'] = body['dueDate']
    
    if not updates:
        return error_response(400, 'No fields to update', headers)
    
    updates.append('updatedAt = :updated')
    expr_values[':updated'] = datetime.now().isoformat()
    
    try:
        table.update_item(
            Key={'taskId': task_id},
            UpdateExpression='SET ' + ', '.join(updates),
            ExpressionAttributeNames=expr_names if expr_names else None,
            ExpressionAttributeValues=expr_values
        )
        
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps({'message': 'Task updated', 'taskId': task_id})
        }
    except Exception as e:
        print(f"Error in update_task: {str(e)}")
        return error_response(500, 'Failed to update task', headers)

def delete_task(event, headers):
    """DELETE /tasks/{taskId} - Delete task"""
    
    task_id = event.get('pathParameters', {}).get('taskId')
    
    if not task_id:
        return error_response(400, 'taskId required', headers)
    
    try:
        response = table.get_item(Key={'taskId': task_id})
        if 'Item' not in response:
            return error_response(404, 'Task not found', headers)
        
        table.delete_item(Key={'taskId': task_id})
        
        return {
            'statusCode': 204,
            'headers': headers,
            'body': ''
        }
    except Exception as e:
        print(f"Error in delete_task: {str(e)}")
        return error_response(500, 'Failed to delete task', headers)

def error_response(status_code, message, headers):
    """Helper for error responses"""
    return {
        'statusCode': status_code,
        'headers': headers,
        'body': json.dumps({'error': message})
    }
```

**AWS Console Steps** (paste code):
1. In Lambda code editor, replace all code with above
2. Click "Deploy"
3. Verify "Deployment successful"

## Step 4: Create HTTP API (15 minutes)

### Create API Gateway HTTP API

**AWS Console Steps**:
1. Go to API Gateway dashboard
2. Click "Create API"
3. Choose "HTTP API"
4. Click "Build"
5. Configure:
   - **API name**: `TaskAPI`
   - **Disable CORS** (we handle it in Lambda)
   - Click "Next"

### Create Routes

**Create GET /tasks**:
1. Method: GET
2. Resource: /tasks
3. Integration: Lambda
4. Function: TaskHandler
5. Click "Create"

**Create GET /tasks/{taskId}**:
1. Method: GET
2. Resource: /tasks/{taskId}
3. Integration: Lambda
4. Function: TaskHandler
5. Click "Create"

**Create POST /tasks**:
1. Method: POST
2. Resource: /tasks
3. Integration: Lambda
4. Function: TaskHandler
5. Click "Create"

**Create PUT /tasks/{taskId}**:
1. Method: PUT
2. Resource: /tasks/{taskId}
3. Integration: Lambda
4. Function: TaskHandler
5. Click "Create"

**Create DELETE /tasks/{taskId}**:
1. Method: DELETE
2. Resource: /tasks/{taskId}
3. Integration: Lambda
4. Function: TaskHandler
5. Click "Create"

**Create OPTIONS /tasks**:
1. Method: OPTIONS
2. Resource: /tasks
3. Integration: Lambda
4. Function: TaskHandler
5. Click "Create"

### Deploy API

1. Click "Deployments" → "Deploy"
2. Stage: `prod`
3. Click "Deploy"
4. Get API endpoint: `https://xxxxx.execute-api.us-east-1.amazonaws.com`

**Save this endpoint URL**.

### AWS CLI Alternative

```bash
# Create API
API_ID=$(aws apigatewayv2 create-api \
    --name TaskAPI \
    --protocol-type HTTP \
    --target arn:aws:lambda:us-east-1:123456789012:function:TaskHandler \
    --region us-east-1 \
    --query 'ApiId' --output text)

echo "API ID: $API_ID"

# Get invoke URL
aws apigatewayv2 get-api \
    --api-id $API_ID \
    --region us-east-1 \
    --query 'ApiEndpoint' --output text
```

## Step 5: Testing All Operations (20 minutes)

### Test Environment Variables

```bash
# Set API endpoint
export API_URL="https://xxxxx.execute-api.us-east-1.amazonaws.com"

# Verify connectivity
curl -X OPTIONS $API_URL/tasks -v
```

### Test 1: Create Task (POST)

```bash
curl -X POST $API_URL/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Complete AWS Project",
    "description": "Finish the task management API",
    "userId": "user-123",
    "dueDate": "2025-02-01"
  }'
```

**Expected Response (201)**:
```json
{
  "taskId": "550e8400-e29b-41d4-a716-446655440000",
  "userId": "user-123",
  "title": "Complete AWS Project",
  "description": "Finish the task management API",
  "status": "open",
  "dueDate": "2025-02-01",
  "createdAt": "2025-01-01T10:00:00.000000",
  "updatedAt": "2025-01-01T10:00:00.000000"
}
```

### Test 2: Create Another Task

```bash
curl -X POST $API_URL/tasks \
  -H "Content-Type: application/json" \
  -d '{
    "title": "Review documentation",
    "description": "Check API docs",
    "userId": "user-123",
    "dueDate": "2025-01-15"
  }'
```

### Test 3: Get Task (GET by ID)

```bash
# Replace with taskId from previous response
TASK_ID="550e8400-e29b-41d4-a716-446655440000"

curl -X GET $API_URL/tasks/$TASK_ID
```

**Expected Response (200)**: Full task details

### Test 4: List Tasks (GET all)

```bash
curl -X GET "$API_URL/tasks?limit=10"
```

**Expected Response (200)**:
```json
{
  "items": [...],
  "count": 2,
  "scannedCount": 2
}
```

### Test 5: List Tasks by User

```bash
curl -X GET "$API_URL/tasks?userId=user-123&limit=10"
```

### Test 6: Update Task (PUT)

```bash
curl -X PUT $API_URL/tasks/$TASK_ID \
  -H "Content-Type: application/json" \
  -d '{
    "status": "in-progress",
    "description": "Currently working on API"
  }'
```

**Expected Response (200)**: Update confirmation

### Test 7: Delete Task (DELETE)

```bash
curl -X DELETE $API_URL/tasks/$TASK_ID
```

**Expected Response (204)**: No content (success)

### Test 8: Verify Deletion

```bash
# Should return 404 Not Found
curl -X GET $API_URL/tasks/$TASK_ID
```

## Testing Summary Table

| Test | Method | URL | Status | Notes |
|------|--------|-----|--------|-------|
| Create task | POST | /tasks | 201 | Save taskId |
| Get task | GET | /tasks/{taskId} | 200 | Full details |
| List all | GET | /tasks | 200 | Pagination |
| List by user | GET | /tasks?userId=X | 200 | GSI query |
| Update task | PUT | /tasks/{taskId} | 200 | Partial update |
| Delete task | DELETE | /tasks/{taskId} | 204 | Verification |
| Get deleted | GET | /tasks/{taskId} | 404 | Confirm deletion |

## Step 6: CloudWatch Monitoring (10 minutes)

### View Lambda Logs

**AWS Console**:
1. Go to Lambda dashboard
2. Select `TaskHandler` function
3. Click "Monitor" tab
4. Click "View logs in CloudWatch"
5. Select latest log stream
6. View function execution logs

### Query Logs with AWS CLI

```bash
# Get latest log streams
aws logs describe-log-streams \
    --log-group-name /aws/lambda/TaskHandler \
    --order-by LastEventTime \
    --descending \
    --max-items 5

# Get log events from latest stream
STREAM=$(aws logs describe-log-streams \
    --log-group-name /aws/lambda/TaskHandler \
    --order-by LastEventTime \
    --descending \
    --max-items 1 \
    --query 'logStreams[0].logStreamName' \
    --output text)

aws logs get-log-events \
    --log-group-name /aws/lambda/TaskHandler \
    --log-stream-name $STREAM
```

### Monitor API Gateway Metrics

1. Go to API Gateway dashboard
2. Select TaskAPI
3. Click "Stage: prod"
4. View metrics:
   - Count
   - 4xx errors
   - 5xx errors
   - Latency

### Create CloudWatch Alarm

```bash
aws cloudwatch put-metric-alarm \
    --alarm-name TaskAPI-5xx-errors \
    --alarm-description "Alert when API returns 5xx errors" \
    --metric-name 5XXError \
    --namespace AWS/ApiGateway \
    --statistic Sum \
    --period 60 \
    --threshold 5 \
    --comparison-operator GreaterThanThreshold \
    --evaluation-periods 1
```

## Cost Analysis

**Expected Monthly Costs (Free Tier)**:

| Service | Free Tier | Expected Usage | Cost |
|---------|-----------|-----------------|------|
| Lambda | 1M invocations, 400K GB-s | ~50 tasks/day = 1.5K invocations | $0 |
| DynamoDB (on-demand) | 25 GB stored, 25 write units, 100 read units | < 1 GB storage | $0 |
| HTTP API | First 1M requests | ~1.5K requests | $0 |
| CloudWatch Logs | 5 GB | ~50 MB | $0 |
| **Total** | | | **$0 (Free Tier)** |

**Calculation**:
- Create task: 1 write unit
- Get task: 1 read unit
- List tasks: 1 read unit
- Update task: 1 write unit
- Delete task: 1 write unit

50 tasks/day × 5 operations avg = 250 operations/day × 30 = 7,500 operations/month

## Step 7: Cleanup (5 minutes)

### Delete API

```bash
aws apigatewayv2 delete-api --api-id $API_ID --region us-east-1
```

### Delete Lambda Function

```bash
aws lambda delete-function --function-name TaskHandler --region us-east-1
```

### Delete Lambda Role

```bash
# Detach policies
aws iam detach-role-policy \
    --role-name TasksLambdaRole \
    --policy-arn arn:aws:iam::aws:policy/service-role/AWSLambdaBasicExecutionRole

# Delete inline policy
aws iam delete-role-policy \
    --role-name TasksLambdaRole \
    --policy-name TasksDynamoDBAccess

# Delete role
aws iam delete-role --role-name TasksLambdaRole
```

### Delete DynamoDB Table

```bash
aws dynamodb delete-table --table-name Tasks --region us-east-1
```

### Wait for Deletion

```bash
aws dynamodb wait table-not-exists --table-name Tasks
echo "Table deleted successfully"
```

## Common Issues and Solutions

**Issue 1**: "Access Denied" error
- **Cause**: Lambda role doesn't have DynamoDB permissions
- **Solution**: Check IAM policy has correct DynamoDB actions and resource ARNs

**Issue 2**: Task not created or found 404
- **Cause**: DynamoDB table not active or wrong table name
- **Solution**: Verify table name is exactly "Tasks"

**Issue 3**: Lambda timeout
- **Cause**: Default timeout too short
- **Solution**: Increase Lambda timeout to 30 seconds in function configuration

**Issue 4**: "Invalid JSON" error
- **Cause**: Request body not properly formatted
- **Solution**: Ensure Content-Type is "application/json" and valid JSON syntax

**Issue 5**: CORS errors in browser
- **Cause**: CORS headers not returned
- **Solution**: Verify Lambda returns 'Access-Control-Allow-Origin': '*'

## Competition Tips

1. **Use HTTP API instead of REST API** for cost optimization (same functionality, lower price)
2. **Use on-demand billing** for DynamoDB to avoid unexpected charges
3. **Test all operations** before submitting (GET, POST, PUT, DELETE)
4. **Include error handling** in Lambda (400, 404, 500 responses)
5. **Use CloudWatch logs** for debugging (check logs if tests fail)
6. **Verify IAM permissions** before deployment (check policy ARNs)
7. **Test with curl** to verify API works independently
8. **Use Global Secondary Index** for efficient queries by userId

## Advanced Variations

### Add Timestamps to Track Updates

In Lambda, automatically add:
```python
'createdAt': datetime.now().isoformat(),
'updatedAt': datetime.now().isoformat(),
'lastModifiedBy': 'system'
```

### Add Soft Delete

Instead of deleting:
```python
'status': 'deleted',
'deletedAt': datetime.now().isoformat()
```

### Add Batch Operations

Handle multiple creates:
```python
def create_batch_tasks(event, headers):
    """POST /tasks/batch"""
    body = json.loads(event['body'])
    tasks = body.get('tasks', [])
    
    with table.batch_writer() as batch:
        for task in tasks:
            batch.put_item(Item=task)
```

### Add Task Search

```bash
curl "$API_URL/tasks?search=API&limit=10"
```

## Verification Checklist

- [ ] DynamoDB table "Tasks" created and active
- [ ] IAM role "TasksLambdaRole" created with correct permissions
- [ ] Lambda function "TaskHandler" created with correct code
- [ ] HTTP API "TaskAPI" created with all routes
- [ ] API deployed to "prod" stage
- [ ] API endpoint URL obtained and saved
- [ ] Test 1: POST /tasks creates task (201)
- [ ] Test 2: GET /tasks/{taskId} retrieves task (200)
- [ ] Test 3: GET /tasks lists tasks (200)
- [ ] Test 4: PUT /tasks/{taskId} updates task (200)
- [ ] Test 5: DELETE /tasks/{taskId} deletes task (204)
- [ ] CloudWatch logs viewable
- [ ] All CRUD operations work with curl
- [ ] Cleanup commands save and ready

## Time Breakdown

| Step | Time | Notes |
|------|------|-------|
| 1. DynamoDB | 10 min | Create table with GSI |
| 2. IAM Role | 10 min | Create role and policies |
| 3. Lambda | 15 min | Write and deploy code |
| 4. HTTP API | 15 min | Create routes and deploy |
| 5. Testing | 20 min | Test all CRUD operations |
| 6. Monitoring | 10 min | Check logs and metrics |
| 7. Cleanup | 5 min | Delete resources |
| **Total** | **85 min** | End-to-end deployment |

## Summary

You've successfully built a complete task management REST API with:
- ✓ Full CRUD operations
- ✓ DynamoDB backend
- ✓ Lambda integration
- ✓ HTTP API exposure
- ✓ CloudWatch monitoring
- ✓ IAM security
- ✓ Production-ready code

**Next Steps**:
- Deploy to production
- Add authentication (API keys, Cognito)
- Add rate limiting
- Add input validation schemas
- Implement soft deletes
- Add batch operations
- Set up CI/CD pipeline
