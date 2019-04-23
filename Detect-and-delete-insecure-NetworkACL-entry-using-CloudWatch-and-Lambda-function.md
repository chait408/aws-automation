# Detect and delete insecure NetworkACL entry using CloudWatch and Lambda function

This project describes the process of automating detection of an insecure entry like `0.0.0.0/0` using CloudWatch events and instantly removing the ACL entry using Lambda function
#### Workflow:
- Create IAM role
- Create Lambda function
- Create CloudWatch event pattern rule

#### IAM role for Lambda function
The Lambda function requires a custom role with two policies that will have the capability of deleting the NetworkACL entry and writing logs to CloudWatch log (This additional policy helps in troubleshooting). Create the IAM role with following policies.

1. AWSLambdaBasicExecutionRole Policy
2. Create a custom policy as below that can delete NetworkACL ingress entry

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "VisualEditor0",
            "Effect": "Allow",
            "Action": "ec2:DeleteNetworkAclEntry",
            "Resource": "*"
        }
    ]
}
```
#### Lambda function:
Create a Lambda function with the above mentioned role (which has the ability to delete the NetworkACL entry and write logs to CloudWatch logs)

***Note:** This function is written in Python*
```py
import boto3

def lambda_handler(event, context):
    print("Checking if the created NetworkACL has an insecure ingress ACL entry: 0.0.0.0/0, rule action: allow, port range -1 and Ingress ACL entry")
    if ((event['detail']['eventName'] == "CreateNetworkAclEntry") and (event['detail']['requestParameters']['ruleAction'] == "allow") and (event['detail']['requestParameters']['portRange']['from'] == -1) and (event['detail']['requestParameters']['cidrBlock'] == "0.0.0.0/0") and (event['detail']['requestParameters']['egress']) == False):

        #getting NetworkACL ID from the event.
        NetworkACLId = event['detail']['requestParameters']['networkAclId']

        #getting Rule number to be deleted.
        RuleNumberACLEntry = event['detail']['requestParameters']['ruleNumber']

        #Deleting the insecure Ingress Network ACL entry.
        ec2 = boto3.resource('ec2')
        network_acl = ec2.NetworkAcl(NetworkACLId)
        response = network_acl.delete_entry(Egress=False,RuleNumber=RuleNumberACLEntry)
```
#### CloudWatch Events:
Create a CloudWatch event with following event pattern and add the above Lambda function as the target and an SNS topic which emails when the NetworkACL entry is created

***Note:** Since, we are using AWS API call via CloudTrail to create an event pattern, CloudTrail logging must be enabled*
```json
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
      "CreateNetworkAclEntry",
    ]
  }
}
```
Whenever a NetworkACL entry is created, the CloudWatch event is triggered and it invokes SNS topic and Lambda function. The Lambda function checks if the entry is created with source IP as `0.0.0.0/0` and "All Traffic", and if this is infact true, then it will delete the NetworkACL entry
