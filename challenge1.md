# Challenge 1: Automated Image Processing

## **Objective**  
Create a serverless solution to resize images uploaded to an S3 bucket. The Lambda function will:  
1. Automatically resize images into multiple sizes (thumbnail, medium, large).  
2. Save resized images in corresponding folders in the same S3 bucket.  
3. Send an email notification using SNS to inform users about the new images.

---

## **Step-by-Step Implementation**

### **Step 1: Set Up S3 Bucket**

1. **Create an S3 Bucket:**
   - Go to the AWS Management Console > S3.
   - Click **Create Bucket**.
   - Name the bucket (e.g., `image-processing-bucket`).
   - Choose the appropriate region and click **Create Bucket**.

2. **Create Folders for Resized Images:**
   - Go to the S3 bucket and click **Create Folder**.
   - Create the following folders:
     - `thumbnail/`
     - `medium/`
     - `large/`

---

### **Step 2: Create an SNS Topic for Notifications**

1. **Go to the SNS Console:**
   - Click **Topics** > **Create Topic**.
   - Name the topic (e.g., `ImageProcessingNotifications`).

2. **Subscribe an Email Address:**
   - Click **Create Subscription**.
   - Protocol: `Email`.
   - Endpoint: Your email address.
   - Confirm the subscription via the email you receive.

3. **Note the SNS Topic ARN.**

---

### **Step 3: Create an IAM Role for Lambda**

1. **Go to the IAM Console:**
   - Click **Roles** > **Create Role**.
   - Choose **AWS Service** > **Lambda**.

2. **Attach the Following Policies:**
   - `AmazonS3FullAccess`
   - `AmazonSNSFullAccess`
   - `AWSLambdaBasicExecutionRole`

3. **Name the Role** (e.g., `LambdaImageProcessingRole`) and click **Create Role**.

---

### **Step 4: Create the Lambda Function**

1. **Go to the AWS Lambda Console:**
   - Click **Create Function**.
   - Choose **Author from Scratch**.
   - Name the function (e.g., `ImageResizerFunction`).
   - Runtime: `Python 3.x`.
   - Execution Role: Choose the role created in Step 3.

2. **Install Dependencies Locally:**
   - Install `PIL` (Pillow library) locally using the following commands:
     ```bash
     mkdir lambda-image-resizer
     cd lambda-image-resizer
     pip install Pillow -t .
     ```

3. **Upload Dependencies and Lambda Function Code:**
   - Create a ZIP file of the contents:
     ```bash
     zip -r lambda_image_resizer.zip .
     ```
   - Upload the `lambda_image_resizer.zip` to the Lambda function.

---

### **Step 5: Lambda Function Code**

```python
import boto3
import os
from PIL import Image
from io import BytesIO

s3 = boto3.client('s3')
sns = boto3.client('sns')

# Replace with your SNS topic ARN
SNS_TOPIC_ARN = 'arn:aws:sns:your-region:account-id:ImageProcessingNotifications'

def resize_image(image_path, size):
    with Image.open(image_path) as img:
        img.thumbnail(size)
        buffer = BytesIO()
        img.save(buffer, 'JPEG')
        buffer.seek(0)
        return buffer

def lambda_handler(event, context):
    bucket_name = event['Records'][0]['s3']['bucket']['name']
    key = event['Records'][0]['s3']['object']['key']
    
    # Download the image from S3
    download_path = '/tmp/original.jpg'
    s3.download_file(bucket_name, key, download_path)
    
    sizes = {
        'thumbnail': (100, 100),
        'medium': (300, 300),
        'large': (800, 800)
    }
    
    for size_name, dimensions in sizes.items():
        resized_buffer = resize_image(download_path, dimensions)
        resized_key = f"{size_name}/{key}"
        
        s3.upload_fileobj(
            resized_buffer,
            bucket_name,
            resized_key,
            ExtraArgs={'ContentType': 'image/jpeg'}
        )
    
    # Send SNS Notification
    sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject='Image Processing Complete',
        Message=f'Images have been resized and saved in {bucket_name}'
    )
    
    return {
        'statusCode': 200,
        'body': f'Images resized and uploaded to {bucket_name}'
    }
