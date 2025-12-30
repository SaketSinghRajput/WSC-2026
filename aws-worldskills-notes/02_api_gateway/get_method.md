# GET Method - Retrieving Resources with API Gateway

## GET Method Overview

The GET HTTP method retrieves resources from the server. GET requests are:
- **Safe**: Do not modify server state
- **Idempotent**: Multiple identical requests return same result
- **Cacheable**: Responses can be cached by browsers and CDNs
- **No Request Body**: Data passed via URL path and query string parameters

**Use Cases**:
- Fetch user profile by ID
- List all items with filtering
- Retrieve product details
- Search operations
- Download files
- Stream data

## GET Request Characteristics

```
GET /users/123 HTTP/1.1
Host: api.example.com
Content-Type: application/json
```

**Key Points**:
- No request body (technically allowed but not recommended)
- Resource identifier in URL path
- Query parameters for filtering/pagination
- Response contains data in JSON body

## API Gateway GET Configuration

### Creating Resource and Method

**Step 1: Create Resource**
1. Navigate to API Gateway dashboard
2. Select your API
3. Click resource (or create new: Actions → Create Resource)
4. Resource name: `users`, Path: `/users`
5. Create child resource: `{userId}`, Path: `/{userId}`

**Step 2: Create GET Method**
1. Select `/{userId}` resource
2. Click "Actions" → "Create Method" → "GET"
3. Integration type: "Lambda Function" (or "Lambda Proxy Function" for proxy integration)
4. Select Lambda function: `UserHandler`
5. Click "Save"

**Step 3: Enable CORS** (if API Gateway + Browser)
1. Select resource
2. Actions → Enable CORS
3. Select "GET" method
4. Click "Enable CORS"
5. Deploy API

### Lambda Proxy Integration Event Structure

API Gateway sends complete HTTP request as event object:

```json
{
  "httpMethod": "GET",
  "path": "/users/123",
  "pathParameters": {
    "userId": "123"
  },
  "queryStringParameters": {
    "includeOrders": "true"
  },
  "headers": {
    "Host": "api.example.com",
    "Content-Type": "application/json",
    "Authorization": "Bearer token123"
  },
  "body": null,
  "isBase64Encoded": false,
  "requestContext": {
    "requestId": "request-id-123",
    "stage": "prod",
    "identity": {
      "sourceIp": "192.0.2.1"
    }
  }
}
```

## Lambda Handler for GET

Basic structure for handling GET requests:

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """
    GET request handler
    Retrieves user by userId from path parameter
    """
    try:
        # Extract path parameters
        path_params = event.get('pathParameters', {})
        user_id = path_params.get('userId')
        
        # Validate input
        if not user_id:
            return {
                'statusCode': 400,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({'error': 'userId parameter required'})
            }
        
        # Retrieve from DynamoDB
        response = table.get_item(Key={'userId': user_id})
        
        # Check if user exists
        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({'error': f'User {user_id} not found'})
            }
        
        # Return success response
        return {
            'statusCode': 200,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps(response['Item'])
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Internal server error'})
        }
```

## Path Parameters

Path parameters identify specific resources in the URL:

### Path Parameter Syntax

**URL Pattern**: `/users/{userId}`
**Example URL**: `/users/123`
**Extracted Parameter**: `userId = "123"`

### Multiple Path Parameters

```
URL Pattern: /users/{userId}/orders/{orderId}
Example URL: /users/123/orders/456
Extracted: userId = "123", orderId = "456"
```

### Lambda Extraction

```python
def lambda_handler(event, context):
    path_params = event.get('pathParameters', {})
    user_id = path_params.get('userId')      # "123"
    order_id = path_params.get('orderId')    # "456"
```

### Validation

```python
def validate_user_id(user_id):
    """Validate userId format"""
    if not user_id:
        return False
    if not user_id.isalnum():  # Only alphanumeric
        return False
    if len(user_id) > 50:  # Max length
        return False
    return True

def lambda_handler(event, context):
    user_id = event['pathParameters'].get('userId')
    
    if not validate_user_id(user_id):
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Invalid userId format'})
        }
```

## Query String Parameters

Query parameters filter or customize responses:

### Query String Syntax

**URL Pattern**: `/users?status=active&limit=10`
**Parsed Parameters**: 
- `status = "active"`
- `limit = "10"`

### Lambda Extraction

```python
def lambda_handler(event, context):
    query_params = event.get('queryStringParameters', {})
    status = query_params.get('status', 'all')  # Default to 'all'
    limit = int(query_params.get('limit', '20'))  # Default to 20
```

### Common Query Parameter Patterns

**Pagination**:
```
GET /users?limit=20&offset=0
GET /products?page=2&pageSize=50
```

**Filtering**:
```
GET /users?status=active
GET /products?category=electronics&inStock=true
```

**Sorting**:
```
GET /users?sortBy=name&sortOrder=asc
```

**Full-Text Search**:
```
GET /users?search=john
```

## Response Format

GET responses typically return JSON:

```python
{
    'statusCode': 200,
    'headers': {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'  # CORS
    },
    'body': json.dumps({
        'userId': '123',
        'name': 'John Doe',
        'email': 'john@example.com',
        'createdAt': '2025-01-01T10:00:00Z'
    })
}
```

### Response Status Codes

**200 OK**: Successful retrieval, data included in body
**204 No Content**: Successful retrieval, no data to return
**400 Bad Request**: Invalid parameters (missing userId, invalid format)
**404 Not Found**: Resource doesn't exist
**500 Internal Server Error**: Server error during processing

## Error Handling

### Missing Required Parameter

```python
if not user_id:
    return {
        'statusCode': 400,
        'body': json.dumps({'error': 'userId is required'})
    }
```

### Resource Not Found

```python
response = table.get_item(Key={'userId': user_id})

if 'Item' not in response:
    return {
        'statusCode': 404,
        'body': json.dumps({'error': f'User {user_id} not found'})
    }
```

### Server Error

```python
try:
    # Process request
except Exception as e:
    print(f"Error: {str(e)}")
    return {
        'statusCode': 500,
        'body': json.dumps({'error': 'Internal server error'})
    }
```

## CORS Configuration

CORS (Cross-Origin Resource Sharing) enables browser-based clients to call your API:

### Enabling CORS in API Gateway

1. Select resource
2. Actions → Enable CORS and replace existing CORS headers
3. Select "GET" method
4. Click "Enable CORS and replace CORS headers"
5. Confirm and deploy API

### CORS Headers in Lambda

```python
def get_cors_headers():
    return {
        'Access-Control-Allow-Origin': '*',  # Allow all origins (or specific domain)
        'Access-Control-Allow-Methods': 'GET,OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type,Authorization'
    }

def lambda_handler(event, context):
    headers = get_cors_headers()
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps(data)
    }
```

## Complete Example 1: Get User by ID

**Scenario**: Retrieve user profile from DynamoDB by userId

**API Gateway Configuration**:
- Resource: `/users/{userId}`
- Method: GET
- Integration: Lambda Proxy

**Lambda Code**:

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """Get user by ID from DynamoDB"""
    
    # CORS headers
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    }
    
    try:
        # Extract userId from path
        path_params = event.get('pathParameters', {})
        user_id = path_params.get('userId')
        
        # Validate
        if not user_id:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'userId required'})
            }
        
        # Get from DynamoDB
        response = table.get_item(Key={'userId': user_id})
        
        # Check if exists
        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': headers,
                'body': json.dumps({'error': 'User not found'})
            }
        
        # Return user
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps(response['Item'])
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({'error': 'Server error'})
        }
```

**IAM Permissions**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["dynamodb:GetItem"],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Users"
    }
  ]
}
```

**Test Event**:
```json
{
  "httpMethod": "GET",
  "path": "/users/123",
  "pathParameters": {"userId": "123"}
}
```

**Expected Response**:
```json
{
  "statusCode": 200,
  "body": {
    "userId": "123",
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

## Complete Example 2: List Users with Pagination

**Scenario**: List all users with limit and offset for pagination

**API Gateway Configuration**:
- Resource: `/users`
- Method: GET
- Integration: Lambda Proxy
- Query string parameters: `limit`, `offset`

**Lambda Code**:

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """List users with DynamoDB pagination"""
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    }
    
    try:
        # Extract query parameters
        query_params = event.get('queryStringParameters', {}) or {}
        limit = int(query_params.get('limit', '20'))
        next_token = query_params.get('nextToken')  # From previous response
        
        # Validate pagination parameters
        if limit < 1 or limit > 100:
            limit = 20
        
        # Build scan kwargs
        scan_kwargs = {'Limit': limit}
        
        # If nextToken provided, use it as ExclusiveStartKey
        if next_token:
            try:
                # Decode nextToken (base64 encoded LastEvaluatedKey)
                import base64
                exclusive_start_key = json.loads(base64.b64decode(next_token))
                scan_kwargs['ExclusiveStartKey'] = exclusive_start_key
            except Exception as e:
                print(f"Invalid nextToken: {str(e)}")
                return {
                    'statusCode': 400,
                    'headers': headers,
                    'body': json.dumps({'error': 'Invalid nextToken'})
                }
        
        # Scan DynamoDB table with pagination
        response = table.scan(**scan_kwargs)
        items = response.get('Items', [])
        
        # Prepare response with nextToken if more results exist
        result = {
            'users': items,
            'count': len(items),
            'limit': limit
        }
        
        # If LastEvaluatedKey exists, encode it as nextToken
        if 'LastEvaluatedKey' in response:
            import base64
            next_token_value = base64.b64encode(
                json.dumps(response['LastEvaluatedKey']).encode()
            ).decode()
            result['nextToken'] = next_token_value
            result['hasMore'] = True
        else:
            result['hasMore'] = False
        
        # Return paginated results
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps(result)
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({'error': 'Server error'})
        }
```

**Test curl Commands**:

```bash
# First request - get first page
curl -X GET "https://api.example.com/users?limit=10"

# Response will include nextToken if more results exist:
# {
#   "users": [...],
#   "count": 10,
#   "limit": 10,
#   "nextToken": "eyJ1c2VySWQiOiAiMTIzIn0=",
#   "hasMore": true
# }

# Second request - get next page using nextToken from previous response
curl -X GET "https://api.example.com/users?limit=10&nextToken=eyJ1c2VySWQiOiAiMTIzIn0="

# Continue until hasMore is false
```

**Pagination Client Example**:

```javascript
// JavaScript client pagination example
async function getAllUsers() {
  let allUsers = [];
  let nextToken = null;
  
  do {
    const url = nextToken 
      ? `https://api.example.com/users?limit=20&nextToken=${encodeURIComponent(nextToken)}`
      : `https://api.example.com/users?limit=20`;
    
    const response = await fetch(url);
    const data = await response.json();
    
    allUsers = allUsers.concat(data.users);
    nextToken = data.nextToken;
    
  } while (data.hasMore);
  
  return allUsers;
}
```

**Test curl Command**:
```bash
curl -X GET "https://api.example.com/users?limit=10&offset=0"
```

**Expected Response**:
```json
{
  "statusCode": 200,
  "body": {
    "users": [
      {"userId": "1", "name": "User One"},
      {"userId": "2", "name": "User Two"}
    ],
    "limit": 10,
    "offset": 0,
    "totalCount": 47
  }
}
```

## Complete Example 3: Search Users by Email

**Scenario**: Search users by email with query parameter

**API Gateway Configuration**:
- Resource: `/users/search`
- Method: GET
- Query parameters: `email`

**Prerequisites**:
- DynamoDB table `Users` must have a Global Secondary Index (GSI) named `email-index` with `email` as the partition key
- Create GSI in console: Table → Indexes → Create index → Index name: `email-index`, Partition key: `email`

**Lambda Code**:

```python
import json
import boto3
from boto3.dynamodb.conditions import Key

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """Search users by email using GSI"""
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    }
    
    try:
        # Get email query parameter
        query_params = event.get('queryStringParameters', {}) or {}
        email = query_params.get('email', '').strip()
        
        # Validate email provided
        if not email:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'email query parameter required'})
            }
        
        # Validate email format
        if '@' not in email:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'Invalid email format'})
            }
        
        # Query DynamoDB using email GSI
        # Option 1: Using Key helper (recommended)
        response = table.query(
            IndexName='email-index',
            KeyConditionExpression=Key('email').eq(email)
        )
        
        # Option 2: Using ExpressionAttributeNames (alternative)
        # response = table.query(
        #     IndexName='email-index',
        #     KeyConditionExpression='#email = :email',
        #     ExpressionAttributeNames={'#email': 'email'},
        #     ExpressionAttributeValues={':email': email}
        # )
        
        items = response.get('Items', [])
        
        if not items:
            return {
                'statusCode': 404,
                'headers': headers,
                'body': json.dumps({'error': 'No users found'})
            }
        
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps({'users': items})
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({'error': 'Server error'})
        }
```

**Test curl Command**:
```bash
curl -X GET "https://api.example.com/users/search?email=john@example.com"
```

## Request Validation in API Gateway

API Gateway can validate requests before invoking Lambda (REST API only):

### JSON Schema Validation

```json
{
  "type": "object",
  "properties": {
    "userId": {
      "type": "string"
    },
    "limit": {
      "type": "integer",
      "minimum": 1,
      "maximum": 100
    }
  },
  "required": ["userId"]
}
```

**Benefits**:
- Invalid requests rejected before Lambda invocation
- Reduces Lambda costs
- Improves performance

## Caching GET Responses

Cache frequently-requested data to reduce DynamoDB calls:

### Enable Caching in API Gateway

1. Select GET method
2. Method Execution → Method Response
3. Click method response (200)
4. Click integration response (200)
5. Enable response caching
6. Set TTL (0 to 3600 seconds)

### Cache Key Parameters

Control what identifies cached responses:

```
GET /users/123
GET /users/123?detail=true
GET /products?category=electronics&inStock=true
```

Each unique combination cached separately.

**Example**: Cache GET /products for 1 hour
- First request → DynamoDB query → Cache result
- Second request (same URL) → Return cached result (no DynamoDB call)
- After 1 hour → Cache expires, next request queries DynamoDB

## AWS Console Steps - Creating GET Method

### Step 1: Create Resource
1. Open API Gateway dashboard
2. Select API
3. Resources → Click `/` (root)
4. Actions → Create Resource
5. Resource name: `users`
6. Resource path: `/users`
7. Click "Create Resource"

### Step 2: Create Child Resource (Optional)
1. Select `/users` resource
2. Actions → Create Resource
3. Resource name: `userId`
4. Resource path: `{userId}`
5. Click "Create Resource"

### Step 3: Create GET Method
1. Select `/{userId}` resource
2. Actions → Create Method → GET
3. Integration type: Lambda Function
4. Lambda function: Select your function
5. Use Lambda Proxy Integration: ✓
6. Click "Save"

### Step 4: Enable CORS
1. Select `/users` resource
2. Actions → Enable CORS and replace existing CORS headers
3. Select "GET" method
4. Click "Enable CORS"
5. Click "Deploy API"

### Step 5: Deploy
1. Click "Actions" → "Deploy API"
2. Deployment stage: `prod` (or new)
3. Click "Deploy"

## AWS CLI Commands

### Create Resource

```bash
RESOURCE_ID=$(aws apigateway get-resources \
    --rest-api-id abc123 \
    --query 'items[0].id' \
    --output text)

aws apigateway create-resource \
    --rest-api-id abc123 \
    --parent-id $RESOURCE_ID \
    --path-part users
```

### Create GET Method

```bash
aws apigateway put-method \
    --rest-api-id abc123 \
    --resource-id resource-id \
    --http-method GET \
    --authorization-type NONE
```

### Add Lambda Integration

```bash
LAMBDA_ARN=$(aws lambda get-function \
    --function-name UserHandler \
    --query 'Configuration.FunctionArn' \
    --output text)

aws apigateway put-integration \
    --rest-api-id abc123 \
    --resource-id resource-id \
    --http-method GET \
    --type AWS_PROXY \
    --integration-http-method POST \
    --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/$LAMBDA_ARN/invocations
```

## Testing with curl

### Simple GET Request

```bash
curl -X GET https://your-api-id.execute-api.us-east-1.amazonaws.com/prod/users/123
```

### GET with Query Parameters

```bash
curl -X GET "https://your-api-id.execute-api.us-east-1.amazonaws.com/prod/users?limit=20&offset=0"
```

### GET with Headers

```bash
curl -X GET https://your-api-id.execute-api.us-east-1.amazonaws.com/prod/users/123 \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer token123"
```

### GET with Verbose Output

```bash
curl -v -X GET https://your-api-id.execute-api.us-east-1.amazonaws.com/prod/users/123
```

## Testing with Postman

1. **Create Request**
   - Method: GET
   - URL: `https://your-api-id.execute-api.us-east-1.amazonaws.com/prod/users/123`

2. **Add Query Parameters** (Params tab)
   - Key: `limit`, Value: `20`
   - Key: `offset`, Value: `0`

3. **Add Headers** (Headers tab)
   - Key: `Content-Type`, Value: `application/json`
   - Key: `Authorization`, Value: `Bearer token123`

4. **Send Request**
   - View response in Response panel
   - Check status code, headers, body

## CloudWatch Logs Analysis

View execution logs for debugging:

```
START RequestId: request-id-123
userId: 123
User found: John Doe
END RequestId: request-id-123
REPORT RequestId: request-id-123 Duration: 245.32 ms
```

**Common Log Patterns**:
- `userId` extraction for debugging
- User found/not found status
- Duration for performance optimization
- Any exceptions or errors

## Performance Optimization

### Minimize Cold Starts

```python
import boto3  # Global scope

# Reused across invocations
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    # Fast database access
    response = table.get_item(Key={'userId': user_id})
```

### Optimize DynamoDB Queries

- Use `get_item` for known keys (O(1))
- Use `query` for range operations (O(n))
- Avoid `scan` (full table scan, inefficient)
- Use GSI (Global Secondary Index) for searching by email

## Common Mistakes

**Mistake 1**: Forgetting to URL-encode path parameters
- Problem: Special characters cause parsing errors
- Solution: Ensure Lambda properly extracts from pathParameters

**Mistake 2**: Returning object instead of JSON string
- Problem: Lambda response body must be JSON string
- Solution: Use `json.dumps(data)` not `data`

**Mistake 3**: Not handling missing query parameters
- Problem: Crash when optional parameter absent
- Solution: Use `.get()` with default values

**Mistake 4**: Missing CORS headers
- Problem: Browser requests fail
- Solution: Include CORS headers in response

**Mistake 5**: Incorrect status codes
- Problem: Clients can't determine success/failure
- Solution: 200 for found, 404 for not found, 400 for invalid input

## Verification Checklist

- [ ] GET method created on correct resource
- [ ] Lambda proxy integration configured
- [ ] Path parameters extracted correctly in Lambda
- [ ] Query parameters parsed correctly
- [ ] 200 status code returned for successful requests
- [ ] 404 status code returned for missing resources
- [ ] 400 status code returned for invalid input
- [ ] CORS headers included in response
- [ ] Error handling for missing/invalid parameters
- [ ] DynamoDB query optimized (get_item, not scan)
- [ ] Lambda timeout set appropriately (5-10 seconds)
- [ ] IAM role has dynamodb:GetItem permission
- [ ] API deployed to stage
- [ ] Testing with curl/Postman succeeds

## Next Steps

After mastering GET operations:
- [post_method.md](post_method.md): Implementing POST for creating resources
- [delete_method.md](delete_method.md): Implementing DELETE for removing resources
- [python_lambda_integration.md](python_lambda_integration.md): Complete CRUD operations
- [server_lab.md](server_lab.md): Hands-on lab combining all HTTP methods
