
# Challenge 4: Data Pipeline Trigger

## **Objective**  
Automate a data processing pipeline using AWS services:  
- Trigger a Lambda function when a file is uploaded to an S3 bucket.  
- Launch an EC2 instance via Lambda, passing S3 file information.  
- EC2 instance processes data and uploads results to an output S3 bucket.  
- SNS sends an email notification when results are available.

---

## **Step-by-Step Implementation**

### **Step 1: Prepare an AMI for EC2**

1. **Launch an EC2 Instance:**
   - Instance Type: `t2.micro`.
   - Security Group: Default or custom security group.

2. **Connect to the Instance:**
   ```bash
   sudo apt update
   sudo apt install unzip -y
   curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
   unzip awscliv2.zip
   sudo ./aws/install
   aws --version
   ```

3. **Configure AWS CLI:**
   ```bash
   aws configure
   ```

4. **Verify S3 Connectivity:**
   ```bash
   aws s3 ls
   ```

5. **Create an AMI:**
   - In the AWS EC2 Console, select the instance.
   - Click **Actions > Create Image** and note the AMI ID.

6. **Terminate the EC2 Instance** (optional to save costs).

---

### **Step 2: Create an SNS Topic**

1. **Go to AWS SNS Console:**
   - Create a new SNS topic named `DataPipelineNotifications`.

2. **Subscribe Your Email:**
   - Protocol: `Email`.
   - Endpoint: Your email address.
   - Confirm the subscription via email.

3. **Note the SNS Topic ARN.**

---

### **Step 3: Create an IAM Role**

1. **Create a New IAM Role:**
   - Attach the following policies:
     - `AmazonEC2FullAccess`
     - `AmazonS3FullAccess`
     - `AmazonSNSFullAccess`

2. **Attach the Role to the Lambda Function.**

---

### **Step 4: Set Up S3 Buckets**

1. **Create Two S3 Buckets:**
   - **Source Bucket:** For input files (e.g., `source-bucket-name`).
   - **Output Bucket:** For processed files (e.g., `file-upload-bucket-name`).

2. **Note the Bucket Names.**

---

### **Step 5: Create the Lambda Function**

1. **Go to AWS Lambda Console:**
   - Create a new function named `DataPipelineTriggerLambda`.
   - Runtime: `Python 3.x`.
   - Execution Role: Use the IAM Role created in Step 3.

2. **Add the Lambda Function Code:**
   ```python
   import boto3

   def lambda_handler(event, context):
       ec2 = boto3.client('ec2', region_name='us-east-1')  # Replace with your region
       user_data_script = """#!/bin/bash
       apt-get update -y
       su - ubuntu -c "aws s3 cp s3://source_bucket_name/input_file.csv /home/ubuntu/input_file/"
       su - ubuntu -c "aws s3 cp /home/ubuntu/input_file/input_file.csv s3://file_upload_bucket_name/output/"
       """
       try:
           response = ec2.run_instances(
               ImageId='ami-0dca3bf41ba82587f',  # Replace with your AMI ID
               InstanceType='t2.micro',
               MinCount=1,
               MaxCount=1,
               KeyName='your-key-name',  # Replace with your Key Pair name
               SecurityGroupIds=['sg-your-security-group-id'],  # Replace with your Security Group ID
               SubnetId='subnet-your-subnet-id',  # Replace with your Subnet ID
               UserData=user_data_script,
               TagSpecifications=[
                   {'ResourceType': 'instance', 'Tags': [{'Key': 'Name', 'Value': 'MyInstance'}]}
               ]
           )
           instance_id = response['Instances'][0]['InstanceId']
           return {'statusCode': 200, 'body': f'Instance {instance_id} created successfully'}
       except Exception as e:
           return {'statusCode': 500, 'body': str(e)}
   ```

---

### **Step 6: Configure S3 Trigger for Lambda**

1. **Go to Source Bucket in S3:**
   - Navigate to **Properties > Event Notifications**.

2. **Create a New Event Notification:**
   - Event Type: `PUT`.
   - Destination: Select the Lambda function created in Step 5.

---

### **Step 7: Prepare the Input File**

1. **Create a Test File:**
   - Example: `input_file.csv`.

2. **Upload the File to the Source Bucket.**

---

### **Step 8: Update Lambda Code Placeholders**

Replace the placeholders in the Lambda function with actual values:
- `AMI ID`: From Step 1.
- `KeyName`: EC2 Key Pair name.
- `SubnetId`: From EC2 instance details.
- `SecurityGroupIds`: EC2 Security Group ID.
- `source_bucket_name`: Source Bucket name.
- `file_upload_bucket_name`: Output Bucket name.

---

### **Step 9: Add SNS Notification**

1. **Publish to SNS Topic in Lambda Code (Optional):**
   ```python
   sns = boto3.client('sns')
   sns.publish(
       TopicArn='arn:aws:sns:your-region:123456789012:DataPipelineNotifications',
       Message=f"Data processing complete for instance {instance_id}.",
       Subject="Data Pipeline Notification"
   )
   ```

---

### **Step 10: Deploy and Test**

1. **Deploy the Lambda Function.**
2. **Upload a File to the Source Bucket.**
3. **Verify:**
   - EC2 instance launches and processes the file.
   - Processed file is uploaded to the output bucket (`/output` folder).
   - SNS sends an email notification.

---

## **Additional Suggestions**

- **Logging:** Add CloudWatch logs for debugging.
- **Error Handling:** Handle cases for missing files or permissions.
- **Cleanup:** Automatically terminate EC2 instances after processing to save costs.
- **Permissions:** Ensure Lambda has permissions for S3, EC2, and SNS.

---
