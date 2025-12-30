# DELETE Method - Removing Resources with API Gateway

## DELETE Method Overview

The DELETE HTTP method removes resources from the server. DELETE requests are:
- **Not Safe**: Modifies server state (removes resources)
- **Idempotent**: Multiple identical requests have same effect as single request
- **Not Cacheable**: Results in resource removal
- **May Include Body**: Optional request body (typically empty)

**Use Cases**:
- Delete user account
- Remove order from system
- Soft delete (mark as deleted)
- Hard delete (permanent removal)
- Cleanup operations

DELETE Response Status Codes:
- **200 OK**: Resource deleted, response body included
- **202 Accepted**: Deletion in progress
- **204 No Content**: Resource deleted, empty response (most common)
- **404 Not Found**: Resource not found

## API Gateway DELETE Configuration

### Creating Resource and Method

**Step 1: Select Resource**
1. Navigate to API Gateway dashboard
2. Select your API
3. Select resource: `/users/{userId}` (path parameter)

**Step 2: Create DELETE Method**
1. Click "Actions" → "Create Method" → "DELETE"
2. Integration type: "Lambda Function"
3. Use Lambda Proxy Integration: ✓
4. Lambda function: `UserHandler`
5. Click "Save"

**Step 3: Configure Integration Response**
1. Select `DELETE` method
2. Click "Integration Response" → "200"
3. Mapping template content type: application/json
4. Mapping template: (leave empty for pass-through)

## Lambda Proxy Integration Event Structure

```json
{
  "httpMethod": "DELETE",
  "path": "/users/550e8400-e29b-41d4-a716-446655440000",
  "pathParameters": {
    "userId": "550e8400-e29b-41d4-a716-446655440000"
  },
  "headers": {
    "Host": "api.example.com",
    "Authorization": "Bearer token..."
  },
  "requestContext": {
    "accountId": "123456789012",
    "stage": "prod"
  }
}
```

## Basic DELETE Lambda Handler

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """
    DELETE request handler
    Removes user from DynamoDB
    """
    try:
        # Extract user ID from path parameters
        user_id = event.get('pathParameters', {}).get('userId')
        
        if not user_id:
            return {
                'statusCode': 400,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({'error': 'userId required'})
            }
        
        # Check if user exists
        response = table.get_item(Key={'userId': user_id})
        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': {'Content-Type': 'application/json'},
                'body': json.dumps({'error': 'User not found'})
            }
        
        # Delete user
        table.delete_item(Key={'userId': user_id})
        
        # Return 204 No Content (empty response)
        return {
            'statusCode': 204,
            'headers': {'Content-Type': 'application/json'},
            'body': ''
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Server error'})
        }
```

## Delete Strategies

### Hard Delete (Permanent)

Immediate permanent removal from database:

```python
def delete_user_hard(user_id):
    """Permanently delete user from database"""
    table.delete_item(Key={'userId': user_id})

def lambda_handler(event, context):
    user_id = event['pathParameters']['userId']
    
    # Check existence
    response = table.get_item(Key={'userId': user_id})
    if 'Item' not in response:
        return error_response(404, 'User not found')
    
    # Hard delete
    delete_user_hard(user_id)
    
    return {
        'statusCode': 204,
        'headers': {'Content-Type': 'application/json'},
        'body': ''
    }
```

### Soft Delete (Logical)

Mark as deleted, keep data for compliance/audit:

```python
def delete_user_soft(user_id):
    """Mark user as deleted (keep data)"""
    table.update_item(
        Key={'userId': user_id},
        UpdateExpression='SET #status = :status, deletedAt = :timestamp',
        ExpressionAttributeNames={'#status': 'status'},
        ExpressionAttributeValues={
            ':status': 'deleted',
            ':timestamp': '2025-01-01T10:00:00Z'
        }
    )

def lambda_handler(event, context):
    user_id = event['pathParameters']['userId']
    
    response = table.get_item(Key={'userId': user_id})
    if 'Item' not in response:
        return error_response(404, 'User not found')
    
    # Soft delete
    delete_user_soft(user_id)
    
    return {
        'statusCode': 204,
        'headers': {'Content-Type': 'application/json'},
        'body': ''
    }
```

### Cascade Delete

Delete resource and dependent resources:

```python
def lambda_handler(event, context):
    order_id = event['pathParameters']['orderId']
    
    try:
        # Get order
        order_response = table.get_item(Key={'orderId': order_id})
        if 'Item' not in order_response:
            return error_response(404, 'Order not found')
        
        # Delete all items in order
        items_response = items_table.query(
            KeyConditionExpression='orderId = :orderId',
            ExpressionAttributeValues={':orderId': order_id}
        )
        
        for item in items_response['Items']:
            items_table.delete_item(
                Key={'orderId': order_id, 'itemId': item['itemId']}
            )
        
        # Delete order
        table.delete_item(Key={'orderId': order_id})
        
        return {
            'statusCode': 204,
            'headers': {'Content-Type': 'application/json'},
            'body': ''
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return error_response(500, 'Server error')
```

## Idempotency

DELETE is idempotent - deleting twice returns same result:

```python
def lambda_handler(event, context):
    user_id = event['pathParameters']['userId']
    
    # Check if user exists
    response = table.get_item(Key={'userId': user_id})
    
    if 'Item' in response:
        # User exists, delete
        table.delete_item(Key={'userId': user_id})
        return {
            'statusCode': 204,
            'headers': {'Content-Type': 'application/json'},
            'body': ''
        }
    else:
        # User already deleted or doesn't exist
        # Still return 204 (idempotent)
        return {
            'statusCode': 204,
            'headers': {'Content-Type': 'application/json'},
            'body': ''
        }
```

## Authorization and Authentication

DELETE operations often require authentication:

```python
def lambda_handler(event, context):
    user_id = event['pathParameters']['userId']
    auth_header = event['headers'].get('Authorization', '')
    
    # Validate authorization
    current_user = validate_token(auth_header)
    if not current_user:
        return {
            'statusCode': 401,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Unauthorized'})
        }
    
    # Check if user can delete
    if current_user != user_id:
        return {
            'statusCode': 403,
            'headers': {'Content-Type': 'application/json'},
            'body': json.dumps({'error': 'Forbidden'})
        }
    
    # Delete user
    table.delete_item(Key={'userId': user_id})
    
    return {
        'statusCode': 204,
        'headers': {'Content-Type': 'application/json'},
        'body': ''
    }
```

## Complete Example 1: Delete User

Simple hard delete with existence check:

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """Delete user account"""
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    }
    
    try:
        # Extract user ID
        user_id = event.get('pathParameters', {}).get('userId')
        
        if not user_id:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'userId required'})
            }
        
        # Check if user exists
        response = table.get_item(Key={'userId': user_id})
        
        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': headers,
                'body': json.dumps({'error': 'User not found'})
            }
        
        # Store user data for response
        user = response['Item']
        
        # Delete user
        table.delete_item(Key={'userId': user_id})
        
        # Return 200 with deleted user data
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps({
                'message': 'User deleted successfully',
                'userId': user_id,
                'deletedUser': {
                    'name': user.get('name'),
                    'email': user.get('email')
                }
            })
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
curl -X DELETE https://api.example.com/users/550e8400-e29b-41d4-a716-446655440000
```

**Expected Response (200)**:
```json
{
  "message": "User deleted successfully",
  "userId": "550e8400-e29b-41d4-a716-446655440000",
  "deletedUser": {
    "name": "John Doe",
    "email": "john@example.com"
  }
}
```

## Complete Example 2: Soft Delete Order

Mark order as deleted but keep data:

```python
import json
import boto3
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Orders')

def lambda_handler(event, context):
    """Soft delete order"""
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    }
    
    try:
        order_id = event['pathParameters']['orderId']
        
        # Check if order exists
        response = table.get_item(Key={'orderId': order_id})
        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': headers,
                'body': json.dumps({'error': 'Order not found'})
            }
        
        order = response['Item']
        
        # Check if already deleted
        if order.get('status') == 'deleted':
            return {
                'statusCode': 410,
                'headers': headers,
                'body': json.dumps({'error': 'Order already deleted'})
            }
        
        # Soft delete - update status
        table.update_item(
            Key={'orderId': order_id},
            UpdateExpression='SET #status = :status, deletedAt = :timestamp',
            ExpressionAttributeNames={'#status': 'status'},
            ExpressionAttributeValues={
                ':status': 'deleted',
                ':timestamp': datetime.now().isoformat()
            }
        )
        
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps({
                'message': 'Order deleted successfully',
                'orderId': order_id,
                'status': 'deleted',
                'deletedAt': datetime.now().isoformat()
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({'error': 'Server error'})
        }
```

## Complete Example 3: Cascade Delete

Delete resource with dependent data:

```python
import json
import boto3

dynamodb = boto3.resource('dynamodb')
orders_table = dynamodb.Table('Orders')
order_items_table = dynamodb.Table('OrderItems')

def lambda_handler(event, context):
    """Delete order and all items"""
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    }
    
    try:
        order_id = event['pathParameters']['orderId']
        
        # Check if order exists
        order_response = orders_table.get_item(Key={'orderId': order_id})
        if 'Item' not in order_response:
            return {
                'statusCode': 404,
                'headers': headers,
                'body': json.dumps({'error': 'Order not found'})
            }
        
        # Delete all items
        items_response = order_items_table.query(
            KeyConditionExpression='orderId = :orderId',
            ExpressionAttributeValues={':orderId': order_id}
        )
        
        deleted_items = 0
        for item in items_response['Items']:
            order_items_table.delete_item(
                Key={
                    'orderId': order_id,
                    'itemId': item['itemId']
                }
            )
            deleted_items += 1
        
        # Delete order
        orders_table.delete_item(Key={'orderId': order_id})
        
        return {
            'statusCode': 200,
            'headers': headers,
            'body': json.dumps({
                'message': 'Order deleted successfully',
                'orderId': order_id,
                'itemsDeleted': deleted_items
            })
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({'error': 'Server error'})
        }
```

## IAM Permissions

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:DeleteItem",
        "dynamodb:UpdateItem",
        "dynamodb:Query"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/Users"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:DeleteItem",
        "dynamodb:Query"
      ],
      "Resource": [
        "arn:aws:dynamodb:us-east-1:123456789012:table/Orders",
        "arn:aws:dynamodb:us-east-1:123456789012:table/OrderItems"
      ]
    }
  ]
}
```

## Response Codes

### 200 OK
Return deleted resource data:
```json
{
  "statusCode": 200,
  "body": {
    "message": "User deleted",
    "userId": "550e8400-e29b-41d4-a716-446655440000"
  }
}
```

### 204 No Content
No response body (cleanest):
```
HTTP/1.1 204 No Content
Content-Length: 0
```

### 404 Not Found
Resource doesn't exist:
```json
{
  "statusCode": 404,
  "body": {"error": "User not found"}
}
```

### 410 Gone
Resource previously deleted:
```json
{
  "statusCode": 410,
  "body": {"error": "Resource has been deleted"}
}
```

## AWS Console Steps

### Step 1: Create DELETE Method
1. Select API in API Gateway
2. Select resource `/users/{userId}`
3. Actions → Create Method → DELETE
4. Integration type: Lambda Function
5. Lambda function: `UserHandler`
6. Use Lambda Proxy Integration: ✓
7. Click "Save"

### Step 2: Test in Console
1. Click "Test" (lightning bolt)
2. Path: /users/123
3. Click "Test"
4. View response

### Step 3: Deploy
1. Actions → Deploy API
2. Stage: prod
3. Click Deploy

## AWS CLI Commands

### Create DELETE Method

```bash
aws apigateway put-method \
    --rest-api-id abc123 \
    --resource-id resource-id \
    --http-method DELETE \
    --authorization-type NONE
```

### Add Lambda Integration

```bash
aws apigateway put-integration \
    --rest-api-id abc123 \
    --resource-id resource-id \
    --http-method DELETE \
    --type AWS_PROXY \
    --integration-http-method POST \
    --uri arn:aws:apigateway:us-east-1:lambda:path/2015-03-31/functions/arn:aws:lambda:us-east-1:123456789012:function:UserHandler/invocations
```

## Testing with curl

```bash
# Simple delete
curl -X DELETE https://api.example.com/users/550e8400-e29b-41d4-a716-446655440000

# Delete with headers
curl -X DELETE https://api.example.com/users/550e8400-e29b-41d4-a716-446655440000 \
  -H "Authorization: Bearer token123"

# Check response code
curl -w "%{http_code}\n" -X DELETE https://api.example.com/users/550e8400-e29b-41d4-a716-446655440000
```

## Common Mistakes

**Mistake 1**: Not checking if resource exists
- Problem: Deleting non-existent resource succeeds silently
- Solution: Query first, return 404 if not found

**Mistake 2**: Hard delete when soft delete needed
- Problem: No audit trail or data recovery option
- Solution: Implement soft delete for compliance

**Mistake 3**: Not handling cascade deletes
- Problem: Orphaned data left in related tables
- Solution: Delete or update dependent records

**Mistake 4**: Wrong HTTP status code
- Problem: Client doesn't know what happened
- Solution: Return 204 for empty response, 200 for data response

**Mistake 5**: Deleting without authorization
- Problem: Users can delete others' data
- Solution: Verify ownership before deletion

## Verification Checklist

- [ ] DELETE method created on path resource (`/{userId}`)
- [ ] Path parameters extracted correctly
- [ ] Existence check before deletion
- [ ] Appropriate status code returned
- [ ] Cascade deletes handled if needed
- [ ] Soft or hard delete strategy chosen
- [ ] CORS headers included
- [ ] Error responses for 404
- [ ] IAM role has dynamodb:GetItem, dynamodb:DeleteItem
- [ ] API deployed to stage
- [ ] Testing with curl succeeds
- [ ] Idempotency verified (delete twice works)

## Next Steps

- [python_lambda_integration.md](python_lambda_integration.md): Complete CRUD patterns
- [server_lab.md](server_lab.md): Hands-on lab with all methods
