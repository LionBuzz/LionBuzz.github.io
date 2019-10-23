---
layout: post
title: EC2-Instance-Scheduler-part-2 
comments: true
tags: [AWS, EC2, CLOUD, LAMBDA, RDS, GITHUB, DYNAMODB]
---

This is the second part of Instance Scheduler where I am going to explain how a lambda function which when invoked by cloudwatch rule, triggers a SNS notification which sends me an email.
Everytime, the specified instance changes state (start,stop,pending), cloudwatch rule triggers the lambda function which then sends sns notification via email to me.

Let's see what the cloudwatch rule looks like,
<img src="/img/cw.png"
     alt="Markdown Monster icon"
     style="float: left; margin-right: 10px;" />

Here is  my lambda function,
```python
import boto3
from multiprocessing import Process
ec2_client = boto3.client('ec2')
ec2_resource = boto3.resource('ec2')
state = ec2_resource.Instance('id')
paginator = ec2_client.get_paginator('describe_instances')

def get_inst():
    pages = paginator.paginate(
         Filters=[
            {
               'Name': 'tag-key',
                'Values': [
                    'ppc-scheduler',
                ]
            },
        ],
    )
    for page in pages:
        return page
        

def lambda_handler(event, context):
    instances = get_inst() # gets the instances with specified tag
        
    instance_ids = []
    
    for reservation in instances['Reservations']:
        for instance in reservation['Instances']:
            instance_ids.append(instance['InstanceId'])
            
    MY_SNS_TOPIC_ARN = 'arn:aws:sns:ap-southeast-2:2222222222:pras-test'
    sns_client = boto3.client('sns')
    for instance_id in instance_ids:
        sns_client.publish(
            TopicArn = MY_SNS_TOPIC_ARN,
            Subject = 'Instance Change State: ' + instance_id,
            Message = 'Instance ' + instance_id + ' has changed state.\n' +
                      'State is: ' + instance['State']['Name']
        )
```
Here is a snippet of my sns notification sent via email,

<img src="/img/sns.png"
     alt="Markdown Monster icon"
     style="float: left; margin-right: 10px;" />
