---
layout: post
title: Encrypting Systems Manager Sessions
tags: [AWS, EC2, VPC, Systems Manager, Session Manager, Encryption, KMS, CloudWatch, S3, CLOUD]
---

I recently discovered that we can encrypt ec2 sessions launched via AWS Systems Manager. I figured it needs a few things in place to make it happen,

* A KMS key to be used for encrypting sessions with the following policy attached to it. 

[Reference](https://docs.aws.amazon.com/AmazonCloudWatch/latest/logs/encrypt-log-data-kms.html)

```{
    "Version": "2012-10-17",
    "Id": "key-default-1",
    "Statement": [
        {
            "Sid": "Enable IAM User Permissions",
            "Effect": "Allow",
            "Principal": {
                "AWS": "arn:aws:iam::1111111111:root"
            },
            "Action": "kms:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Principal": {
                "Service": "logs.ap-southeast-2.amazonaws.com"
            },
            "Action": [
                "kms:Encrypt*",
                "kms:Decrypt*",
                "kms:ReEncrypt*",
                "kms:GenerateDataKey*",
                "kms:Describe*"
            ],
            "Resource": "*"
        }
    ]
}
```

* A S3 bucket to store the logs; also encrypted. Turn on default encryption for the bucket using the same KMS Key.
* A cloudwatch log group.
* Association of the cloudwatch log group with the KMS key defined above. Example command,

```aws logs associate-kms-key --log-group-name stax-session-manager --kms-key-id "arn:aws:kms:ap-southeast-2:1111111111:key/17624518-1d24-4205-9801-a4138937206c"```

* Also, the instances need to have KMS list and get permissions to verify the KMS key presented by Systems Manager.

<img src="/img/sessions-1.png"
     alt="Markdown Monster icon"
     style="float: left; margin-right: 10px;" />

<img src="/img/sessions-2.png"
     alt="Markdown Monster icon"
     style="float: left; margin-right: 20px;" />


`Hoorah! Easily done!`
