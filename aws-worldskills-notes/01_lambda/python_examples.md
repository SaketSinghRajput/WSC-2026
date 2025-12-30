# Lambda Python Development - Code Examples and Best Practices

## Lambda Handler Function

The handler is the entry point for Lambda function execution. Lambda calls this method when your function is invoked.

### Standard Handler Signature

```python
def lambda_handler(event, context):
    """
    AWS Lambda handler function
    
    Args:
        event (dict): Input data for the function (varies by trigger)
        context (object): Runtime information and methods
        
    Returns:
        dict: Response data (format depends on integration type)
    """
    # Your code here
    return response
```

### Event Object

The `event` parameter contains input data. Structure varies by trigger:

**API Gateway Event**: HTTP method, path, query parameters, headers, body  
**S3 Event**: Bucket name, object key, event type  
**EventBridge Event**: Time-based schedule or custom event data  
**SQS Event**: Array of messages with body and attributes

### Context Object

The `context` parameter provides runtime information:

**Common Properties**:
- `context.function_name`: Name of Lambda function
- `context.function_version`: Function version being executed
- `context.memory_limit_in_mb`: Memory allocated to function
- `context.request_id`: Unique request identifier for this invocation
- `context.aws_request_id`: AWS request ID (same as request_id)
- `context.log_group_name`: CloudWatch Logs group name
- `context.log_stream_name`: CloudWatch Logs stream name

**Useful Methods**:
- `context.get_remaining_time_in_millis()`: Milliseconds remaining before timeout

```python
def lambda_handler(event, context):
    print(f"Function: {context.function_name}")
    print(f"Memory: {context.memory_limit_in_mb}MB")
    print(f"Request ID: {context.request_id}")
    print(f"Time remaining: {context.get_remaining_time_in_millis()}ms")
    return {'statusCode': 200}
```

## Basic Hello World Example

Minimal Lambda function returning JSON response:

```python
def lambda_handler(event, context):
    """
    Basic Hello World Lambda function
    Returns simple JSON response
    """
    return {
        'statusCode': 200,
        'body': 'Hello from Lambda!'
    }
```

**Testing**: Create test event in Lambda console with empty JSON `{}`

**Expected Output**:
```json
{
  "statusCode": 200,
  "body": "Hello from Lambda!"
}
```

## Processing API Gateway Events

API Gateway Lambda proxy integration sends HTTP request details in the event object.

### API Gateway Event Structure

```python
{
    "httpMethod": "POST",
    "path": "/users",
    "queryStringParameters": {"filter": "active"},
    "pathParameters": {"userId": "123"},
    "headers": {
        "Content-Type": "application/json",
        "Authorization": "Bearer token123"
    },
    "body": "{\"name\":\"John\",\"email\":\"john@example.com\"}"
}
```

### Extracting HTTP Method

```python
def lambda_handler(event, context):
    http_method = event.get('httpMethod', 'UNKNOWN')
    
    if http_method == 'GET':
        return handle_get(event)
    elif http_method == 'POST':
        return handle_post(event)
    elif http_method == 'PUT':
        return handle_put(event)
    elif http_method == 'DELETE':
        return handle_delete(event)
    else:
        return {
            'statusCode': 405,
            'body': 'Method Not Allowed'
        }
```

### Extracting Path Parameters

```python
def lambda_handler(event, context):
    # Path: /users/{userId}
    path_params = event.get('pathParameters', {})
    user_id = path_params.get('userId')
    
    if not user_id:
        return {
            'statusCode': 400,
            'body': 'Missing userId parameter'
        }
    
    return {
        'statusCode': 200,
        'body': f'User ID: {user_id}'
    }
```

### Extracting Query String Parameters

```python
def lambda_handler(event, context):
    # URL: /users?filter=active&limit=10
    query_params = event.get('queryStringParameters', {})
    filter_value = query_params.get('filter', 'all')
    limit = int(query_params.get('limit', 20))
    
    return {
        'statusCode': 200,
        'body': f'Filter: {filter_value}, Limit: {limit}'
    }
```

### Parsing Request Body

```python
import json

def lambda_handler(event, context):
    try:
        # API Gateway sends body as string, must parse JSON
        body = json.loads(event.get('body', '{}'))
        name = body.get('name')
        email = body.get('email')
        
        if not name or not email:
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'Missing required fields'})
            }
        
        # Process data
        result = {
            'message': f'User {name} created successfully',
            'email': email
        }
        
        return {
            'statusCode': 201,
            'body': json.dumps(result)
        }
    except json.JSONDecodeError:
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Invalid JSON'})
        }
```

### Complete API Gateway Handler with CORS

```python
import json

def lambda_handler(event, context):
    """
    Complete API Gateway handler with CORS support
    Handles POST requests for user registration
    """
    # Enable CORS
    headers = {
        'Access-Control-Allow-Origin': '*',
        'Access-Control-Allow-Headers': 'Content-Type,Authorization',
        'Access-Control-Allow-Methods': 'POST,OPTIONS'
    }
    
    # Handle preflight OPTIONS request
    if event.get('httpMethod') == 'OPTIONS':
        return {
            'statusCode': 200,
            'headers': headers,
            'body': ''
        }
    
    try:
        # Parse request body
        body = json.loads(event.get('body', '{}'))
        name = body.get('name', '').strip()
        email = body.get('email', '').strip()
        
        # Validate input
        if not name:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'Name is required'})
            }
        
        if not email or '@' not in email:
            return {
                'statusCode': 400,
                'headers': headers,
                'body': json.dumps({'error': 'Valid email is required'})
            }
        
        # Process registration (simplified)
        result = {
            'message': 'Registration successful',
            'user': {
                'name': name,
                'email': email
            }
        }
        
        return {
            'statusCode': 201,
            'headers': headers,
            'body': json.dumps(result)
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'headers': headers,
            'body': json.dumps({'error': 'Internal server error'})
        }
```

## Working with S3 Events

S3 events trigger Lambda when objects are created, deleted, or modified.

### S3 Event Structure

```python
{
    "Records": [
        {
            "eventName": "ObjectCreated:Put",
            "s3": {
                "bucket": {
                    "name": "my-upload-bucket"
                },
                "object": {
                    "key": "uploads/image.jpg",
                    "size": 1024
                }
            }
        }
    ]
}
```

### Basic S3 Event Handler

```python
def lambda_handler(event, context):
    """
    Process S3 event when object is uploaded
    """
    for record in event['Records']:
        bucket_name = record['s3']['bucket']['name']
        object_key = record['s3']['object']['key']
        event_name = record['eventName']
        
        print(f"Event: {event_name}")
        print(f"Bucket: {bucket_name}")
        print(f"Object: {object_key}")
    
    return {'statusCode': 200}
```

### Reading S3 Object Content

```python
import boto3
import json

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    """
    Read S3 object content and process
    """
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        try:
            # Download object from S3
            response = s3_client.get_object(Bucket=bucket, Key=key)
            content = response['Body'].read().decode('utf-8')
            
            print(f"File content length: {len(content)} bytes")
            
            # Process content (example: count lines)
            lines = content.split('\n')
            print(f"Number of lines: {len(lines)}")
            
        except Exception as e:
            print(f"Error processing {key}: {str(e)}")
            raise
    
    return {'statusCode': 200}
```

### CSV Processing Example

```python
import boto3
import csv
import io

s3_client = boto3.client('s3')
dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('UserData')

def lambda_handler(event, context):
    """
    Parse CSV file from S3 and insert records into DynamoDB
    """
    for record in event['Records']:
        bucket = record['s3']['bucket']['name']
        key = record['s3']['object']['key']
        
        # Only process CSV files
        if not key.endswith('.csv'):
            print(f"Skipping non-CSV file: {key}")
            continue
        
        try:
            # Download CSV from S3
            response = s3_client.get_object(Bucket=bucket, Key=key)
            content = response['Body'].read().decode('utf-8')
            
            # Parse CSV
            csv_reader = csv.DictReader(io.StringIO(content))
            
            # Insert each row into DynamoDB
            for row in csv_reader:
                table.put_item(Item={
                    'id': row['id'],
                    'name': row['name'],
                    'email': row['email']
                })
                print(f"Inserted: {row['name']}")
            
            print(f"Successfully processed {key}")
            
        except Exception as e:
            print(f"Error processing {key}: {str(e)}")
            raise
    
    return {'statusCode': 200}
```

## Environment Variables

Environment variables configure Lambda functions without hardcoding values.

### Using Environment Variables

```python
import os

def lambda_handler(event, context):
    # Read environment variables
    table_name = os.environ.get('TABLE_NAME', 'DefaultTable')
    api_key = os.environ.get('API_KEY')
    max_items = int(os.environ.get('MAX_ITEMS', '100'))
    
    print(f"Using table: {table_name}")
    print(f"Max items: {max_items}")
    
    # Never log secrets
    if api_key:
        print("API key is configured")
    
    return {'statusCode': 200}
```

### Setting Environment Variables (AWS Console)

1. Open Lambda function in console
2. Navigate to "Configuration" tab
3. Select "Environment variables"
4. Click "Edit"
5. Add key-value pairs:
   - `TABLE_NAME`: `UserRegistrations`
   - `MAX_ITEMS`: `100`
6. Click "Save"

### Setting Environment Variables (AWS CLI)

```bash
aws lambda update-function-configuration \
    --function-name MyFunction \
    --environment "Variables={TABLE_NAME=UserRegistrations,MAX_ITEMS=100}"
```

## Error Handling

Proper error handling ensures functions are resilient and provide useful debugging information.

### Basic Try-Except Pattern

```python
def lambda_handler(event, context):
    try:
        # Main logic
        result = process_data(event)
        
        return {
            'statusCode': 200,
            'body': json.dumps(result)
        }
    except ValueError as e:
        # Handle validation errors
        print(f"Validation error: {str(e)}")
        return {
            'statusCode': 400,
            'body': json.dumps({'error': 'Invalid input'})
        }
    except Exception as e:
        # Handle unexpected errors
        print(f"Unexpected error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }
```

### Returning Proper HTTP Status Codes

```python
import json

def lambda_handler(event, context):
    try:
        body = json.loads(event.get('body', '{}'))
        user_id = body.get('userId')
        
        if not user_id:
            # Bad Request
            return {
                'statusCode': 400,
                'body': json.dumps({'error': 'userId is required'})
            }
        
        user = get_user(user_id)
        
        if not user:
            # Not Found
            return {
                'statusCode': 404,
                'body': json.dumps({'error': 'User not found'})
            }
        
        # Success
        return {
            'statusCode': 200,
            'body': json.dumps(user)
        }
        
    except Exception as e:
        # Internal Server Error
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps({'error': 'Internal server error'})
        }
```

## Logging Best Practices

CloudWatch Logs automatically captures all `print()` output from Lambda functions.

### Basic Logging

```python
def lambda_handler(event, context):
    print(f"Received event: {event}")
    print(f"Function name: {context.function_name}")
    
    # Process event
    result = process_event(event)
    
    print(f"Processing complete: {result}")
    return result
```

### Structured Logging with JSON

```python
import json
import time

def log_event(level, message, **kwargs):
    """
    Log structured event in JSON format for easier parsing
    """
    log_entry = {
        'timestamp': time.time(),
        'level': level,
        'message': message,
        **kwargs
    }
    print(json.dumps(log_entry))

def lambda_handler(event, context):
    log_event('INFO', 'Function invoked', function=context.function_name)
    
    try:
        result = process_data(event)
        log_event('INFO', 'Processing successful', result=result)
        return result
    except Exception as e:
        log_event('ERROR', 'Processing failed', error=str(e))
        raise
```

### Logging Sensitive Data

**Never log sensitive information**:
- Passwords
- API keys
- Credit card numbers
- Personal identification information

```python
def lambda_handler(event, context):
    # BAD: Logs API key
    api_key = os.environ['API_KEY']
    print(f"Using API key: {api_key}")  # NEVER DO THIS
    
    # GOOD: Confirms presence without logging value
    if os.environ.get('API_KEY'):
        print("API key configured")
```

## boto3 SDK Examples

boto3 is the AWS SDK for Python, used to interact with AWS services from Lambda.

### S3 Operations

```python
import boto3

s3_client = boto3.client('s3')

def lambda_handler(event, context):
    bucket = 'my-bucket'
    key = 'data/file.txt'
    
    # Upload object
    s3_client.put_object(
        Bucket=bucket,
        Key=key,
        Body='Hello from Lambda',
        ContentType='text/plain'
    )
    
    # Download object
    response = s3_client.get_object(Bucket=bucket, Key=key)
    content = response['Body'].read().decode('utf-8')
    print(f"Content: {content}")
    
    # List objects
    response = s3_client.list_objects_v2(Bucket=bucket, Prefix='data/')
    for obj in response.get('Contents', []):
        print(f"Object: {obj['Key']}")
    
    # Delete object
    s3_client.delete_object(Bucket=bucket, Key=key)
    
    return {'statusCode': 200}
```

### DynamoDB Operations

```python
import boto3
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
table = dynamodb.Table('Users')

def lambda_handler(event, context):
    # Put item (create/update)
    table.put_item(
        Item={
            'userId': '123',
            'name': 'John Doe',
            'email': 'john@example.com',
            'age': 30
        }
    )
    
    # Get item (read)
    response = table.get_item(Key={'userId': '123'})
    user = response.get('Item')
    print(f"User: {user}")
    
    # Update item
    table.update_item(
        Key={'userId': '123'},
        UpdateExpression='SET age = :age',
        ExpressionAttributeValues={':age': 31}
    )
    
    # Query items
    response = table.query(
        KeyConditionExpression='userId = :id',
        ExpressionAttributeValues={':id': '123'}
    )
    items = response['Items']
    
    # Scan table (less efficient than query)
    response = table.scan(
        FilterExpression='age > :min_age',
        ExpressionAttributeValues={':min_age': 25}
    )
    filtered_items = response['Items']
    
    # Delete item
    table.delete_item(Key={'userId': '123'})
    
    return {'statusCode': 200}
```

### SNS Publish

```python
import boto3
import json

sns_client = boto3.client('sns')

def lambda_handler(event, context):
    topic_arn = 'arn:aws:sns:us-east-1:123456789012:MyTopic'
    
    # Publish simple message
    sns_client.publish(
        TopicArn=topic_arn,
        Subject='Lambda Notification',
        Message='Processing complete'
    )
    
    # Publish JSON message
    message_data = {
        'event': 'user_registered',
        'userId': '123',
        'timestamp': '2025-01-01T10:00:00Z'
    }
    
    sns_client.publish(
        TopicArn=topic_arn,
        Subject='User Registration',
        Message=json.dumps(message_data)
    )
    
    return {'statusCode': 200}
```

## Response Format for API Gateway

API Gateway proxy integration requires specific response format.

### Required Response Structure

```python
{
    "statusCode": 200,                 # HTTP status code (required)
    "headers": {                       # HTTP headers (optional)
        "Content-Type": "application/json",
        "Access-Control-Allow-Origin": "*"
    },
    "body": "{\"message\":\"Success\"}" # Response body as STRING (required)
}
```

### Complete Example

```python
import json

def lambda_handler(event, context):
    response_data = {
        'message': 'User created successfully',
        'userId': '123',
        'timestamp': '2025-01-01T10:00:00Z'
    }
    
    return {
        'statusCode': 201,
        'headers': {
            'Content-Type': 'application/json',
            'Access-Control-Allow-Origin': '*',
            'Access-Control-Allow-Methods': 'POST,OPTIONS'
        },
        'body': json.dumps(response_data)  # Must stringify JSON
    }
```

**Common Mistake**: Returning dictionary instead of JSON string for body
```python
# WRONG
return {
    'statusCode': 200,
    'body': {'message': 'Success'}  # Dict instead of string
}

# CORRECT
return {
    'statusCode': 200,
    'body': json.dumps({'message': 'Success'})  # JSON string
}
```

## Testing Locally

### Creating Test Events in Lambda Console

1. Open Lambda function in console
2. Click "Test" tab
3. Click "Create new event"
4. Enter event name (e.g., "TestAPIGatewayEvent")
5. Paste test event JSON
6. Click "Save"
7. Click "Test" to invoke function

### Sample API Gateway Test Event

```json
{
  "httpMethod": "POST",
  "path": "/users",
  "headers": {
    "Content-Type": "application/json"
  },
  "body": "{\"name\":\"John Doe\",\"email\":\"john@example.com\"}"
}
```

### Sample S3 Test Event

```json
{
  "Records": [
    {
      "eventName": "ObjectCreated:Put",
      "s3": {
        "bucket": {
          "name": "my-test-bucket"
        },
        "object": {
          "key": "uploads/test.txt",
          "size": 1024
        }
      }
    }
  ]
}
```

## AWS CLI Commands

### Create Lambda Function

```bash
# Zip function code
zip function.zip lambda_function.py

# Create function
aws lambda create-function \
    --function-name MyFunction \
    --runtime python3.11 \
    --role arn:aws:iam::123456789012:role/LambdaExecutionRole \
    --handler lambda_function.lambda_handler \
    --zip-file fileb://function.zip \
    --timeout 30 \
    --memory-size 128
```

### Update Function Code

```bash
# Zip updated code
zip function.zip lambda_function.py

# Update function
aws lambda update-function-code \
    --function-name MyFunction \
    --zip-file fileb://function.zip
```

### Invoke Function

```bash
# Synchronous invocation
aws lambda invoke \
    --function-name MyFunction \
    --payload '{"key":"value"}' \
    response.json

# View response
cat response.json
```

### View Logs

```bash
# Tail logs in real-time
aws logs tail /aws/lambda/MyFunction --follow

# Get recent logs
aws logs tail /aws/lambda/MyFunction --since 1h
```

## Next Steps

After mastering Python Lambda development:
- [permissions_iam.md](permissions_iam.md): Configure IAM roles for each example
- [serverless_lab.md](serverless_lab.md): Build complete API with these patterns
- [cost_optimization.md](cost_optimization.md): Optimize code for performance and cost
