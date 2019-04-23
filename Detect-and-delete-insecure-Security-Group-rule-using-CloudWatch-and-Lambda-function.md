# Detect and delete insecure Security Group rule using CloudWatch and Lambda function.

This project describes the process of automating detection of an insecure rule like 0.0.0.0/0 using CloudWatch events and instantly removing the rule using Lambda function.
#### Workflow:
- Create IAM role.
- Create Lambda function.
- Create CloudWatch event pattern rule.

#### IAM role for Lambda function:
The Lambda function requires a custom role with two policies that will have the capability of revoking the SG rule and writing logs to CloudWatch log (This additional policy helps in troubleshooting). Create the IAM role with following policies.

1. AWSLambdaBasicExecutionRole Policy.
2. Create a custom policy as below that can revoke SG Ingress rule.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "ec2:RevokeSecurityGroupIngress",
            "Resource": "*"
        }
    ]
}
```
#### Lambda function:
Create a Lambda function with the above mentioned role (which has the ability to revoke the SG ingress rule and write logs to CloudWatch logs).

***Note:** This function is written in Python.*
```
import boto3

def lambda_handler(event, context):
    # Check if Authorized Ingress rule has All protocol, All Ports and Source of 0.0.0.0/0:
    print("Checking if the Authorized Ingress rule has All protocol, All Ports and Source of 0.0.0.0/0")
    
    if ((event['detail']['requestParameters']['ipPermissions']['items'][0]['ipRanges']['items'][0]['cidrIp'] == "0.0.0.0/0") and (event['detail']['requestParameters']['ipPermissions']['items'][0]['ipProtocol'] == "-1")):
     
     #Grabs Security Group ID from the event
     sgGroupId = event['detail']['requestParameters']['groupId']
     
     #Revokes the insecure rule
     client = boto3.client('ec2')
     client.revoke_security_group_ingress(GroupId=sgGroupId, IpPermissions=[{'IpProtocol': '-1', 'IpRanges':[{'CidrIp': '0.0.0.0/0'}]}])

```
#### CloudWatch Events:
Create a CloudWatch event with the following Event pattern and add the above Lambda function as the target and an SNS topic which emails when a SG Ingress rule is created.

***Note:** Since, we are using AWS API call via CloudTrail to create an event pattern, CloudTrail logging must be enabled.*

```
{
  "source": [
    "aws.ec2"
  ],
  "detail-type": [
    "AWS API Call via CloudTrail"
  ],
  "detail": {
    "eventSource": [
      "ec2.amazonaws.com"
    ],
    "eventName": [
      "AuthorizeSecurityGroupIngress"
    ]
  }
}
```
Whenever a SG rule is created, the CloudWatch event is triggered and it invokes SNS topic and  Lambda function. The Lambda function checks if the rule is created with source IP as "0.0.0.0/0" and "All Traffic", and if this is infact true, then it will revoke the SG rule. 
