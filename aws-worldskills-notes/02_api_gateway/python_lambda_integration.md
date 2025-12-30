# Python Lambda Integration Patterns for API Gateway

## CRUD Router Pattern

Building complete CRUD APIs with single Lambda function using router pattern:

```python
import json
import boto3
import uuid
from datetime import datetime

dynamodb = boto3.resource('dynamodb')

def lambda_handler(event, context):
    """Main router for all HTTP methods"""
    
    method = event.get('httpMethod')
    resource = event.get('resource', '')
    
    # Route to appropriate handler
    try:
        if method == 'GET':
            if '{userId}' in resource:
                return handle_get_user(event)
            else:
                return handle_list_users(event)
        
        elif method == 'POST':
            return handle_create_user(event)
        
        elif method == 'PUT':
            return handle_update_user(event)
        
        elif method == 'DELETE':
            return handle_delete_user(event)
        
        else:
            return error_response(405, 'Method not allowed')
    
    except Exception as e:
        print(f"Error: {str(e)}")
        return error_response(500, 'Server error')

def handle_get_user(event):
    """GET /users/{userId}"""
    user_id = event['pathParameters']['userId']
    table = dynamodb.Table('Users')
    
    response = table.get_item(Key={'userId': user_id})
    
    if 'Item' not in response:
        return error_response(404, 'User not found')
    
    return success_response(200, response['Item'])

def handle_list_users(event):
    """GET /users?limit=10&lastKey=..."""
    table = dynamodb.Table('Users')
    
    limit = int(event.get('queryStringParameters', {}).get('limit', 10))
    limit = min(limit, 100)  # Max 100
    
    scan_kwargs = {'Limit': limit}
    last_key = event.get('queryStringParameters', {}).get('lastKey')
    if last_key:
        scan_kwargs['ExclusiveStartKey'] = {'userId': last_key}
    
    response = table.scan(**scan_kwargs)
    
    return {
        'statusCode': 200,
        'headers': {'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*'},
        'body': json.dumps({
            'items': response.get('Items', []),
            'count': response['Count'],
            'lastEvaluatedKey': response.get('LastEvaluatedKey')
        })
    }

def handle_create_user(event):
    """POST /users"""
    body = json.loads(event.get('body', '{}'))
    table = dynamodb.Table('Users')
    
    # Validate
    name = body.get('name', '').strip()
    email = body.get('email', '').strip()
    
    if not name:
        return error_response(400, 'Name required')
    if not email:
        return error_response(400, 'Email required')
    
    # Create
    user_id = str(uuid.uuid4())
    table.put_item(Item={
        'userId': user_id,
        'name': name,
        'email': email,
        'createdAt': datetime.now().isoformat()
    })
    
    return {
        'statusCode': 201,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
            'Location': f'/users/{user_id}'
        },
        'body': json.dumps({'userId': user_id, 'name': name, 'email': email})
    }

def handle_update_user(event):
    """PUT /users/{userId}"""
    user_id = event['pathParameters']['userId']
    body = json.loads(event.get('body', '{}'))
    table = dynamodb.Table('Users')
    
    # Check exists
    response = table.get_item(Key={'userId': user_id})
    if 'Item' not in response:
        return error_response(404, 'User not found')
    
    # Update
    update_expr = 'SET '
    expr_values = {}
    
    if 'name' in body:
        update_expr += '#name = :name, '
        expr_values[':name'] = body['name']
    
    if 'email' in body:
        update_expr += 'email = :email, '
        expr_values[':email'] = body['email']
    
    update_expr += 'updatedAt = :updated'
    expr_values[':updated'] = datetime.now().isoformat()
    
    table.update_item(
        Key={'userId': user_id},
        UpdateExpression=update_expr,
        ExpressionAttributeNames={'#name': 'name'} if 'name' in body else {},
        ExpressionAttributeValues=expr_values
    )
    
    return success_response(200, {'message': 'User updated'})

def handle_delete_user(event):
    """DELETE /users/{userId}"""
    user_id = event['pathParameters']['userId']
    table = dynamodb.Table('Users')
    
    response = table.get_item(Key={'userId': user_id})
    if 'Item' not in response:
        return error_response(404, 'User not found')
    
    table.delete_item(Key={'userId': user_id})
    
    return {
        'statusCode': 204,
        'headers': {'Access-Control-Allow-Origin': '*'},
        'body': ''
    }

def error_response(status_code, message):
    """Helper for error responses"""
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*'},
        'body': json.dumps({'error': message})
    }

def success_response(status_code, data):
    """Helper for success responses"""
    return {
        'statusCode': status_code,
        'headers': {'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*'},
        'body': json.dumps(data)
    }
```

## Environment Variables and Configuration

Use environment variables for configuration:

```python
import os
import json
import boto3

# Get environment variables
USERS_TABLE = os.environ.get('USERS_TABLE', 'Users')
REGION = os.environ.get('AWS_REGION', 'us-east-1')
DEBUG = os.environ.get('DEBUG', 'false').lower() == 'true'

dynamodb = boto3.resource('dynamodb', region_name=REGION)

def lambda_handler(event, context):
    if DEBUG:
        print(f"Event: {json.dumps(event)}")
    
    table = dynamodb.Table(USERS_TABLE)
    
    # Handler logic using table
    user_id = event['pathParameters']['userId']
    response = table.get_item(Key={'userId': user_id})
    
    if DEBUG:
        print(f"Response: {json.dumps(response)}")
    
    return {'statusCode': 200, 'body': json.dumps(response.get('Item'))}
```

**Deploy with Environment Variables**:
```bash
aws lambda update-function-configuration \
    --function-name UserHandler \
    --environment Variables="{USERS_TABLE=Users,DEBUG=false}" \
    --region us-east-1
```

## Complete User Management API

Full implementation with validation, error handling:

```python
import json
import boto3
import uuid
import re
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    """User Management API"""
    
    method = event.get('httpMethod')
    path = event.get('path')
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Methods': 'GET,POST,PUT,DELETE,OPTIONS',
        'Access-Control-Allow-Headers': 'Content-Type'
    }
    
    # Handle preflight
    if method == 'OPTIONS':
        return {'statusCode': 200, 'headers': headers, 'body': ''}
    
    try:
        if path == '/users' or path.endswith('/users'):
            if method == 'GET':
                return handle_list_users(event, headers)
            elif method == 'POST':
                return handle_create_user(event, headers)
        
        elif '/users/' in path:
            if method == 'GET':
                return handle_get_user(event, headers)
            elif method == 'PUT':
                return handle_update_user(event, headers)
            elif method == 'DELETE':
                return handle_delete_user(event, headers)
        
        return error_response(404, 'Not found', headers)
    
    except Exception as e:
        print(f"Error: {str(e)}")
        return error_response(500, 'Server error', headers)

def validate_email(email):
    """Validate email format"""
    pattern = r'^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}$'
    return re.match(pattern, email) is not None

def handle_get_user(event, headers):
    """GET /users/{userId}"""
    user_id = event.get('pathParameters', {}).get('userId')
    
    if not user_id:
        return error_response(400, 'userId required', headers)
    
    response = table.get_item(Key={'userId': user_id})
    
    if 'Item' not in response:
        return error_response(404, 'User not found', headers)
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps(response['Item'])
    }

def handle_list_users(event, headers):
    """GET /users with pagination"""
    
    query_params = event.get('queryStringParameters', {}) or {}
    limit = min(int(query_params.get('limit', 10)), 100)
    
    scan_kwargs = {'Limit': limit}
    
    if 'lastKey' in query_params:
        scan_kwargs['ExclusiveStartKey'] = {
            'userId': query_params['lastKey']
        }
    
    response = table.scan(**scan_kwargs)
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps({
            'items': response.get('Items', []),
            'count': response['Count'],
            'total': response.get('ScannedCount'),
            'nextToken': response.get('LastEvaluatedKey', {}).get('userId')
        })
    }

def handle_create_user(event, headers):
    """POST /users - Create user"""
    
    try:
        body = json.loads(event.get('body', '{}'))
    except json.JSONDecodeError:
        return error_response(400, 'Invalid JSON', headers)
    
    # Validate inputs
    name = body.get('name', '').strip()
    email = body.get('email', '').strip()
    
    if not name:
        return error_response(400, 'Name required', headers)
    
    if len(name) < 2:
        return error_response(400, 'Name too short', headers)
    
    if not email:
        return error_response(400, 'Email required', headers)
    
    if not validate_email(email):
        return error_response(400, 'Invalid email format', headers)
    
    # Check for duplicate
    response = table.query(
        IndexName='email-index',
        KeyConditionExpression='email = :email',
        ExpressionAttributeValues={':email': email}
    )
    
    if response['Items']:
        return error_response(409, 'Email already exists', headers)
    
    # Create user
    user_id = str(uuid.uuid4())
    table.put_item(Item={
        'userId': user_id,
        'name': name,
        'email': email,
        'createdAt': datetime.now().isoformat(),
        'status': 'active'
    })
    
    return {
        'statusCode': 201,
        'headers': {**headers, 'Location': f'/users/{user_id}'},
        'body': json.dumps({
            'userId': user_id,
            'name': name,
            'email': email,
            'createdAt': datetime.now().isoformat()
        })
    }

def handle_update_user(event, headers):
    """PUT /users/{userId} - Update user"""
    
    user_id = event.get('pathParameters', {}).get('userId')
    if not user_id:
        return error_response(400, 'userId required', headers)
    
    try:
        body = json.loads(event.get('body', '{}'))
    except json.JSONDecodeError:
        return error_response(400, 'Invalid JSON', headers)
    
    # Check existence
    response = table.get_item(Key={'userId': user_id})
    if 'Item' not in response:
        return error_response(404, 'User not found', headers)
    
    # Build update expression
    updates = []
    expr_values = {}
    expr_names = {}
    
    if 'name' in body:
        name = body['name'].strip()
        if len(name) < 2:
            return error_response(400, 'Name too short', headers)
        updates.append('#name = :name')
        expr_values[':name'] = name
        expr_names['#name'] = 'name'
    
    if 'email' in body:
        email = body['email'].strip()
        if not validate_email(email):
            return error_response(400, 'Invalid email', headers)
        updates.append('email = :email')
        expr_values[':email'] = email
    
    if not updates:
        return error_response(400, 'No fields to update', headers)
    
    updates.append('updatedAt = :updated')
    expr_values[':updated'] = datetime.now().isoformat()
    
    # Update
    table.update_item(
        Key={'userId': user_id},
        UpdateExpression='SET ' + ', '.join(updates),
        ExpressionAttributeNames=expr_names if expr_names else None,
        ExpressionAttributeValues=expr_values
    )
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps({'message': 'User updated', 'userId': user_id})
    }

def handle_delete_user(event, headers):
    """DELETE /users/{userId}"""
    
    user_id = event.get('pathParameters', {}).get('userId')
    if not user_id:
        return error_response(400, 'userId required', headers)
    
    response = table.get_item(Key={'userId': user_id})
    if 'Item' not in response:
        return error_response(404, 'User not found', headers)
    
    table.delete_item(Key={'userId': user_id})
    
    return {
        'statusCode': 204,
        'headers': headers,
        'body': ''
    }

def error_response(status, message, headers):
    """Error response helper"""
    return {
        'statusCode': status,
        'headers': headers,
        'body': json.dumps({'error': message})
    }
```

## Complete Product Catalog API

Different domain showing router pattern flexibility:

```python
import json
import boto3
import uuid
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
products_table = dynamodb.Table('Products')
inventory_table = dynamodb.Table('Inventory')

def lambda_handler(event, context):
    """Product Catalog API"""
    
    method = event.get('httpMethod')
    resource = event.get('resource', '')
    
    headers = {
        'Content-Type': 'application/json',
        'Access-Control-Allow-Origin': '*'
    }
    
    try:
        if '/products' in resource:
            if '{productId}' in resource:
                if method == 'GET':
                    return get_product(event, headers)
                elif method == 'PUT':
                    return update_product(event, headers)
                elif method == 'DELETE':
                    return delete_product(event, headers)
            else:
                if method == 'GET':
                    return list_products(event, headers)
                elif method == 'POST':
                    return create_product(event, headers)
        
        elif '/inventory' in resource:
            if method == 'GET':
                return get_inventory(event, headers)
            elif method == 'PUT':
                return update_inventory(event, headers)
        
        return {'statusCode': 404, 'headers': headers, 'body': '{}'}
    
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({'error': str(e)})
        }

def get_product(event, headers):
    """GET /products/{productId}"""
    product_id = event['pathParameters']['productId']
    
    response = products_table.get_item(Key={'productId': product_id})
    if 'Item' not in response:
        return {
            'statusCode': 404,
            'headers': headers,
            'body': json.dumps({'error': 'Product not found'})
        }
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps(response['Item'])
    }

def list_products(event, headers):
    """GET /products with filtering"""
    query_params = event.get('queryStringParameters', {}) or {}
    
    category = query_params.get('category')
    limit = min(int(query_params.get('limit', 20)), 100)
    
    if category:
        # Use GSI for category filtering
        response = products_table.query(
            IndexName='category-index',
            KeyConditionExpression='category = :cat',
            ExpressionAttributeValues={':cat': category},
            Limit=limit
        )
    else:
        response = products_table.scan(Limit=limit)
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps({
            'items': response['Items'],
            'count': response['Count']
        })
    }

def create_product(event, headers):
    """POST /products - Create product"""
    body = json.loads(event.get('body', '{}'))
    
    # Validate
    name = body.get('name', '').strip()
    price = float(body.get('price', 0))
    category = body.get('category', '').strip()
    
    if not name or price <= 0 or not category:
        return {
            'statusCode': 400,
            'headers': headers,
            'body': json.dumps({'error': 'name, price, category required'})
        }
    
    product_id = str(uuid.uuid4())
    
    products_table.put_item(Item={
        'productId': product_id,
        'name': name,
        'price': price,
        'category': category,
        'description': body.get('description', ''),
        'createdAt': datetime.now().isoformat()
    })
    
    # Initialize inventory
    inventory_table.put_item(Item={
        'productId': product_id,
        'quantity': int(body.get('initialQuantity', 0))
    })
    
    return {
        'statusCode': 201,
        'headers': {**headers, 'Location': f'/products/{product_id}'},
        'body': json.dumps({
            'productId': product_id,
            'name': name,
            'price': price
        })
    }

def update_product(event, headers):
    """PUT /products/{productId}"""
    product_id = event['pathParameters']['productId']
    body = json.loads(event.get('body', '{}'))
    
    # Check existence
    response = products_table.get_item(Key={'productId': product_id})
    if 'Item' not in response:
        return {
            'statusCode': 404,
            'headers': headers,
            'body': json.dumps({'error': 'Not found'})
        }
    
    # Update
    updates = ['updatedAt = :updated']
    expr_values = {':updated': datetime.now().isoformat()}
    
    if 'price' in body:
        updates.append('price = :price')
        expr_values[':price'] = float(body['price'])
    
    if 'description' in body:
        updates.append('description = :desc')
        expr_values[':desc'] = body['description']
    
    products_table.update_item(
        Key={'productId': product_id},
        UpdateExpression='SET ' + ', '.join(updates),
        ExpressionAttributeValues=expr_values
    )
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps({'message': 'Updated'})
    }

def delete_product(event, headers):
    """DELETE /products/{productId}"""
    product_id = event['pathParameters']['productId']
    
    # Delete product and inventory
    products_table.delete_item(Key={'productId': product_id})
    inventory_table.delete_item(Key={'productId': product_id})
    
    return {
        'statusCode': 204,
        'headers': headers,
        'body': ''
    }

def get_inventory(event, headers):
    """GET /inventory/{productId}"""
    product_id = event['pathParameters']['productId']
    
    response = inventory_table.get_item(Key={'productId': product_id})
    if 'Item' not in response:
        return {
            'statusCode': 404,
            'headers': headers,
            'body': json.dumps({'error': 'Not found'})
        }
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps(response['Item'])
    }

def update_inventory(event, headers):
    """PUT /inventory/{productId}"""
    product_id = event['pathParameters']['productId']
    body = json.loads(event.get('body', '{}'))
    
    quantity = int(body.get('quantity', 0))
    
    inventory_table.put_item(Item={
        'productId': product_id,
        'quantity': quantity,
        'updatedAt': datetime.now().isoformat()
    })
    
    return {
        'statusCode': 200,
        'headers': headers,
        'body': json.dumps({'quantity': quantity})
    }
```

## Error Handling Best Practices

```python
def lambda_handler(event, context):
    """Best practice error handling"""
    
    try:
        # Validate event structure
        if not event.get('httpMethod'):
            return error(400, 'Missing httpMethod')
        
        method = event['httpMethod']
        
        # Route handling
        if method == 'GET':
            return handle_get(event)
        
        elif method == 'POST':
            # Validate body
            try:
                body = json.loads(event.get('body', '{}'))
            except json.JSONDecodeError:
                return error(400, 'Invalid JSON in body')
            
            return handle_post(event, body)
        
        else:
            return error(405, f'Method {method} not allowed')
    
    except KeyError as e:
        print(f"KeyError: {str(e)}")
        return error(400, f'Missing parameter: {str(e)}')
    
    except ValueError as e:
        print(f"ValueError: {str(e)}")
        return error(400, f'Invalid value: {str(e)}')
    
    except Exception as e:
        print(f"Unexpected error: {str(e)}")
        import traceback
        traceback.print_exc()
        return error(500, 'Internal server error')

def error(status, message):
    """Standard error response"""
    return {
        'statusCode': status,
        'headers': {'Content-Type': 'application/json'},
        'body': json.dumps({'error': message})
    }
```

## Local Testing

Create local test file:

```python
# test_handler.py
import json
from handler import lambda_handler

def test_get_user():
    """Test GET user"""
    event = {
        'httpMethod': 'GET',
        'resource': '/users/{userId}',
        'pathParameters': {'userId': '123'},
        'headers': {}
    }
    
    response = lambda_handler(event, None)
    print(f"Response: {response}")

def test_create_user():
    """Test POST user"""
    event = {
        'httpMethod': 'POST',
        'resource': '/users',
        'body': json.dumps({
            'name': 'John Doe',
            'email': 'john@example.com'
        }),
        'headers': {}
    }
    
    response = lambda_handler(event, None)
    print(f"Response: {response}")

if __name__ == '__main__':
    test_get_user()
    test_create_user()
```

**Run locally**:
```bash
python test_handler.py
```

## Deployment Strategies

### Package Dependencies

```bash
# Create deployment package
mkdir lambda-deployment
cd lambda-deployment

# Copy handler
cp handler.py .

# Create requirements.txt
echo "boto3==1.26.0" > requirements.txt

# Install dependencies
pip install -r requirements.txt -t .

# Create zip
zip -r handler.zip .

# Deploy
aws lambda update-function-code \
    --function-name UserHandler \
    --zip-file fileb://handler.zip
```

### Using Layers for Dependencies

```bash
# Create layer directory
mkdir python
cd python
pip install -r requirements.txt -t .
cd ..

# Create layer zip
zip -r layer.zip python

# Publish layer
aws lambda publish-layer-version \
    --layer-name dependencies \
    --zip-file fileb://layer.zip \
    --compatible-runtimes python3.11
```

## Performance Optimization

### Connection Reuse

```python
# Initialize outside handler for reuse
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    # Reuse existing connection
    response = table.get_item(...)
```

### Batch Operations

```python
def batch_create_users(users):
    """Create multiple users efficiently"""
    with table.batch_writer(
        batch_size=25  # DynamoDB limit
    ) as batch:
        for user in users:
            batch.put_item(Item=user)

def lambda_handler(event, context):
    users = json.loads(event['body'])
    batch_create_users(users)
```

### Query Optimization

```python
def get_users_by_email_index(email):
    """Use index instead of scan"""
    response = table.query(
        IndexName='email-index',
        KeyConditionExpression='email = :email',
        ExpressionAttributeValues={':email': email}
    )
    return response['Items']
```

## Verification Checklist

- [ ] Router pattern correctly routes all HTTP methods
- [ ] Path and query parameters extracted correctly
- [ ] Request body parsed and validated
- [ ] Environment variables used for configuration
- [ ] CORS headers included in responses
- [ ] Error responses with appropriate status codes
- [ ] Batch operations used for efficiency
- [ ] Connection reuse for performance
- [ ] Index queries used instead of scans
- [ ] Local testing verified
- [ ] Deployment package includes dependencies
- [ ] Error handling covers JSON parsing errors
- [ ] Exception logging with traceback

## Next Steps

- [server_lab.md](server_lab.md): Complete hands-on lab with all patterns
