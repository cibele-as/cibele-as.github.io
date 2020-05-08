---
title: Creating budget alarms in AWS using Terraform and integrate with Slack or Email
tags: [AWS, Terraform]
style: fill
color: success
---

Who never had a surprise when checking the billing panel from Amazon, google, or another cloud provider? It happened 
with me, a couple of times when checking my AWS personal account and I am sure it happens with you as well as in your 
organization 😅

## Common problems and motivations 💸

It’s not rare that we sometimes think a service would cost X and it ends up costing 2X in the final of the month, 
sometimes we also create resources manually for experimentations and forget to clean them up, on the other hand, 
it is not healthy to access the billings every day to check whether the costs are okay or not, so the best way 
to avoid surprises at the end of the month would be automating this process and creating some sort of alarms when 
something is wrong.

Is this post we are going to create a process to keep track of our billings in AWS by defining a threshold in the account
level as well as by services level and also creating an integration with Slack since I have seen that people prefer slack 
rather than email, for that, we are going to use [Terraform](https://www.terraform.io/) and the [Email Slack App](https://slack.com/apps/A0F81496D-email) integration to send alarms from our account to a specific channel in Slack.

## Implementation Scenario

Using AWS Billings its possible to keep track of costs using a lot of approaches, for instance, resource tags where we can filter costs by a specific business or teams, resource names using amazon services names, etc. It is possible to set alarms
when for forecast values or current values.

In our study case, we have a `development account` in AWS and we use two services `EC2` and `S3`, looking into the previous 
billing we usually pay for those:

- **EC2:** $ 10,00 
- **S3:** $ 5,00

We also want to reserve $ 5,00 in the account for eventual costs, in the end, what we need to achieve is:

Receive an alert when:
- Forecast amount for the `Development account` is `greater than 20,00`
- Forecast amount for `EC2` is `greater than 10,00`
- Forecast amount for `S3` is `greater than 5,00`

After implementing the module in terraform, we are going to use it like this:

``` hcl
module "billing_alarm" {
  source = "./../../modules/budgets"

  account_name         = "Development"
  account_budget_limit = "20.0"

  services = {
    EC2 = {
      budget_limit = "10.0"
    },
    S3 = {
      budget_limit = "5.0"
    }
  }
}
```

## Terraform Module Structure

Let’s get started by creating a terraform module so that we can reuse in cases where we have more than one account, 
our folder's structure will be something like this:

``` bash
.
├── README.md
├── accounts
│   └── development
│       ├── budgets.tf
│       └── provider.tf     
│       
└── modules
    └── budgets
        ├── main.tf
        ├── services.tf
        └── variables.tf
```

#### Input Variables 

First, create the directory `modules/budgets` and the file `variables.tf`, it will have input variables for our module:

- **Account name:** Name of the AWS account to be tracked.
- **Account budget limit:** Threshold cost for the account level.
- **Services:** List of AWS services to be tracked, we need to inform the name and values for each service.

``` hcl
# modules/budgets/variables.tf

variable "account_name" {}

variable "account_budget_limit" {}

variable "services" {
  description = "List of AWS services to be monitored in terms of costs"

  type = map(object({
    budget_limit = string
  }))
}
```

#### Services Helper

The name of the service should match the name AWS expects otherwise the filter will not work properly, thus, we can create a support map to define some common services (if the service you want is not listed here you can add it later), in the same folder create a new
file called `service.tf`

``` hcl
# modules/budgets/service.tf

variable "aws_services_map" {
  type = map(string)

  default = {
    Athena         = "Amazon Athena"
    EC2            = "Amazon Elastic Compute Cloud - Compute"
    ECR            = "Amazon EC2 Container Registry (ECR)"
    ECS            = "Amazon EC2 Container Service"
    Kubernetes     = "Amazon Elastic Container Service for Kubernetes"
    EBS            = "Amazon Elastic Block Store"
    CloudFront     = "Amazon CloudFront"
    CloudTrail     = "AWS CloudTrail"
    CloudWatch     = "AmazonCloudWatch"
    Cognito        = "Amazon Cognito"
    Config         = "AWS Config"
    DynamoDB       = "Amazon DynamoDB"
    DMS            = "AWS Database Migration Service"
    ElastiCache    = "Amazon ElastiCache"
    Elasticsearch  = "Amazon Elasticsearch Service"
    ELB            = "Amazon Elastic Load Balancing"
    Gateway        = "Amazon API Gateway"
    Glue           = "AWS Glue"
    Kafka          = "Managed Streaming for Apache Kafka"
    KMS            = "AWS Key Management Service"
    Kinesis        = "Amazon Kinesis"
    Lambda         = "AWS Lambda"
    Lex            = "Amazon Lex"
    Matillion      = "Matillion ETL for Amazon Redshift"
    Pinpoint       = "AWS Pinpoint"
    Polly          = "Amazon Polly"
    Rekognition    = "Amazon Rekognition"
    RDS            = "Amazon Relational Database Service"
    Redshift       = "Amazon Redshift"
    S3             = "Amazon Simple Storage Service"
    SFTP           = "AWS Transfer for SFTP"
    Route53        = "Amazon Route 53"
    SageMaker      = "Amazon SageMaker"
    SecretsManager = "AWS Secrets Manager"
    SES            = "Amazon Simple Email Service"
    SNS            = "Amazon Simple Notification Service"
    SQS            = "Amazon Simple Queue Service"
    Tax            = "Tax"
    VPC            = "Amazon Virtual Private Cloud"
    WAF            = "AWS WAF"
    XRay           = "AWS X-Ray"
  }
}
```
Now, we can implement the budgets module, let's start by creating a SNS topic so that when an alarm is triggered, it will send a message in this topic and everyone subscribed in this topic will receive the message, in our case, it will be the email app for slack, we will get into that later.

#### Budget Module

Here we are creating a new topic and defining a policy that allows AWS budgets to Publish Message.

``` hcl
# modules/budgets/main.tf

resource "aws_sns_topic" "account_billing_alarm_topic" {
  name = "account-billing-alarm-topic"
}

resource "aws_sns_topic_policy" "account_billing_alarm_policy" {
  arn    = aws_sns_topic.account_billing_alarm_topic.arn
  policy = data.aws_iam_policy_document.sns_topic_policy.json
}

data "aws_iam_policy_document" "sns_topic_policy" {
  
  statement {
    sid = "AWSBudgetsSNSPublishingPermissions"
    effect = "Allow"

    actions = [
      "SNS:Receive",
      "SNS:Publish"
    ]

    principals {
      type        = "Service"
      identifiers = ["budgets.amazonaws.com"]
    }

    resources = [
      aws_sns_topic.account_billing_alarm_topic.arn
    ]
  }
}
```

Once we have a topic, we can create a budget resource for the account level and subscribe to the topic.

``` hcl
# modules/budgets/main.tf

resource "aws_budgets_budget" "budget_account" {
  name              = "${var.account_name} Account Monthly Budget"
  budget_type       = "COST"
  limit_amount      = var.account_budget_limit
  limit_unit        = "USD"
  time_unit         = "MONTHLY"
  time_period_start = "2020-01-01_00:00"

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_sns_topic_arns = [
      aws_sns_topic.account_billing_alarm_topic.arn
    ]
  }

  depends_on = [
    aws_sns_topic.account_billing_alarm_topic
  ]
}
```

The last resource will be the budgets for services, it will receive a list of services and create a budget for each one.

``` hcl
# modules/budgets/main.tf

resource "aws_budgets_budget" "budget_resources" {
  for_each = var.services

  name              = "${var.account_name} ${each.key} Monthly Budget"
  budget_type       = "COST"
  limit_amount      = each.value.budget_limit
  limit_unit        = "USD"
  time_unit         = "MONTHLY"
  time_period_start = "2020-01-01_00:00"

  cost_filters = {
    Service = lookup(var.aws_services_map, each.key)
  }

  notification {
    comparison_operator       = "GREATER_THAN"
    threshold                 = 100
    threshold_type            = "PERCENTAGE"
    notification_type         = "FORECASTED"
    subscriber_sns_topic_arns = [
      aws_sns_topic.account_billing_alarm_topic.arn
    ]
  }

  depends_on = [
    aws_sns_topic.account_billing_alarm_topic
  ]
}
```

The module is complete, now we can easily reuse in our amazon accounts, to do so, create a `budgets.tf`
in the account folder and import the module, like this:

``` hcl
# accounts/development/budgets.tf

module "billing_alarm" {
  source = "./../../modules/budgets"

  account_name         = "Development"
  account_budget_limit = "20.0"

  services = {
    EC2 = {
      budget_limit = "10.0"
    },
    S3 = {
      budget_limit = "5.0"
    }
  }
}
```

To deploy this infrastructure, go to the development folder and type:

``` bash
$ terraform init
$ terraform plan

# After check the plan command we can apply 
$ terraform apply
```

After applying you should have the budgets as shown below:

![budgets.png](/assets/public/budgets.png)

And the SNS topic:

![sns-topic.png](/assets/public/sns-topic.png)

Last step is to subscribe to this topic via Email app in Slack or any email.

## Slack or Email Integration 

If you have a slack paid plan, you can use the [Email Slack App](https://slack.com/apps/A0F81496D-email) integration, it provides a custom email to be used in the slack channel, thus, every email that is sent to this account will appear in the channel.

I don't have a pro slack plan, so for simplicity, I will use my email for it, however, the process would be the same using the email integration app or email. 

**Note:** This part of the process is not supported by Terraform, so we are going to do it manually. 

### Subscribing the email to the topic

Go to SNS panel in AWS and select `Subscriptions` and create a new subscription by choosing the topic that was created in the previous step and in Protocol chose **email**, after creating the subscription you should receive an email from AWS SNS.

![subscription.png](/assets/public/subscription.png)

From now on, every time you hit a threshold you will receive an email notification like this:

```
AWS Budget Notification May 04, 2020
AWS Account XXXXXXXXX

Dear AWS Customer,

You requested that we alert you when the FORECASTED Cost associated with your 
Development EC2 Monthly Budget is greater than $10.00 for the current month. 

The FORECASTED Cost associated with this budget is $15.00. 

You can find additional details below and by accessing the AWS Budgets dashboard.

Budget Name: Development EC2 Monthly Budget
Budget Type: Cost
Budgeted Amount: $10.00
Alert Type: FORECASTED
Alert Threshold: > $10.00
FORECASTED Amount: $15.00 
```

That's it! 

**Note:** This implementation uses the version 0.12 of Terraform, you can find the same implementation using the version 0.11
[here](https://gist.github.com/rribeiro1/9062cec2e0f89564314ecca771ae1111)
