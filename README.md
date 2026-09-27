# Assignment 3 – Monitor EC2 State Changes

## Objective

Create an AWS Lambda function using Python and Boto3 to monitor EC2 instance state changes and send notifications through Amazon SNS.

The solution will:

- Monitor EC2 instance state changes.
- Use Amazon EventBridge to detect EC2 state-change events.
- Trigger an AWS Lambda function when an EC2 instance changes state.
- Retrieve the EC2 instance ID and new state.
- Send the event details through Amazon SNS.
- Record Lambda execution details in CloudWatch Logs.

---

## AWS Services Used

- Amazon EC2
- AWS Lambda
- Amazon EventBridge
- Amazon SNS
- AWS IAM
- Amazon CloudWatch Logs

---

# Architecture

```text
                  Amazon EC2
                      |
                      | State Change
                      v
              Amazon EventBridge
                      |
                      | Event
                      v
                 AWS Lambda
                      |
              +-------+-------+
              |               |
        Process Event     CloudWatch Logs
              |
              v
             SNS
              |
              v
       Email Notification
```

---

# Prerequisites

Before starting this assignment, make sure you have:

- An AWS account.
- Permission to create Lambda functions.
- Permission to create IAM roles and policies.
- Permission to create EventBridge rules.
- Permission to create SNS topics and subscriptions.
- At least one EC2 instance for testing.
- Python 3.x supported by AWS Lambda.
- An email address that can receive SNS notifications.

---

# Project Structure

The project uses the following structure:

```text
assignment-03-ec2-state-monitor/
│
├── lambda_function.py
├── README.md
│
└── screenshots/
    ├── 01-sns-topic.png
    ├── 02-sns-subscription.png
    ├── 03-iam-policy.png
    ├── 04-lambda-configuration.png
    ├── 05-lambda-environment-variable.png
    ├── 06-eventbridge-rule.png
    ├── 07-eventbridge-target.png
    ├── 08-ec2-state-change.png
    ├── 09-lambda-execution.png
    ├── 10-cloudwatch-logs.png
    └── 11-sns-email.png
```

---

# Step 1 – Create SNS Topic

Amazon SNS will be used to send notifications when an EC2 instance changes state.

Go to:

```text
AWS Console
    ↓
Amazon SNS
    ↓
Topics
    ↓
Create topic
```

Select:

```text
Type: Standard
```

Use the following topic name:

```text
ec2-state-change-alerts
```

Leave the remaining settings at their default values.

Click:

```text
Create topic
```

After the topic is created, open the topic and copy the **Topic ARN**.

Example:

```text
arn:aws:sns:ap-south-1:123456789012:ec2-state-change-alerts
```

> Replace the example ARN with the actual ARN from your AWS account.

### Screenshot

**Screenshot required:** Yes

Capture:

- SNS topic name
- SNS topic ARN
<img width="1139" height="786" alt="image" src="https://github.com/user-attachments/assets/2cdbacf1-6886-4b7e-82a8-0e420137d950" />

---

# Step 2 – Create SNS Email Subscription

The SNS topic needs an email subscription to receive the EC2 state-change notifications.

Open:

```text
Amazon SNS
    ↓
Topics
    ↓
ec2-state-change-alerts
    ↓
Create subscription
```

Configure:

| Setting | Value |
|---|---|
| Protocol | `Email` |
| Endpoint | Your email address |

Click:

```text
Create subscription
```

AWS will send a confirmation email.

Open the email and click:

```text
Confirm subscription
```

Return to:

```text
SNS
    ↓
Topics
    ↓
ec2-state-change-alerts
    ↓
Subscriptions
```

The subscription should show:

```text
Confirmed
```

### Screenshot


SNS subscription showing the confirmed status.
<img width="1146" height="788" alt="image" src="https://github.com/user-attachments/assets/97ba4e5a-c6a9-4a73-8b05-52aebad7d550" />


---

# Step 3 – Create IAM Role

Create an IAM execution role that will be used by the Lambda function.

Go to:

```text
AWS Console
    ↓
IAM
    ↓
Roles
    ↓
Create role
```

Select:

```text
Trusted entity type:
AWS service
```

Select:

```text
Use case:
Lambda
```

Click:

```text
Next
```

Attach the AWS managed policy:

```text
AWSLambdaBasicExecutionRole
```

This allows Lambda to write logs to CloudWatch Logs.

Create the role with:

```text
Role name:
ec2-state-monitor-lambda-role
```

Click:

```text
Create role
```

---

# Step 4 – Add SNS Permission to IAM Role

Open:

```text
IAM
    ↓
Roles
    ↓
ec2-state-monitor-lambda-role
```

Select:

```text
Add permissions
    ↓
Create inline policy
```

Select:

```text
JSON
```

Use the following policy.

Replace:

```text
YOUR_SNS_TOPIC_ARN
```

with the actual SNS Topic ARN from Step 1.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "PublishToEC2StateChangeSNS",
      "Effect": "Allow",
      "Action": [
        "sns:Publish"
      ],
      "Resource": "arn:aws:sns:ap-south-1:075237969193:ec2-state-change-alerts"
    }
  ]
}
```

Set the policy name to:

```text
EC2StateChangeSNSPublishPolicy
```

Create the policy.

### Permission Description

| Permission | Purpose |
|---|---|
| `sns:Publish` | Allows Lambda to publish EC2 state-change notifications |
| `AWSLambdaBasicExecutionRole` | Allows Lambda to write CloudWatch Logs |
<img width="1140" height="779" alt="image" src="https://github.com/user-attachments/assets/df925f02-1bbf-4aae-9268-5d952ec33a02" />
<img width="1919" height="797" alt="image" src="https://github.com/user-attachments/assets/d1f8d17f-2309-4318-a46a-70fa27195c9d" />


---

# Step 5 – Create Lambda Function

Go to:

```text
AWS Console
    ↓
Lambda
    ↓
Functions
    ↓
Create function
```

Select:

```text
Author from scratch
```

Configure:

| Configuration | Value |
|---|---|
| Function name | `ec2-state-change-monitor` |
| Runtime | Python 3.x |
| Architecture | x86_64 |

Under:

```text
Change default execution role
```

select:

```text
Use an existing role
```

Select:

```text
ec2-state-monitor-lambda-role
```

Click:

```text
Create function
```
<img width="1132" height="777" alt="image" src="https://github.com/user-attachments/assets/d8e80b1a-70d5-4313-85f7-971b5067ed7e" />


---

# Step 6 – Add Lambda Function Code

Open:

```text
Lambda
    ↓
ec2-state-change-monitor
    ↓
Code
```

Replace the default Lambda code with:

```python
import boto3
import os


sns = boto3.client("sns")

SNS_TOPIC_ARN = os.environ["SNS_TOPIC_ARN"]


def lambda_handler(event, context):

    print("===== EC2 STATE CHANGE EVENT =====")
    print(event)

    detail = event.get("detail", {})

    instance_id = detail.get("instance-id")
    state = detail.get("state")

    message = (
        "EC2 State Change Notification\n\n"
        f"Instance ID: {instance_id}\n"
        f"New State: {state}\n"
    )

    print(message)

    response = sns.publish(
        TopicArn=SNS_TOPIC_ARN,
        Subject="EC2 State Change Alert",
        Message=message
    )

    print(
        f"SNS Message ID: {response['MessageId']}"
    )

    return {
        "statusCode": 200,
        "instance_id": instance_id,
        "state": state,
        "sns_message_id": response["MessageId"]
    }
```

Click:

```text
Deploy
```

---

# Step 7 – Configure Lambda Environment Variable

The SNS Topic ARN is stored as an environment variable instead of hard-coding it into the Python code.

Go to:

```text
Lambda
    ↓
ec2-state-change-monitor
    ↓
Configuration
    ↓
Environment variables
```

Click:

```text
Edit
```

Add:

| Key | Value |
|---|---|
| `SNS_TOPIC_ARN` | Your SNS Topic ARN |

Example:

```text
Key:
SNS_TOPIC_ARN

Value:
arn:aws:sns:ap-south-1:123456789012:ec2-state-change-alerts
```

Save the environment variable.

### Screenshot

<img width="1126" height="782" alt="image" src="https://github.com/user-attachments/assets/026cc6b2-63cb-40b5-9555-bec17364cf28" />

```

> Avoid exposing sensitive information in screenshots. An SNS topic ARN is generally not a secret, but avoid including unrelated account information where possible.

---

# Step 8 – Create EventBridge Rule

Amazon EventBridge will detect EC2 instance state-change events and invoke the Lambda function.

Go to:

```text
AWS Console
    ↓
Amazon EventBridge
    ↓
Rules
    ↓
Create rule
```

Set:

```text
Name:
ec2-state-change-monitor-rule
```

Choose the appropriate event bus, normally:

```text
default
```

For the rule type, select:

```text
Rule with an event pattern
```

---

# Step 9 – Configure Event Pattern

Configure the event pattern for EC2 state changes.

Select:

```text
Event source:
AWS services
```

Select:

```text
AWS service:
EC2
```

Select:

```text
Event type:
EC2 Instance State-change Notification
```

The resulting event pattern should be similar to:

```json
{
  "source": [
    "aws.ec2"
  ],
  "detail-type": [
    "EC2 Instance State-change Notification"
  ],
  "detail": {
    "state": [
      "pending",
      "running",
      "stopping",
      "stopped",
      "shutting-down",
      "terminated"
    ]
  }
}
```

This allows the rule to respond to EC2 state-change events.

---

# Step 10 – Configure EventBridge Target

Under:

```text
Target
```

select:

```text
AWS service
```

Select:

```text
Lambda function
```

Choose:

```text
ec2-state-change-monitor
```

Complete the rule creation.

EventBridge should now invoke the Lambda function whenever a matching EC2 state-change event occurs.

### Screenshot

**Screenshot required:** Yes

Capture the EventBridge rule showing:

- Rule name
- Event pattern
- Event source
- EC2 state-change event type
<img width="1146" height="794" alt="image" src="https://github.com/user-attachments/assets/00cece1e-78c2-4490-8cc0-b6d0dae638a6" />

---

# Step 11 – Verify EventBridge Target

Open the created rule:

```text
EventBridge
    ↓
Rules
    ↓
ec2-state-change-monitor-rule
```

Check the target configuration.

The target should be:

```text
ec2-state-change-monitor
```

### Screenshot

**Screenshot required:** Yes

Capture the target configuration.

Save as:

```text
screenshots/07-eventbridge-target.png
```

---

# Step 12 – Test EC2 State Change

Use an existing EC2 instance that is safe to stop and start.

Go to:

```text
AWS Console
    ↓
EC2
    ↓
Instances
```

Select the test instance.

Record the instance ID.

Example:

```text
i-0123456789abcdef0
```

If the instance is running, select:

```text
Instance state
    ↓
Stop instance
```

The EC2 instance should transition through states such as:

```text
stopping
    ↓
stopped
```

These state changes generate EventBridge events.

### Screenshot

**Screenshot required:** Yes

Capture the EC2 instance showing the state change.

Save as:

```text
screenshots/08-ec2-state-change.png
```

---

# Step 13 – Verify Lambda Execution

After the EC2 state change, go to:

```text
AWS Lambda
    ↓
ec2-state-change-monitor
    ↓
Monitor
    ↓
View CloudWatch logs
```

Open the latest log stream.

The Lambda log should contain information similar to:

```text
===== EC2 STATE CHANGE EVENT =====

EC2 State Change Notification

Instance ID: i-0123456789abcdef0
New State: stopped

SNS Message ID: xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx
```

The actual instance ID, state and SNS message ID will depend on the test.

### Screenshot

**Screenshot required:** Yes

Capture the successful Lambda execution.

Save as:

```text
screenshots/09-lambda-execution.png
```

---

# Step 14 – Verify CloudWatch Logs

AWS Lambda automatically sends its execution output to CloudWatch Logs.

The log group will normally be:

```text
/aws/lambda/ec2-state-change-monitor
```

The logs should show:

```text
===== EC2 STATE CHANGE EVENT =====
```

followed by the EC2 event information.

The important information is:

```text
Instance ID
New State
SNS Message ID
```
 CloudWatch log containing the EC2 instance ID and new state.
<img width="1656" height="501" alt="image" src="https://github.com/user-attachments/assets/f54d894f-4f69-49df-95b6-d362c4db5e62" />

```

---

# Step 15 – Verify SNS Email Notification

Check the email address subscribed to the SNS topic.

You should receive an email with a subject similar to:

```text
EC2 State Change Alert
```

The message should contain:

```text
EC2 State Change Notification

Instance ID: i-0123456789abcdef0
New State: stopped
```

The actual instance ID and state depend on the EC2 instance used for testing.

 SNS notification email.


<img width="1787" height="569" alt="image" src="https://github.com/user-attachments/assets/bc7d4338-35b4-4561-bd19-3f5c8abd35b3" />

```

---

# Testing

## Test Scenario

The complete workflow is tested using an EC2 instance.

### Test Procedure

1. Select a test EC2 instance.
2. Change the instance state.
3. EC2 generates a state-change event.
4. EventBridge receives the event.
5. EventBridge invokes the Lambda function.
6. Lambda extracts the instance ID and new state.
7. Lambda publishes the notification to SNS.
8. SNS sends the notification to the subscribed email address.
9. Lambda execution details are written to CloudWatch Logs.

---

## Expected Workflow

```text
EC2 State Change
       |
       v
EventBridge Rule
       |
       v
Lambda Function
       |
       +--------------------+
       |                    |
       v                    v
CloudWatch Logs            SNS
                            |
                            v
                         Email
```

---

## Expected Result

Lambda execution should complete successfully.

Example:

```text
Lambda execution: SUCCESS
```

The SNS notification should contain:

```text
Instance ID: <EC2_INSTANCE_ID>
New State: <EC2_STATE>
```

---

# Troubleshooting

## 1. Lambda Does Not Receive the EC2 Event

Check:

- EventBridge rule is enabled.
- Event pattern matches EC2 state-change events.
- Lambda is configured as the EventBridge target.
- EventBridge has permission to invoke Lambda.
- The EC2 instance actually changed state.

---

## 2. SNS Notification Is Not Received

Check:

- SNS subscription is confirmed.
- Lambda execution role has `sns:Publish`.
- `SNS_TOPIC_ARN` contains the correct SNS topic ARN.
- Lambda execution logs for errors.
- The email inbox and spam folder.

---

## 3. Lambda Returns AccessDenied

Verify that the Lambda execution role contains:

```text
sns:Publish
```

for the correct SNS topic ARN.

Also verify that:

```text
SNS_TOPIC_ARN
```

matches the actual SNS topic ARN.

---

## 4. Lambda Environment Variable Error

If Lambda returns an error similar to:

```text
KeyError: 'SNS_TOPIC_ARN'
```

verify that the environment variable exists under:

```text
Lambda
    ↓
Configuration
    ↓
Environment variables
```

---

# Screenshot Checklist

The following screenshots provide evidence of the implementation:

| # | Screenshot | Required |
|---|---|---|
| 1 | SNS Topic | Yes |
| 2 | SNS Subscription | Yes |
| 3 | IAM Role and Policy | Yes |
| 4 | Lambda Configuration | Yes |
| 5 | Lambda Environment Variable | Yes |
| 6 | EventBridge Rule | Yes |
| 7 | EventBridge Target | Yes |
| 8 | EC2 State Change | Yes |
| 9 | Lambda Execution | Yes |
| 10 | CloudWatch Logs | Yes |
| 11 | SNS Email Notification | Yes |

---

# Project Structure

```text
#assignment-03-ec2-state-monitor/
│
├── lambda_function.py
├── README.md
```

---

# Conclusion
This assignment demonstrates an event-driven AWS monitoring workflow using:

- Amazon EC2
- Amazon EventBridge
- AWS Lambda
- Amazon SNS
- AWS IAM
- Amazon CloudWatch Logs

When an EC2 instance changes state, EventBridge detects the event and invokes the Lambda function.

The Lambda function extracts the EC2 instance ID and new state, publishes the information to SNS, and records the execution details in CloudWatch Logs.

The SNS subscription then delivers the EC2 state-change notification to the configured email address.
