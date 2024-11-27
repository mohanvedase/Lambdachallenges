### Auto Scaling Notification with AWS Lambda and SNS  
This document outlines the setup and code for an automatic EC2 scaling notification system using AWS Lambda, CloudWatch Alarms, Auto Scaling Groups, and SNS (Simple Notification Service).

---

### **Objective**  
To monitor EC2 Auto Scaling events (launch or termination) and send email notifications using SNS whenever an instance is scaled.

---

### **Table of Contents**  
1. [Setup Procedure](#setup-procedure)  
   - [Create Auto Scaling Group (ASG)](#1-create-auto-scaling-group-asg)  
   - [Create CloudWatch Alarm](#2-create-cloudwatch-alarm)  
   - [Create SNS Topic](#3-create-sns-topic)  
   - [Create Lambda Function](#4-create-lambda-function)  
   - [Attach Lambda to EventBridge](#5-attach-lambda-to-eventbridge)  
2. [Lambda Python Code](#lambda-python-code)  
3. [IAM Permissions for Lambda Role](#iam-permissions-for-lambda-role)  
4. [Testing the Solution](#testing-the-solution)  

---

## **Setup Procedure**  

### 1. **Create Auto Scaling Group (ASG)**  
1. Go to **EC2 Dashboard** > **Auto Scaling Groups**.  
2. Click **Create Auto Scaling Group** and follow these steps:  
   - Select an existing Launch Template or Launch Configuration.  
   - Specify desired instance count, minimum, and maximum instances.  
   - Set scaling policies (e.g., scale out at CPU > 80%, scale in at CPU < 20%).  
3. Complete the setup and create the group.

---

### 2. **Create CloudWatch Alarm**  
1. Go to **CloudWatch Dashboard** > **Alarms**.  
2. Click **Create Alarm**:  
   - **Metric**: Choose **EC2 > Auto Scaling Group Metrics > CPUUtilization**.  
   - Set threshold: CPU utilization > 80% for scaling out, < 20% for scaling in.  
   - **Action**: Trigger the Lambda function when the alarm is breached.  

---

### 3. **Create SNS Topic**  
1. Go to **SNS Dashboard** > **Topics**.  
2. Click **Create Topic**:  
   - **Name**: `EC2ScalingNotifications`.  
   - Copy the **ARN** of the topic for later use.  
3. Create a subscription for email:  
   - **Protocol**: Email.  
   - **Endpoint**: Your email address.  
   - Confirm the subscription via the email received.

---

### 4. **Create Lambda Function**  
1. Go to **Lambda Dashboard** > **Create Function**:  
   - **Runtime**: Python 3.x.  
   - **Execution Role**: Use a role with EC2, AutoScaling, CloudWatch, and SNS permissions.  

2. Add the following **Python code**:  

```python
import boto3
import json

# Initialize AWS clients
sns_client = boto3.client('sns')

# Replace with your SNS Topic ARN
SNS_TOPIC_ARN = 'arn:aws:sns:your-region:123456789012:EC2ScalingNotifications'

def lambda_handler(event, context):
    try:
        # Log the event for debugging
        print(f"Received event: {json.dumps(event)}")
        
        # Parse the event
        message = json.loads(event['Records'][0]['Sns']['Message'])
        event_type = message['Event']
        asg_name = message['AutoScalingGroupName']
        instance_id = message['EC2InstanceId']
        time_occurred = message['Time']
        
        # Create notification message
        notification_message = (
            f"Auto Scaling Event:\n"
            f"Event Type: {event_type}\n"
            f"Auto Scaling Group: {asg_name}\n"
            f"Instance ID: {instance_id}\n"
            f"Time: {time_occurred}\n"
        )
        
        # Send notification via SNS
        sns_client.publish(
            TopicArn=SNS_TOPIC_ARN,
            Subject="Auto Scaling Event Notification",
            Message=notification_message
        )
        
        return {
            'statusCode': 200,
            'body': json.dumps('Notification sent successfully.')
        }
        
    except Exception as e:
        print(f"Error: {str(e)}")
        return {
            'statusCode': 500,
            'body': json.dumps(f"Error: {str(e)}")
        }
```

---

### 5. **Attach Lambda to EventBridge**  
1. Go to **EventBridge (CloudWatch Events)** > **Rules**.  
2. Create a new rule:  
   - **Event Source**: AWS services > **Auto Scaling**.  
   - **Event Types**:  
     - `EC2 Instance Launch Successful`.  
     - `EC2 Instance Terminate Successful`.  
3. Select the Lambda function you created as the **Target** and save the rule.

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
        "ec2:DescribeInstances",
        "autoscaling:DescribeAutoScalingGroups",
        "sns:Publish"
      ],
      "Resource": "*"
    }
  ]
}
```

---

## **Testing the Solution**  
1. **Simulate Scaling**:
   - Use a stress test tool or manually adjust scaling policies to trigger an Auto Scaling event.
   
2. **Check CloudWatch and SNS**:  
   - Verify the scaling event appears in CloudWatch.  
   - Check your email or SNS endpoint for notifications.

---

### **Conclusion**  
With this setup, your Lambda function will monitor EC2 Auto Scaling events and send detailed notifications via SNS whenever instances are launched or terminated.

---  

