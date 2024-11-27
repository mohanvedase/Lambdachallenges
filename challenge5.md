Here’s a properly formatted **Markdown (MD)** documentation for **Challenge 5: Real-time Log Monitoring**:

---

# Challenge 5: Real-time Log Monitoring (Beginner)

## Objective  
Create an AWS Lambda function to monitor logs stored in an S3 bucket, detect specific error patterns, and send alerts via SNS.

---

## Steps

### 1. **Create an IAM Role for Lambda**  
Ensure the role has the following permissions:
- **AmazonS3ReadOnlyAccess**: To read the log files from S3.
- **AmazonSNSFullAccess**: To publish messages to SNS.

### 2. **Create an S3 Bucket**  
1. Go to the **S3 Console**.
2. Create a bucket to store log files (e.g., `log-monitoring-bucket`).
3. Upload a sample log file (`logs.txt`) to the bucket.

### 3. **Create an SNS Topic**  
1. Go to the **SNS Console**.
2. Create a new **SNS Topic**:
   - **Topic Name**: `LogErrorAlerts`
3. Add an email subscription to the topic:
   - Enter your email address and confirm the subscription via the email sent by SNS.

### 4. **Create the Lambda Function**  
1. Open the **Lambda Console**.
2. Create a new function:
   - **Name**: `challenge5`
   - **Runtime**: Python 3.x
   - **Role**: Choose the IAM role created in Step 1.
3. Add **S3** as a **trigger**:
   - Choose the S3 bucket created earlier (`log-monitoring-bucket`).
   - Select **All Object Create Events**.

### 5. **Lambda Function Code**

Paste the following code into the **Lambda Function Editor**:

```python
import boto3
import json

def lambda_handler(event, context):
    # Initialize AWS clients
    s3 = boto3.client('s3')
    sns = boto3.client('sns')
    
    # Get S3 bucket and object details from the event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download the log file from S3
    response = s3.get_object(Bucket=bucket, Key=key)
    log_content = response['Body'].read().decode('utf-8')
    
    # Error detection patterns
    error_patterns = ['ERROR', 'Exception', 'CRITICAL', 'FATAL']
    
    # Find errors in the log file
    found_errors = []
    for line in log_content.split('\n'):
        for pattern in error_patterns:
            if pattern in line:
                found_errors.append(line)
    
    # If errors are found, send an SNS notification
    if found_errors:
        message = f"Errors detected in log file {key}:\n" + "\n".join(found_errors[:10])
        sns.publish(
            TopicArn='arn:aws:sns:ap-southeast-2:975050024946:batch8:dd1ff5f9-b56f-42b9-b689-f8deb5c7178b',
            Subject='Log Error Alert',
            Message=message
        )
    
    return {
        'statusCode': 200,
        'body': json.dumps(f'Processed {key}. Errors found: {len(found_errors)}')
    }
```

### 6. **Test the Lambda Function**
1. In the Lambda Console, click **Test**.
2. Create a new test event with the following sample event data:

```json
{
  "Records": [
    {
      "s3": {
        "bucket": {
          "name": "log-monitoring-bucket"
        },
        "object": {
          "key": "logs.txt.txt"
        }
      }
    }
  ]
}
```
3. Run the test. If the log file contains any of the error patterns (`ERROR`, `Exception`, `CRITICAL`, `FATAL`), you will receive an email notification via SNS.

---

## Expected Output
- If errors are detected, the Lambda function sends an email alert via SNS with the first 10 error lines.
- The output in the Lambda execution console should look similar to this:

```json
{
  "statusCode": 200,
  "body": "\"Processed logs.txt. Errors found: X\""
}
```

Where `X` is the number of errors detected.

### Example Email Notification:
```
Subject: Log Error Alert

Errors detected in log file logs.txt:
ERROR: Something went wrong
Exception: Failed to connect
...
```
### Output
![alt text](image-1.png)



This documentation should provide a clear guide for configuring the system and testing it effectively.