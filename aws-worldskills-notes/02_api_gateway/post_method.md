# POST Method - Creating Resources with API Gateway

## POST Method Overview

The POST HTTP method creates new resources on the server. POST requests are:
- **Not Safe**: Modifies server state (creates new resources)
- **Not Idempotent**: Multiple identical requests create multiple resources
- **Not Cacheable**: Results depend on request body content
- **Includes Request Body**: Data for new resource included in body

**Use Cases**:
- User registration (create new user)
- Order creation
- Comment submission
- File upload metadata
- Any operation creating new records

## POST Request Characteristics

```
POST /users HTTP/1.1
Host: api.example.com
Content-Type: application/json
Content-Length: 47

{"name":"John Doe","email":"john@example.com"}
```

**Key Points**:
- Includes request body with JSON data
- Resource identifier typically in response (Location header)
- Response status: 201 Created (with Location header and created resource)
- Content-Type header required

## API Gateway POST Configuration

### Creating Resource and Method

**Step 1: Create Resource**
1. Navigate to API Gateway dashboard
2. Select your API
3. Click resource or create new
4. Resource name: `users`, Path: `/users`

**Step 2: Create POST Method**
1. Select `/users` resource
2. Click "Actions" → "Create Method" → "POST"
3. Integration type: "Lambda Function"
4. Use Lambda Proxy Integration: ✓
5. Lambda function: `UserHandler`
6. Click "Save"

**Step 3: Enable CORS**
1. Select `/users` resource
2. Actions → Enable CORS and replace existing CORS headers
3. Select "POST" method
4. Click "Enable CORS and replace CORS headers"
5. Deploy API

### Lambda Proxy Integration Event Structure

```json
{
  "httpMethod": "POST",
  "path": "/users",
  "headers": {
    "Content-Type": "application/json",
    "Host": "api.example.com"
  },
  "body": "{\"name\":\"John Doe\",\"email\":\"john@example.com\"}",
  "isBase64Encoded": false,
  "requestContext": {
    "requestId": "request-id-123",
    "stage": "prod"
  }
}
```

## Lambda Handler for POST

Basic structure for handling POST requests:

```python
import json
import boto3
import uuid

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """
    POST request handler
    Creates new user in DynamoDB
    """
    try:
        # Parse request body
        if not event.get('body'):
            return {
                'statusCode': 400,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({'error': 'Request body required'})
            }
        
        body = json.loads(event['body'])
        
        # Extract and validate fields
        name = body.get('name', '').strip()
        email = body.get('email', '').strip()
        
        # Validation
        if not name:
            return error_response(400, 'Name is required')
        
        if not email or '@' not in email:
            return error_response(400, 'Valid email is required')
        
        # Generate unique ID
        user_id = str(uuid.uuid4())
        
        # Store in DynamoDB
        table.put_item(
            Item={
                'userId': user_id,
                'name': name,
                'email': email
            }
        )
        
        # Return created response
        return {
            'statusCode': 201,
            'headers': {
                'Content-Type': 'application/json',
                'Location': f'/users/{user_id}'
            },
            'body': json.dumps({
                'userId': user_id,
                'name': name,
                'email': email
            })
        }
        
    except json.JSONDecodeError:
        return error_response(400, 'Invalid JSON format')
    except Exception as e:
        print(f"Error: {str(e)}")
        return error_response(500, 'Internal server error')

def error_response(status_code, message):
    """Helper function to return error response"""
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'error': message})
    }
```

## Request Body Parsing

### Basic JSON Parsing

```python
def lambda_handler(event, context):
    # Parse JSON body
    try:
        body = json.loads(event.get('body', '{}'))
    except json.JSONDecodeError:
        return error_response(400, 'Invalid JSON')
    
    # Extract fields
    name = body.get('name')
    email = body.get('email')
```

### Handling Malformed JSON

```python
def lambda_handler(event, context):
    body_str = event.get('body', '')
    
    if not body_str:
        return error_response(400, 'Empty request body')
    
    try:
        body = json.loads(body_str)
    except json.JSONDecodeError as e:
        print(f"JSON parse error: {str(e)}")
        return error_response(400, 'Invalid JSON format')
    
    return body
```

### Base64-Encoded Bodies

Some integrations send base64-encoded bodies:

```python
import base64

def lambda_handler(event, context):
    if event.get('isBase64Encoded'):
        body_str = base64.b64decode(event['body']).decode('utf-8')
    else:
        body_str = event.get('body', '')
    
    body = json.loads(body_str)
```

## Input Validation

### Required Fields

```python
def validate_user_data(data):
    """Validate user registration data"""
    errors = []
    
    # Check required fields
    if not data.get('name'):
        errors.append('name is required')
    
    if not data.get('email'):
        errors.append('email is required')
    
    if errors:
        return False, errors
    
    return True, []

def lambda_handler(event, context):
    body = json.loads(event['body'])
    valid, errors = validate_user_data(body)
    
    if not valid:
        return {
            'statusCode': 400,
            'body': json.dumps({'errors': errors})
        }
```

### Email Format Validation

```python
import re

def validate_email(email):
    """Simple email validation"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def lambda_handler(event, context):
    body = json.loads(event['body'])
    email = body.get('email', '').strip()
    
    if not validate_email(email):
        return error_response(400, 'Invalid email format')
```

### String Length Validation

```python
def validate_input(data):
    """Validate input field lengths"""
    name = data.get('name', '').strip()
    email = data.get('email', '').strip()
    
    if len(name) < 2:
        return False, 'Name must be at least 2 characters'
    
    if len(name) > 100:
        return False, 'Name cannot exceed 100 characters'
    
    if len(email) > 254:
        return False, 'Email cannot exceed 254 characters'
    
    return True, None
```

## Response Format

POST responses typically return 201 Created with Location header:

```python
{
    'statusCode': 201,
    'headers': {
        'Content-Type': 'application/json',
        'Location': '/users/123-456-789',
        'Access-Control-Allow-Origin': '*'
    },
    'body': json.dumps({
        'userId': '123-456-789',
        'name': 'John Doe',
        'email': 'john@example.com',
        'createdAt': '2025-01-01T10:00:00Z'
    })
}
```

### Response Status Codes

**201 Created**: Resource created successfully, includes Location header
**400 Bad Request**: Validation error or malformed JSON
**409 Conflict**: Resource already exists (duplicate email)
**500 Internal Server Error**: Server error during creation

## Error Handling

### Validation Error Response

```python
def validate_unique_email(email):
    """Check if email already exists"""
    response = table.query(
        KeyConditionExpression='email = :email',
        ExpressionAttributeValues={':email': email}
    )
    return len(response['Items']) == 0

def lambda_handler(event, context):
    body = json.loads(event['body'])
    email = body.get('email', '').strip()
    
    if not validate_unique_email(email):
        return {
            'statusCode': 409,
            'body': json.dumps({'error': 'Email already registered'})
        }
```

### Comprehensive Error Handling

```python
def lambda_handler(event, context):
    try:
        # Parse and validate
        body = json.loads(event.get('body', '{}'))
        valid, error_msg = validate_input(body)
        
        if not valid:
            return error_response(400, error_msg)
        
        # Check for duplicates
        if not validate_unique_email(body['email']):
            return error_response(409, 'Email already exists')
        
        # Create resource
        item_id = str(uuid.uuid4())
        table.put_item(Item={
            'userId': item_id,
            'name': body['name'],
            'email': body['email']
        })
        
        return success_response(201, {'userId': item_id, 'name': body['name']})
        
    except json.JSONDecodeError:
        return error_response(400, 'Invalid JSON')
    except Exception as e:
        print(f"Error: {str(e)}")
        return error_response(500, 'Server error')
```

## CORS Configuration for POST

POST requests from browsers require CORS preflight:

```python
def lambda_handler(event, context):
    # CORS headers
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'POST,OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type,Authorization',
        'Content-Type': 'application/json'
    }
    
    # Handle OPTIONS preflight
    if event.get('httpMethod') == 'OPTIONS':
        return {
            'statusCode': 200,
            'headers': headers,
            'body': ''
        }
    
    # Handle POST
    # ... POST logic ...
    
    return {
        'statusCode': 201,
        'headers': headers,
        'body': json.dumps(data)
    }
```

## Complete Example 1: User Registration API

**Scenario**: Create new user account with email validation

**Lambda Code**:

```python
import json
import boto3
import uuid
import re

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def validate_email(email):
    """Validate email format"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def lambda_handler(event, context):
    """Create new user"""
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    }
    
    # Handle OPTIONS
    if event.get('httpMethod') == 'OPTIONS':
        return {
            'statusCode': 200,
            'headers': headers,
            'body': ''
        }
    
    try:
        # Parse body
        if not event.get('body'):
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'Body required'})
            }
        
        body = json.loads(event['body'])
        
        # Validate inputs
        name = body.get('name', '').strip()
        email = body.get('email', '').strip()
        
        if not name or len(name) < 2:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'Name must be at least 2 chars'})
            }
        
        if not validate_email(email):
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'Invalid email'})
            }
        
        # Check for duplicates
        response = table.query(
            IndexName='email-index',
            KeyConditionExpression='email = :email',
            ExpressionAttributeValues={':email': email}
        )
        
        if response['Items']:
            return {
                'statusCode': 409,
                'headers': headers,
                'body': json.dumps({'error': 'Email already registered'})
            }
        
        # Create user
        user_id = str(uuid.uuid4())
        table.put_item(
            Item={
                'userId': user_id,
                'name': name,
                'email': email,
                'createdAt': '2025-01-01T10:00:00Z'
            }
        )
        
        # Return created
        return {
            'statusCode': 201,
            'headers': {
                **headers,
                'Location': f'/users/{user_id}'
            },
            'body': json.dumps({
                'userId': user_id,
                'name': name,
                'email': email,
                'message': 'User registered successfully'
            })
        }
        
    except json.JSONDecodeError:
        return {
            'statusCode': 400,
            'headers': headers,
            'body': json.dumps({'error': 'Invalid JSON'})
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
      "Action": ["dynamodb:PutItem", "dynamodb:Query"],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:123456789012:table/Users",
        "arn:aws:dynamodb:us-east-1:123456789012:table/Users/index/*"
      ]
    }
  ]
}
```

**Test curl Command**:
```bash
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com"}'
```

**Expected Response (201)**:
```json
{
  "statusCode": 201,
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "name": "John Doe",
  "email": "john@example.com",
  "message": "User registered successfully"
}
```

## Complete Example 2: Create Order with Items

**Scenario**: Create order with multiple items, store in separate DynamoDB tables

**Lambda Code**:

```python
import json
import boto3
import uuid

dynamodb = boto3.resource('dynamodb')
orders_table = dynamodb.Table('Orders')
order_items_table = dynamodb.Table('OrderItems')

def lambda_handler(event, context):
    """Create order with items"""
    
    headers = {'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*'}
    
    try:
        body = json.loads(event.get('body', '{}'))
        
        # Validate order data
        customer_id = body.get('customerId')
        items = body.get('items', [])
        
        if not customer_id:
            return error_response(400, 'customerId required', headers)
        
        if not items or len(items) == 0:
            return error_response(400, 'At least one item required', headers)
        
        # Generate order ID
        order_id = str(uuid.uuid4())
        total_price = 0
        
        # Create order
        orders_table.put_item(
            Item={
                'orderId': order_id,
                'customerId': customer_id,
                'status': 'pending',
                'createdAt': '2025-01-01T10:00:00Z'
            }
        )
        
        # Add items to order
        for item in items:
            item_id = str(uuid.uuid4())
            price = float(item.get('price', 0))
            quantity = int(item.get('quantity', 1))
            item_total = price * quantity
            total_price += item_total
            
            order_items_table.put_item(
                Item={
                    'orderId': order_id,
                    'itemId': item_id,
                    'productId': item.get('productId'),
                    'quantity': quantity,
                    'price': price,
                    'total': item_total
                }
            )
        
        # Update order total
        orders_table.update_item(
            Key={'orderId': order_id},
            UpdateExpression='SET totalPrice = :total',
            ExpressionAttributeValues={':total': total_price}
        )
        
        return {
            'statusCode': 201,
            'headers': headers,
            'body': json.dumps({
                'orderId': order_id,
                'customerId': customer_id,
                'totalPrice': total_price,
                'itemCount': len(items),
                'status': 'pending'
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return error_response(500, 'Server error', headers)

def error_response(status, msg, headers):
    return {
        'statusCode': status,
        'headers': headers,
        'body': json.dumps({'error': msg})
    }
```

**Test Request**:
```bash
curl -X POST https://api.example.com/orders \
  -H "Content-Type: application/json" \
  -d '{
    "customerId": "customer-123",
    "items": [
      {"productId": "prod-1", "quantity": 2, "price": 29.99},
      {"productId": "prod-2", "quantity": 1, "price": 49.99}
    ]
  }'
```

## Complete Example 3: File Metadata Upload

**Scenario**: Store file metadata and generate presigned URL for S3 upload

**Lambda Code**:

```python
import json
import boto3
import uuid

dynamodb = boto3.resource('dynamodb')
s3_client = boto3.client('s3')
files_table = dynamodb.Table('Files')

def lambda_handler(event, context):
    """Upload file metadata and get presigned URL"""
    
    headers = {'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*'}
    
    try:
        body = json.loads(event.get('body', '{}'))
        
        # Validate
        filename = body.get('filename', '').strip()
        file_type = body.get('fileType', '').strip()
        
        if not filename:
            return error_response(400, 'Filename required', headers)
        
        # Generate file ID
        file_id = str(uuid.uuid4())
        
        # Store metadata in DynamoDB
        files_table.put_item(
            Item={
                'fileId': file_id,
                'filename': filename,
                'fileType': file_type,
                'status': 'pending',
                'createdAt': '2025-01-01T10:00:00Z'
            }
        )
        
        # Generate presigned URL for S3 upload
        presigned_url = s3_client.generate_presigned_post(
            Bucket='my-upload-bucket',
            Key=f'uploads/{file_id}/{filename}',
            ExpiresIn=3600  # 1 hour
        )
        
        return {
            'statusCode': 201,
            'headers': headers,
            'body': json.dumps({
                'fileId': file_id,
                'filename': filename,
                'presignedUrl': presigned_url['url'],
                'fields': presigned_url['fields']
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return error_response(500, 'Server error', headers)

def error_response(status, msg, headers):
    return {
        'statusCode': status,
        'headers': headers,
        'body': json.dumps({'error': msg})
    }
```

## Request Validation Schema (REST API)

REST API supports JSON schema validation (HTTP API does not):

```json
{
  "type": "object",
  "required": ["name", "email"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 2,
      "maxLength": 100
    },
    "email": {
      "type": "string",
      "format": "email"
    },
    "age": {
      "type": "integer",
      "minimum": 0,
      "maximum": 150
    }
  }
}
```

**Benefits**:
- Validation before Lambda invocation
- Reduces Lambda costs
- Better performance

## Idempotency Considerations

POST is not idempotent, but you can implement idempotent POST using idempotency keys:

```python
def lambda_handler(event, context):
    headers = {'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*'}
    body = json.loads(event['body'])
    
    # Get idempotency key from header or body
    idempotency_key = event['headers'].get('Idempotency-Key') or body.get('idempotencyKey')
    
    if not idempotency_key:
        return error_response(400, 'Idempotency-Key required', headers)
    
    # Check if already processed
    idempotency_table = dynamodb.Table('Idempotency')
    response = idempotency_table.get_item(Key={'key': idempotency_key})
    
    if 'Item' in response:
        # Already processed, return cached result
        return {
            'statusCode': 201,
            'headers': headers,
            'body': json.dumps(response['Item']['result'])
        }
    
    # Process request
    result = create_resource(body)
    
    # Cache result
    idempotency_table.put_item(
        Item={'key': idempotency_key, 'result': result}
    )
    
    return {
        'statusCode': 201,
        'headers': headers,
        'body': json.dumps(result)
    }
```

## Security Best Practices

### Input Sanitization

```python
def sanitize_input(text):
    """Remove potentially dangerous characters"""
    # Remove HTML tags
    text = re.sub(r'<[^>]+>', '', text)
    # Remove script tags
    text = re.sub(r'<script[^>]*>.*?</script>', '', text, flags=re.IGNORECASE)
    return text

def lambda_handler(event, context):
    body = json.loads(event['body'])
    name = sanitize_input(body.get('name', ''))
```

### NoSQL Injection Prevention

```python
# Bad: Concatenating user input
response = table.query(
    KeyConditionExpression=f"email = {email}"  # VULNERABLE
)

# Good: Using expression attributes
response = table.query(
    KeyConditionExpression="email = :email",
    ExpressionAttributeValues={':email': email}  # SAFE
)
```

## AWS Console Steps - Creating POST Method

### Step 1: Create Resource
1. Select API in API Gateway
2. Actions → Create Resource
3. Resource path: `/users`
4. Click "Create Resource"

### Step 2: Create POST Method
1. Select `/users` resource
2. Actions → Create Method → POST
3. Integration type: Lambda Function
4. Lambda function: `UserHandler`
5. Use Lambda Proxy Integration: ✓
6. Click "Save"

### Step 3: Enable CORS
1. Select resource
2. Actions → Enable CORS and replace existing CORS headers
3. Select "POST" method
4. Click "Enable CORS and replace CORS headers"

### Step 4: Deploy
1. Actions → Deploy API
2. Stage: `prod`
3. Click "Deploy"

## AWS CLI Commands

### Create POST Method

```bash
aws apigateway put-method \
    --rest-api-id abc123 \
    --resource-id resource-id \
    --http-method POST \
    --authorization-type NONE
```

### Add Integration

```bash
aws apigateway put-integration \
    --rest-api-id abc123 \
    --resource-id resource-id \
    --http-method POST \
    --type AWS_PROXY \
    --integration-http-method POST \
    --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789012:function:UserHandler/invocations
```

## Testing with curl

```bash
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -d '{"name":"John Doe","email":"john@example.com"}'
```

## Testing with Postman

1. **Create POST Request**
   - Method: POST
   - URL: `https://api.example.com/users`

2. **Set Body**
   - Raw JSON:
   ```json
   {"name":"John Doe","email":"john@example.com"}
   ```

3. **Send Request**
   - Check status code (should be 201)
   - Verify response body and Location header

## Common Mistakes

**Mistake 1**: Returning object instead of JSON string
- Problem: Lambda response body must be string
- Solution: Use `json.dumps(data)`

**Mistake 2**: Missing Content-Type header
- Problem: Clients can't parse response
- Solution: Add `'Content-Type': 'application/json'`

**Mistake 3**: Not handling validation errors
- Problem: Invalid input creates corrupted data
- Solution: Validate before database write

**Mistake 4**: Forgetting Location header
- Problem: Clients can't find newly created resource
- Solution: Add Location header with resource URI

**Mistake 5**: Not checking for duplicates
- Problem: Duplicate resources created
- Solution: Query for existing resource before creating

## Verification Checklist

- [ ] POST method created on correct resource
- [ ] Lambda proxy integration configured
- [ ] Request body parsed as JSON
- [ ] All required fields validated
- [ ] Email format validated
- [ ] Duplicate check implemented
- [ ] 201 Created status returned
- [ ] Location header included
- [ ] CORS headers present
- [ ] Error responses return 400 for validation
- [ ] Error responses return 409 for conflicts
- [ ] IAM role has dynamodb:PutItem, dynamodb:Query permissions
- [ ] API deployed to stage
- [ ] Testing with curl/Postman succeeds

## Next Steps

After mastering POST operations:
- [delete_method.md](delete_method.md): Implementing DELETE operations
- [python_lambda_integration.md](python_lambda_integration.md): Complete CRUD patterns
- [server_lab.md](server_lab.md): Hands-on lab with all HTTP methods
