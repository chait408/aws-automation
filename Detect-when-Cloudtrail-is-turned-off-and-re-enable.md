# Detect when Cloudtrail is turned off and re-enable.

This project describes the process of automating when a Cloudtrail is turned off and re-enabling it.
#### Workflow:
- Create IAM role
- Create Lambda function
- Create CloudWatch event pattern rule

#### IAM role for Lambda function
The Lambda function requires a custom role with two policies that will have the capability of re-enabling cloudtrail logging and writing logs to CloudWatch log (This additional policy helps in troubleshooting). Create the IAM role with following policies.

1. AWSLambdaBasicExecutionRole Policy
2. Create a custom policy as below that can re-enable cloudtrail logging

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "cloudtrail:StartLogging",
            "Resource": "<Insert_Cloudtrail_ARN>"
        }
    ]
}
```

#### Lambda function:
Create a Lambda function with the above mentioned role (which has the ability to reenable Cloudtrail logging and write logs to CloudWatch logs)

***Note:** This function is written in Python*
```py
import json
import boto3

def lambda_handler(event, context):
    # Check if the stopped cloudtrail is "<Insert_Cloudtrail_Name>" trail.
    print("Checking if the stopped cloudtrail is the <Insert_Cloudtrail_Name> trail")
    if ((event['detail']['eventName'] == "StopLogging") and (event['detail']['requestParameters']['name'] == "<Insert_Cloudtrail_ARN>")):
        client = boto3.client('cloudtrail')
        #Re-Start logging on <Insert_Cloudtrail_Name> trail
        response = client.start_logging(Name='<Insert_Cloudtrail_ARN>')
```

#### CloudWatch Events:
Create a CloudWatch event with following event pattern and add the above Lambda function as the target and an SNS topic which emails when the Cloudtrail logging is turned off

***Note:** Since, we are using AWS API call via CloudTrail to create an event pattern, CloudTrail logging must be enabled*
```json
{
  "source": [
    "aws.cloudtrail"
  ],
  "detail-type": [
    "AWS API Call via CloudTrail"
  ],
  "detail": {
    "eventSource": [
      "cloudtrail.amazonaws.com"
    ],
    "eventName": [
      "StopLogging"
    ]
  }
}
```
Whenever Cloudtrail is turned off, the CloudWatch event is triggered and it invokes SNS topic and Lambda function. The Lambda function checks if the Cloudtrail logging is turned off and re-enables it.
