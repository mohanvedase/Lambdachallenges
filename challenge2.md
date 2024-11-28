
# Challenge 2: Scheduled EC2 Backup to S3

## **Objective**  
Automate daily backups of an EC2 instance by:  
1. Creating an AMI snapshot of the instance.  
2. Copying the AMI snapshot to an S3 bucket.  
3. Sending an email notification upon successful backup using SNS.

---

## **Step-by-Step Implementation**

### **Step 1: Set Up an S3 Bucket**

1. **Create a Backup S3 Bucket:**
   - Go to the AWS Management Console > S3.
   - Click **Create Bucket**.
   - Name the bucket (e.g., `ec2-backup-bucket`).
   - Choose the appropriate region and click **Create Bucket**.
   
---

### **Step 2: Create an SNS Topic for Notifications**

1. **Go to the SNS Console:**
   - Click **Topics** > **Create Topic**.
   - Name the topic (e.g., `EC2BackupNotifications`).

2. **Subscribe an Email Address:**
   - Click **Create Subscription**.
   - Protocol: `Email`.
   - Endpoint: Enter your email address.
   - Confirm the subscription via the email you receive.

3. **Note the SNS Topic ARN.**

---

### **Step 3: Create an IAM Role for Lambda**

1. **Go to the IAM Console:**
   - Click **Roles** > **Create Role**.
   - Choose **AWS Service** > **Lambda**.

2. **Attach the Following Policies:**
   - `AmazonEC2FullAccess`
   - `AmazonS3FullAccess`
   - `AmazonSNSFullAccess`
   - `AWSLambdaBasicExecutionRole`

3. **Name the Role** (e.g., `LambdaEC2BackupRole`) and click **Create Role**.

---

### **Step 4: Create the Lambda Function**

1. **Go to the AWS Lambda Console:**
   - Click **Create Function**.
   - Choose **Author from Scratch**.
   - Name the function (e.g., `EC2BackupFunction`).
   - Runtime: `Python 3.x`.
   - Execution Role: Choose the IAM role created in Step 3.

2. **Write the Lambda Function Code:**
   Modify the placeholders in the code with your specific values (e.g., EC2 instance ID, S3 bucket name, SNS topic ARN).

```python
import boto3
import datetime

ec2 = boto3.client('ec2')
s3 = boto3.client('s3')
sns = boto3.client('sns')

# Replace with your SNS topic ARN
SNS_TOPIC_ARN = 'arn:aws:sns:your-region:account-id:EC2BackupNotifications'

# Replace with your S3 bucket name
S3_BUCKET_NAME = 'ec2-backup-bucket'

# Replace with your EC2 instance ID
INSTANCE_ID = 'i-xxxxxxxxxxxxxxx'

def lambda_handler(event, context):
    try:
        # Create AMI snapshot
        timestamp = datetime.datetime.now().strftime("%Y-%m-%d-%H-%M-%S")
        ami_name = f"Backup-{INSTANCE_ID}-{timestamp}"
        
        ami_response = ec2.create_image(
            InstanceId=INSTANCE_ID,
            Name=ami_name,
            NoReboot=True
        )
        
        ami_id = ami_response['ImageId']
        
        # Wait for the AMI to be available
        waiter = ec2.get_waiter('image_available')
        waiter.wait(ImageIds=[ami_id])

        # Copy AMI snapshot to S3
        s3_key = f"backups/{ami_name}.json"
        s3.put_object(
            Bucket=S3_BUCKET_NAME,
            Key=s3_key,
            Body=str(ami_response)
        )
        
        # Send SNS notification
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject='EC2 Backup Successful',
            Message=f'Backup completed successfully.\n\nAMI ID: {ami_id}\nStored in S3 bucket: {S3_BUCKET_NAME}/{s3_key}'
        )
        
        return {
            'statusCode': 200,
            'body': f'Backup successful. AMI ID: {ami_id}'
        }
    
    except Exception as e:
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject='EC2 Backup Failed',
            Message=f'Backup failed.\n\nError: {str(e)}'
        )
        return {
            'statusCode': 500,
            'body': str(e)
        }
```

---

### **Step 5: Schedule the Lambda Function with CloudWatch**

1. **Go to the CloudWatch Console:**
   - Navigate to **Events** > **Rules**.
   
2. **Create a New Rule:**
   - **Event Source:** Choose **Event Schedule**.
   - **Schedule Expression:** Set the desired schedule (e.g., `rate(1 day)` for daily backups).
   
3. **Add Target:**
   - Target: Select your Lambda function (`EC2BackupFunction`).
   - Click **Configure Details** and name the rule (e.g., `DailyEC2BackupRule`).

4. **Create the Rule.**

---

### **Step 6: Test the Lambda Function**

1. **Manually Test the Function:**
   - Go to the Lambda function and click **Test**.
   - Create a test event with a basic structure.
   
2. **Verify Backup:**
   - Check the AMI list in the EC2 Console under **AMIs**.
   - Check the S3 bucket for the AMI backup file.
   - Confirm the SNS email notification.

---

## **Additional Considerations**

- **Error Handling:** Enhance the Lambda function with detailed error handling.
- **Cost Management:** Ensure AMIs are cleaned up regularly to avoid excess costs.
- **Logging:** Use CloudWatch Logs for debugging and monitoring Lambda execution.
- **Permissions:** Verify that the Lambda IAM role has necessary permissions for EC2, S3, and SNS.

---
