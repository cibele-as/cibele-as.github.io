---
title: Budgets alarms for AWS accounts and services using Terraform
tags: [aws, budgets, terraform, slack, email]
language: 🇬🇧
---

Who never surprised while checking the billing panel from Amazon, Google, or another cloud provider? It happened to me a couple of times, actually, and I am sure it happens to you as well as in your organization 😅

## Common problems and motivations 💸

It’s not rare that we sometimes think service would cost X, and it ends up costing 2X at the final of the month, sometimes we also create resources manually for experimentation and forget to clean them up. On the other hand, it is not healthy to access the budget console every day to check whether the costs are okay or not, so the best way to avoid surprises would be automating this process and creating some alarms when something goes wrong.

In this post, we will create a process to keep track of our budgets in AWS by defining a threshold in the account level and AWS services and making integration with email or Slack. We are going to use [Terraform](https://www.terraform.io/) and the [Email Slack App](https://slack.com/apps/A0F81496D-email) integration to receive alarms from our account to a specific channel in Slack.

## Prerequisites

This guide assumes some basic familiarity with the usual Terraform workflow (init, plan and apply). It also takes that you have your terraform and AWS CLI adequately configured.

## Implementation Scenario

For our study case, let's say that we have a `development account` on AWS, and we used to use two services `EC2` and `S3`, which costs $20/month:

- **EC2:** $ 10.00 
- **S3:** $ 5.00

We also want to reserve $5,00 for eventual costs. In the end, what we need to achieve is:

Receive an alert when:
- Forecast amount for the `Development account` is `greater than 20.00`
- Forecast amount for `EC2` is `greater than $10.00`
- Forecast amount for `S3` is `greater than $5.00`

After implementing the module in terraform, we are going to use it like this:

```tf
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

## Before Starting

If you in a hurry, this module is ready to use in the Terraform Registry. Check in the link below.

{% include elements/button.html link="https://registry.terraform.io/modules/rribeiro1/budget-alarms/aws" text="Terraform Registry" block=true  style="primary" size="sm" %}

## Terraform Module Structure

Let's get started by creating a Terraform module so that we can reuse it in cases where we have more than one account, our folder's structure will be something like this:

``` bash
.
├── accounts
│        └── development
│            ├── budgets.tf
│            └── provider.tf # We are not configuring this in this post.
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

```tf
# modules/budgets/variables.tf

variable "account_name" {
  type = string
}

variable "account_budget_limit" {
  type = string
}

variable "services" {
  description = "List of AWS services to be monitored in terms of costs"

  type = map(object({
    budget_limit = string
  }))
}
```

#### Services Helper

The name of the service should match the name AWS expects. Otherwise, the filter will not work correctly. Thus, we can create a support map to define some standard services (if the service you want is not listed here, you can add it later). In the same folder, create a new file called `service.tf`

```tf
# modules/budgets/service.tf

locals {
  aws_services = {
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
Now, we can implement the budget module. Let's start by creating an SNS topic so that when an alarm is triggered, it will send a message on this topic, and everyone subscribed to this topic will receive the message, in our case, it will be the email app for slack, we will get into that later.

#### Budget Module

Here we are creating a new topic and defining a policy that allows AWS budgets to Publish Messages.

```tf
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

```tf
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

The last resource will be the budgets for services. It will receive a list of services and create a budget entry for each one.

```tf
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
    Service = lookup(local.aws_services, each.key)
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

The module is complete. Now we can easily reuse in our amazon accounts, to do so, create a `budgets.tf`
in the account folder and import the module, like this:

```tf
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

After applying, you should have the budgets as shown below:

![budgets.png](/assets/public/budgets.png)

And the SNS topic:

![sns-topic.png](/assets/public/sns-topic.png)

Last step is to subscribe to this topic via Email app in Slack or any email.

## Slack or Email Integration 

If you have a slack paid plan, you can use the [Email Slack App](https://slack.com/apps/A0F81496D-email) integration. It provides a custom email to be used in the slack channel. Thus, every email that is sent to this account will appear in the channel.

I don't have a pro slack plan, so for simplicity, I will use my email for it. However, the process would be the same using the email integration app or email.

**Note:** Terraform does not support this part of the process, so that we will do it manually.

### Subscribing the email to the topic

Go to the SNS panel in AWS and select `Subscriptions` and create a new subscription by choosing the topic that was created in the previous step and in Protocol chose **email**, after completing the subscription, you should receive an email from AWS SNS.

![subscription.png](/assets/public/subscription.png)

From now on, every time you hit a threshold, you will receive an email notification like this:

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
