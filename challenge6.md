# Challenge 6: Serverless File Validator  

## **Objective**  
Validate files uploaded to an S3 bucket using AWS Lambda and trigger different SNS notifications based on the validation results.  

## **Instructions**  
1. **Configure an S3 bucket to trigger a Lambda function on file uploads.**  
2. **Write a Lambda function** to validate the file's format and size.  
3. If the file meets the criteria, move it to the `validated` folder in the same bucket.  
4. If the file fails validation, move it to the `failed` folder and send an SNS notification with the failure reason.  

---

## **Table of Contents**  
1. [Setup Procedure](#setup-procedure)  
   - [Create S3 Bucket](#1-create-s3-bucket)  
   - [Create SNS Topic](#2-create-sns-topic)  
   - [Create Lambda Function](#3-create-lambda-function)  
2. [Lambda Python Code](#lambda-python-code)  
3. [IAM Permissions for Lambda Role](#iam-permissions-for-lambda-role)  
4. [Testing the Solution](#testing-the-solution)  

---

## **Setup Procedure**  

### 1. **Create S3 Bucket**  
1. Go to **S3 Dashboard**.  
2. Click **Create Bucket**:  
   - **Bucket Name**: `challenge6herov`.  
   - Enable **Versioning** and **Server-Side Encryption** if needed.  
3. Go to the **Properties** tab and enable **Event Notifications**:  
   - Event Name: `FileUploadEvent`.  
   - Event Type: `PUT` (ObjectCreated).  
   - Destination: Choose **Lambda Function**.  

---

### 2. **Create SNS Topic**  
1. Go to **SNS Dashboard** > **Topics**.  
2. Click **Create Topic**:  
   - **Name**: `FileValidationNotifications`.  
   - Copy the **ARN** of the topic for later use.  
3. Create a subscription:  
   - **Protocol**: Email.  
   - **Endpoint**: Enter your email address.  
   - Confirm the subscription via the email received.  

---

### 3. **Create Lambda Function**  
1. Go to **Lambda Dashboard** and click **Create Function**.  
   - **Runtime**: Python 3.x.  
   - **Execution Role**: Choose or create a role with S3, SNS, and CloudWatch permissions.  

2. Add the following **Python code** in the Lambda function editor.  

---

## **Lambda Python Code**  

```python
import boto3
import json
import os

# Initialize AWS clients
s3 = boto3.client('s3')
sns = boto3.client('sns')

# Environment variables for SNS Topic ARN and Bucket Name
SNS_TOPIC_ARN = os.getenv('SNS_TOPIC_ARN')
BUCKET_NAME = os.getenv('BUCKET_NAME')

# Validation Criteria
MAX_FILE_SIZE = 5 * 1024 * 1024  # 5 MB
VALID_EXTENSIONS = ['.txt', '.csv', '.json']

def lambda_handler(event, context):
    # Extract S3 bucket and object key from event
    bucket = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    try:
        # Get file metadata
        response = s3.head_object(Bucket=bucket, Key=key)
        file_size = response['ContentLength']
        file_extension = key.split('.')[-1]
        
        # Validate file size
        if file_size > MAX_FILE_SIZE:
            move_file(bucket, key, 'failed/')
            send_notification(key, "File exceeds maximum size limit (5 MB).")
            return generate_response(400, f"File {key} is too large.")
        
        # Validate file extension
        if not any(key.endswith(ext) for ext in VALID_EXTENSIONS):
            move_file(bucket, key, 'failed/')
            send_notification(key, "Invalid file format. Allowed formats: .txt, .csv, .json.")
            return generate_response(400, f"Invalid file format for {key}.")
        
        # If validation passes, move to 'validated/' folder
        move_file(bucket, key, 'validated/')
        return generate_response(200, f"File {key} validated successfully.")
    
    except Exception as e:
        print(f"Error processing file {key}: {str(e)}")
        return generate_response(500, f"Error: {str(e)}")

def move_file(bucket, key, folder):
    """Moves file to the specified folder."""
    copy_source = {'Bucket': bucket, 'Key': key}
    new_key = folder + key.split('/')[-1]
    s3.copy_object(Bucket=bucket, CopySource=copy_source, Key=new_key)
    s3.delete_object(Bucket=bucket, Key=key)

def send_notification(file_name, reason):
    """Sends an SNS notification with the validation failure reason."""
    message = f"Validation failed for file {file_name}. Reason: {reason}"
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="File Validation Failed",
        Message=message
    )

def generate_response(status_code, message):
    """Generates a response for Lambda."""
    return {
        'statusCode': status_code,
        'body': json.dumps(message)
    }
```

---

## **IAM Permissions for Lambda Role**  
Ensure the Lambda execution role has the following permissions:  

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:CopyObject",
        "s3:DeleteObject",
        "sns:Publish"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## **Testing the Solution**  
1. **Upload a file to S3**:  
   - Upload files of varying sizes and formats to the root of the S3 bucket.  
   
2. **Check the Results**:  
   - Files meeting the criteria should be moved to the `validated/` folder.  
   - Invalid files should be moved to the `failed/` folder.  
   - Check your email for SNS notifications on validation failures.  

---

## **Conclusion**  
This setup provides a serverless solution to validate files uploaded to S3, ensuring only valid files are processed while notifying users of any issues.

---
