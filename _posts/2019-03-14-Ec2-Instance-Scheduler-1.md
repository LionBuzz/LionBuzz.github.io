---
layout: post
title: EC2-Instance-Scheduler
tags: [AWS, EC2, CLOUD, LAMBDA, RDS, GITHUB, DYNAMODB]
---

One fine day at work, I was playing around with "Cloud Custodian" which is a tool to schedule instances on-and-off during certain time provided. For example, we would want to have those instances running between 9am to 5pm weekdays. This helps us reduce cost specially if we have 1000's of instances running daily.

Cloud Custodian is a good fit for this task, but I wanted to go AWS native so I tried out the new Instance-Scheduler which is fairly easy to setup as it has a readily available CloudFormation Template for us to use.

This post covers the basic usage of AWS Native Instance Scheduler. One way of reducing costs in our environments is to stop resources that are not in use, and then start those resources again when their capacity is needed. The AWS Instance Scheduler is a solution that automates the starting and stopping of Amazon Elastic Compute Cloud (Amazon EC2) and Amazon Relational Database Service (Amazon RDS) instances.

The Instance Scheduler leverages AWS resource tags and AWS Lambda to automatically stop and restart instances across multiple AWS Regions and accounts on a customer-defined schedule. The solution is easy to deploy and can help reduce operational costs. For example, an organization can use the Instance Scheduler in a production environment to automatically stop instances every day, outside of business hours. For customers who leave all of their instances running at full utilization, this solution can result in up to 70% cost savings for those instances that are only necessary during regular business hours (weekly utilization reduced from 168 hours to 50 hours).

<img src="/img/instance.png"
     alt="Markdown Monster icon"
     style="float: left; margin-right: 10px;" />


Here is the link to AWS Documentation on [EC2 Instance Scheduler](https://aws.amazon.com/solutions/instance-scheduler/)

And, [Github Repo](https://github.com/awslabs/aws-instance-scheduler)

The AWS CloudFormation template sets up an Amazon CloudWatch event at a customer-defined interval. This event invokes the Instance Scheduler AWS Lambda function. During
configuration, the user defines the AWS Regions and accounts, as well as a custom tag that
the Instance Scheduler will use to associate schedules with applicable Amazon EC2 instances.These values are stored in Amazon DynamoDB, and the Lambda
function retrieves them each time it runs. The customer then applies the custom tag to
applicable instances.

During initial configuration of the Instance Scheduler, you define a tag key you will use to
identify applicable Amazon EC2 instances. When you create a schedule,
the name you specify is used as the tag value that identifies the schedule you want to apply
to the tagged resource. For example, a user might use the solution’s default tag name (tag
key) Schedule and create a schedule called SydneyOfficeHours. To identify an instance
that will use the SydneyOfficeHours schedule, the user adds the Schedule tag key with a
value of SydneyOfficeHours.

<img src="/img/dynamo.png"
     alt="Markdown Monster icon"
     style="float: left; margin-right: 10px;" />

Each time the solution’s Lambda function runs, it checks the current state of each
appropriately tagged instance against the targeted state (defined by one or more periods in a
schedule in the instance tag) in the associated schedule, and then applies the appropriate
start or stop action, as necessary.

The Lambda function also records the name of the schedule, the number of instances
associated with that schedule, and the number of running instances as an optional custom
metric in Amazon CloudWatch.

To create a period, to define the time. I used scheduler-cli and the command,
~~~
scheduler-cli create-period --name sydneyofficehours --description "Office Time for Sydney Office" --begintime 08:00 --endtime 18:00 --weekdays mon-fri --stack pcst-instance-scheduler
~~~

Then I associated the period with a schedule,
~~~
scheduler-cli create-schedule --name SydneyOfficeHours --periods sydneyofficehours --description "Schedule the Instances to be monitored and brought up/shut down" --timezone Australia/Sydney --stack pcst-instance-scheduler
~~~
After setting the schedule, the instances that have the tags defined in the cloudformation stack for instance-scheduler will be started every weekday at 8 am and shut down at 6 pm.

This took me 15 minutes from deployment to testing and I find it to be really useful in terms of cost savings and to manage the instances properly. The great thing about this feature is that we can have instances in another account or region and we can still monitor and change thier state from one parent account. All we need is to specify the account-id's while deploying the cloudformation template.

In the second post, I will talk about creating a cloudwatch rule to check the state of these instances and everytime they change state, a lambda function will be triggered to send a SNS notification to me. 