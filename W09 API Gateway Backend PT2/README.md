# Week 09

# File Management with DynamoDB and Lambda

## Step 1: Set Up Your DynamoDB Table

### 1.1 Create the DynamoDB Table

1. Navigate to the DynamoDB service in your AWS console
2. Click "Create table"
3. Table name: `your-name-file-storage-lab`
4. Partition key: `filename` (String)
5. Leave sort key empty
6. Use default settings for the rest
7. Click "Create table"

### 1.2 Note Your Table Configuration

Once created, note down:

- Table name (you'll need this in your Lambda functions)
- Table ARN (visible in the table overview)

## Step 2: Create Lambda Functions

### 2.1 Create File Upload Lambda Function

1. Navigate to Lambda service
2. Click "Create function"
3. Choose "Author from scratch"
4. Function name: `file-upload-function`
5. Runtime: Python 3.13 or later
6. Under "Change default execution role," select "Use an existing role"
7. Choose **LabRole** from the dropdown
8. Click "Create function"

### 2.2 Upload Function Code

1. Replace the default code with:

```python
import json
import boto3
import base64
from botocore.exceptions import ClientError
from datetime import datetime

dynamodb = boto3.resource('dynamodb')
TABLE_NAME = 'your-table-name-here'  # Replace with your table name
table = dynamodb.Table(TABLE_NAME)

def lambda_handler(event, context):
    try:
        # Parse the request body
        body = json.loads(event['body']) if event.get('body') else {}

        # Get filename and file content
        filename = body.get('filename')
        file_content = body.get('content')

        if not filename or not file_content:
            return {
                'statusCode': 400,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': 'Missing filename or content'
                })
            }

        # Validate base64 content
        try:
            decoded_content = base64.b64decode(file_content)
            # Re-encode to ensure it's valid base64
            validated_content = base64.b64encode(decoded_content).decode('utf-8')
        except:
            return {
                'statusCode': 400,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': 'Invalid base64 content'
                })
            }

        # Store in DynamoDB
        table.put_item(
            Item={
                'filename': filename,
                'content': validated_content,
                'size': len(decoded_content),
                'uploaded_at': datetime.now().isoformat(),
                'content_type': 'application/octet-stream'
            }
        )

        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'message': f'File {filename} uploaded successfully'
            })
        }

    except ClientError as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'DynamoDB error: {str(e)}'
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'Unexpected error: {str(e)}'
            })
        }
```

**Important:** Replace `your-table-name-here` with your actual table name.

2. Then click the "Deploy" button

### 2.3 Create File Download Lambda Function

1. Create another function named `file-download-function`
2. Use the same configuration as the upload function (select **LabRole**)
3. Replace the code with:

```python
import json
import boto3
from botocore.exceptions import ClientError
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
TABLE_NAME = 'your-table-name-here'  # Replace with your table name
table = dynamodb.Table(TABLE_NAME)

def decimal_default(obj):
    """Convert Decimal objects to int or float for JSON serialization"""
    if isinstance(obj, Decimal):
        return int(obj) if obj % 1 == 0 else float(obj)
    raise TypeError

def lambda_handler(event, context):
    try:
        # Get filename from path parameters
        filename = event['pathParameters']['filename'] if event.get('pathParameters') else None

        if not filename:
            return {
                'statusCode': 400,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': 'Missing filename parameter'
                })
            }

        # Get item from DynamoDB
        response = table.get_item(
            Key={'filename': filename}
        )

        if 'Item' not in response:
            return {
                'statusCode': 404,
                'headers': {
                    'Content-Type': 'application/json',
                    'Access-Control-Allow-Origin': '*'
                },
                'body': json.dumps({
                    'error': f'File {filename} not found'
                })
            }

        item = response['Item']

        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'filename': item['filename'],
                'content': item['content'],
                'content_type': item.get('content_type', 'application/octet-stream'),
                'size': item.get('size', 0),
                'uploaded_at': item.get('uploaded_at', '')
            }, default=decimal_default)
        }

    except ClientError as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'DynamoDB error: {str(e)}'
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'Unexpected error: {str(e)}'
            })
        }
```

**Important:** Replace `your-table-name-here` with your actual table name.

4. Then click the "Deploy" button

### 2.4 Create File List Lambda Function

1. Create a third function named `file-list-function` using **LabRole** with this code:

```python
import json
import boto3
from botocore.exceptions import ClientError
from decimal import Decimal

dynamodb = boto3.resource('dynamodb')
TABLE_NAME = 'your-table-name-here'  # Replace with your table name
table = dynamodb.Table(TABLE_NAME)

def decimal_default(obj):
    """Convert Decimal objects to int or float for JSON serialization"""
    if isinstance(obj, Decimal):
        return int(obj) if obj % 1 == 0 else float(obj)
    raise TypeError

def lambda_handler(event, context):
    try:
        # Scan the table to get all files
        response = table.scan(
            ProjectionExpression='filename, #size, uploaded_at, content_type',
            ExpressionAttributeNames={
                '#size': 'size'  # 'size' is a reserved keyword in DynamoDB
            }
        )

        files = []
        for item in response['Items']:
            files.append({
                'filename': item['filename'],
                'size': item.get('size', 0),  # This will be a Decimal object
                'uploaded_at': item.get('uploaded_at', ''),
                'content_type': item.get('content_type', 'application/octet-stream')
            })

        # Sort files by filename for consistent ordering
        files.sort(key=lambda x: x['filename'])

        return {
            'statusCode': 200,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'files': files
            }, default=decimal_default)
        }

    except ClientError as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'DynamoDB error: {str(e)}'
            })
        }
    except Exception as e:
        return {
            'statusCode': 500,
            'headers': {
                'Content-Type': 'application/json',
                'Access-Control-Allow-Origin': '*'
            },
            'body': json.dumps({
                'error': f'Unexpected error: {str(e)}'
            })
        }
```

**Important:** Replace `your-table-name-here` with your actual table name.

2. Then click the "Deploy" button

## Step 3: Create API Gateway

### 3.1 Create REST API

1. Navigate to API Gateway service
2. Click "Create API"
3. Choose "REST API" (not private)
4. Click "Build"
5. Choose "New API"
6. API name: `file-management-api`
7. Description: `API for file upload and download operations`
8. Click "Create API"

### 3.2 Create Resources and Methods

#### Create Upload Endpoint

1. Click "Create Resource"
2. Resource Name: `upload`
3. Click "Create Resource"

4. Select the `/upload` resource
5. Click "Create Method"
6. Choose "POST" from dropdown
7. Integration type: Lambda Function
8. Use Lambda Proxy integration
9. Lambda Region: Your region
10. Lambda Function: `file-upload-function`
11. Click "Create Method"

#### Create Download Endpoint

1. Click "Create Resource"
2. Resource Name: `download`
3. Resource Path: `/`
4. Click "Create Resource"

5. Select the `/download` resource
6. Click "Create Resource"
7. Resource Name: `{filename}`
8. Resource Path: `/download`
9. Click "Create Resource"

10. Select the `/{filename}` resource
11. Click "Create Method"
12. Choose "GET"
13. Integration type: Lambda Function
14. Use Lambda Proxy integration
15. Lambda Function: `file-download-function`
16. Click "Create Method"

#### Create List Files Endpoint

1. Click "Actions" → "Create Resource"
2. Resource Name: `files`
3. Resource Path: `/`
4. Click "Create Resource"

5. Select the `/files` resource
6. Click "Create Method"
7. Choose "GET"
8. Integration type: Lambda Function
9. Use Lambda Proxy integration
10. Lambda Function: `file-list-function`
11. Click "Create Method"

### 3.3 Enable CORS

For each method (POST /upload, GET /download/{filename}, GET /files):

1. Select the method
2. Click "Actions" → "Enable CORS"
3. Keep default settings
4. Click "Save"

### 3.4 Deploy the API

1. Click "Deploy API"
2. Deployment stage: New Stage
3. Stage name: `prod`
4. Click "Deploy"
5. Copy the Invoke URL

## Step 4: Test Your API

Go to [http://test-invoke-url-rgm.s3-website-us-west-2.amazonaws.com/](http://test-invoke-url-rgm.s3-website-us-west-2.amazonaws.com/)

## Step 5: Monitor and Debug

### 5.1 Check CloudWatch Logs

1. Navigate to CloudWatch
2. Click "Log groups"
3. Find logs for your Lambda functions (`/aws/lambda/function-name`)
4. Review execution logs for any errors

### 5.2 Check DynamoDB Items

1. Navigate to DynamoDB console
2. Select your table
3. Click "Explore table items" to view stored files

### 5.3 Test in API Gateway Console

1. Go to API Gateway console
2. Select your API
3. Choose a method
4. Click "TEST"
5. For POST /upload, use this test body:

```json
{
  "filename": "test.txt",
  "content": "SGVsbG8gV29ybGQhCg=="
}
```

## Troubleshooting Common Issues

**Lambda timeout errors:** Increase timeout in Lambda function configuration (Configuration → General configuration → Edit → Timeout)

**Permission errors:** The LabRole should have sufficient permissions for DynamoDB, but if you encounter issues, check CloudWatch logs for specific error messages

**CORS errors:** Ensure CORS is enabled on all methods and the OPTIONS method is created automatically

**Base64 encoding issues:** Make sure file content is properly base64 encoded for upload. You can use online base64 encoders or command line tools

**API Gateway Integration:** Make sure "Use Lambda Proxy integration" is checked when setting up the methods

**DynamoDB Item Size Limits:** DynamoDB has a 400KB limit per item. For larger files, consider using S3 with DynamoDB storing metadata only

**Reserved Keywords:** If you encounter errors with attribute names like 'size', use ExpressionAttributeNames to avoid conflicts

## Extension Ideas

- Add file deletion functionality (DELETE method with table.delete_item())
- Implement file type validation in Lambda
- Add file size limits and validation
- Store only metadata in DynamoDB and actual files in S3 for larger files
- Add file versioning using sort keys
- Implement file search functionality using DynamoDB queries
- Add user authentication and file ownership

## Notes

- **File Size:** DynamoDB has a 400KB limit per item.
- **Cost:** DynamoDB charges for read/write operations and storage. For frequently accessed large files, S3 is cost-effective.
- **Performance:** DynamoDB provides consistent single-digit millisecond performance, which is excellent for small file retrieval.
