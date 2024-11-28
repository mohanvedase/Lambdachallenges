
# Challenge 3: Dynamic Resource Cleanup

## **Objective**  
Automatically detect idle EC2 instances based on low CPU utilization over a specific period and terminate them. Notify the admin via email with details of the terminated instances, including their instance IDs and termination timestamps.

---

## **Step-by-Step Implementation**

### **Step 1: Create an SNS Topic for Notifications**

1. **Go to the SNS Console:**
   - Click **Topics** > **Create Topic**.
   - Name the topic (e.g., `EC2IdleInstanceCleanupNotifications`).

2. **Subscribe an Email Address:**
   - Click **Create Subscription**.
   - Protocol: `Email`.
   - Endpoint: Enter the admin's email address.
   - Confirm the subscription via the email you receive.

3. **Note the SNS Topic ARN**. You will need this for sending notifications.

---

### **Step 2: Create an IAM Role for Lambda**

1. **Go to the IAM Console:**
   - Click **Roles** > **Create Role**.
   - Choose **AWS Service** > **Lambda**.

2. **Attach the Following Policies:**
   - `AmazonEC2ReadOnlyAccess` (to read EC2 instance CPU metrics).
   - `AmazonEC2FullAccess` (to terminate EC2 instances).
   - `AmazonSNSFullAccess` (to publish notifications via SNS).
   - `AWSLambdaBasicExecutionRole` (for Lambda execution permissions).

3. **Name the Role** (e.g., `LambdaEC2IdleCleanupRole`) and click **Create Role**.

---

### **Step 3: Create the Lambda Function**

1. **Go to the Lambda Console:**
   - Click **Create Function**.
   - Choose **Author from Scratch**.
   - Name the function (e.g., `EC2IdleInstanceCleanup`).
   - Runtime: **Python 3.x**.
   - Execution Role: Choose the IAM role created in Step 2.

2. **Write the Lambda Function Code:**
   The function will check for instances with CPU utilization below a specified threshold (e.g., 10%) over the past 30 minutes and terminate them. The function will also send an SNS notification listing the terminated instances.

```python
import boto3
import datetime

ec2 = boto3.client('ec2')
cloudwatch = boto3.client('cloudwatch')
sns = boto3.client('sns')

# Replace with your SNS topic ARN
SNS_TOPIC_ARN = 'arn:aws:sns:your-region:account-id:EC2IdleInstanceCleanupNotifications'

# CPU utilization threshold (percentage)
CPU_THRESHOLD = 10.0

# Time period for checking idle instances (e.g., 30 minutes)
TIME_PERIOD = 30  # minutes

def lambda_handler(event, context):
    try:
        # Get the current time and the time 30 minutes ago
        end_time = datetime.datetime.utcnow()
        start_time = end_time - datetime.timedelta(minutes=TIME_PERIOD)
        
        # Describe EC2 instances
        instances = ec2.describe_instances(
            Filters=[{'Name': 'instance-state-name', 'Values': ['running']}]
        )
        
        terminated_instances = []
        
        for reservation in instances['Reservations']:
            for instance in reservation['Instances']:
                instance_id = instance['InstanceId']
                
                # Get CPU utilization for the instance
                metric_data = cloudwatch.get_metric_statistics(
                    Period=300,  # 5-minute granularity
                    StartTime=start_time,
                    EndTime=end_time,
                    MetricName='CPUUtilization',
                    Namespace='AWS/EC2',
                    Statistics=['Average'],
                    Dimensions=[{'Name': 'InstanceId', 'Value': instance_id}]
                )
                
                if metric_data['Datapoints']:
                    avg_cpu_utilization = metric_data['Datapoints'][0]['Average']
                    
                    # Check if CPU utilization is below the threshold
                    if avg_cpu_utilization < CPU_THRESHOLD:
                        # Terminate the instance
                        ec2.terminate_instances(InstanceIds=[instance_id])
                        terminated_instances.append({
                            'InstanceId': instance_id,
                            'Timestamp': datetime.datetime.utcnow().strftime("%Y-%m-%d %H:%M:%S")
                        })
        
        # Send SNS notification with the list of terminated instances
        if terminated_instances:
            message = "The following EC2 instances were terminated due to low CPU utilization:\n\n"
            for instance in terminated_instances:
                message += f"Instance ID: {instance['InstanceId']} - Terminated at: {instance['Timestamp']}\n"
            
            sns.publish(
                TopicArn=SNS_TOPIC_ARN,
                Subject="EC2 Idle Instance Cleanup Notification",
                Message=message
            )
        
        return {
            'statusCode': 200,
            'body': 'Cleanup process completed successfully.'
        }
    
    except Exception as e:
        sns.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="EC2 Idle Instance Cleanup Failed",
            Message=f"Error: {str(e)}"
        )
        return {
            'statusCode': 500,
            'body': str(e)
        }
```

---

### **Step 4: Test the Lambda Function**

1. **Create a Test Event:**
   - Go to the Lambda console and click **Test**.
   - Create a test event with a basic structure, such as an empty JSON object `{}`.

2. **Verify Execution:**
   - After testing the Lambda function, check the **CloudWatch Logs** for any errors or logs from the execution.
   - Ensure that the instances with low CPU utilization are terminated.
   - Verify that the admin receives an SNS email notification listing the terminated instances.

---

### **Step 5: Schedule the Lambda Function with CloudWatch Events**

1. **Go to the CloudWatch Console:**
   - Navigate to **Events** > **Rules**.

2. **Create a New Rule:**
   - **Event Source:** Choose **Event Schedule**.
   - **Schedule Expression:** Set the desired schedule (e.g., `rate(1 hour)` to run every hour).
   
3. **Add Target:**
   - Target: Select your Lambda function (`EC2IdleInstanceCleanup`).
   - Click **Configure Details** and name the rule (e.g., `HourlyEC2IdleCleanupRule`).

4. **Create the Rule.**

---

### **Step 6: Monitor and Verify the Function**

1. **Monitor Logs:**
   - Check CloudWatch Logs for logs from the Lambda function to ensure it is executing correctly.

2. **Verify Termination:**
   - Go to the EC2 console and check if instances with low CPU utilization were terminated.

3. **Check Notifications:**
   - Ensure that SNS sends the email notifications about terminated instances.

---

## **Additional Considerations**

- **Cost Management:** Regularly review the Lambda and EC2 costs to ensure the solution is cost-effective.
- **Permissions:** Ensure the Lambda IAM role has proper permissions for EC2, SNS, and CloudWatch.
- **Monitoring:** Use CloudWatch for monitoring Lambda execution and performance.
- **Error Handling:** Enhance the function to handle specific errors (e.g., no data points for an instance).

---
