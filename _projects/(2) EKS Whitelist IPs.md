---
name: Orb to whitelist CircleCI public IP on AWS EKS Cluster
tools: [CircleCI, Orb, AWS]
image: https://circleci.com/blog/media/DesigningPackageManager.png
description: This CircleCI orb aims to get the public ip address of the current CircleCI build machine and add it in the source whitelist on EKS cluster. 
---

# aws-eks-whitelist-circleci-ip

This CircleCI orb aims to get the public IP address of the current CircleCI build machine and add it in the source whitelist on the EKS cluster. 

The orb is available in the [CircleCI Orb Registry](https://circleci.com/orbs/registry/orb/rribeiro1/aws-eks-whitelist-circleci-ip) 

## Motivation

After creating an Amazon Elastic Kubernetes Cluster (EKS) we receive a public endpoint to access the cluster and by 
default it is open to all traffic `0.0.0.0/0`. to date, it is not managed using Security groups so that to restrict 
this access to a specific CIDR, we should either use the web console or AWS CLI.

For the sake of security, we should only allow traffic coming from addresses we trusted on to reach the public endpoint 
in EKS, for instance, the company VPN CIDR for internal access or, in this case, CircleCI access to create deployments 
as part of continuous integration, here we face another problem, to date, CircleCI does not have a static IP as 
discussed [here](https://support.circleci.com/hc/en-us/articles/115014372807-IP-Address-ranges-for-whitelisting-Do-you-have-static-IP-addresses-available-), 
thus, one of the possibilities to implement this could be using **Dynamic Whitelisting**.

## Dynamic Whitelisting

Basically, we dynamically fetch the current IP address from the builder machine and add it to the source whitelist in EKS, at the end of the build or if something goes wrong during this process, we remove the IP address to prevent having leftover IPs.

### IAM Setup

In order to use the orb we should create an IAM role with a policy that grants access on EKS, the example below shows the minimum privilege required to make it work properly.

``` json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": [
                "eks:UpdateClusterConfig"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:eks:{Region}:{Account}:cluster/{ClusterName}/update-config"
        },
        {
            "Action": [
                "eks:DescribeUpdate"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:eks:{Region}:{Account}:cluster/{ClusterName}/update/*"
        },
        {
            "Action": [
                "eks:DescribeCluster"
            ],
            "Effect": "Allow",
            "Resource": "arn:aws:eks:{Region}:{Account}:cluster/{ClusterName}"
        }
    ]
}
```

Role content example:

``` json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::{TrustedAccount}:root"
      },
      "Action": "sts:AssumeRole"
    }
  ]
}
```

## How to use

The example below shows how to use the Orb in the CircleCI pipeline.

Add those environment variables in the project, this user should have access to assume the role described above. 

- `AWS_ACCESS_KEY_ID` Access key from the user on AWS 
- `AWS_SECRET_ACCESS_KEY` Secret access key from the user on AWS 

``` yaml
version: 2.1
orbs:
  aws-cli: circleci/aws-cli@0.1.13
  aws-eks-whitelist-circleci-ip: rribeiro1/aws-eks-whitelist-circleci-ip@1.0.0
jobs:
  build:
    docker:
      - image: circleci/python:stretch
    working_directory: ~/app
    steps:
      - checkout
      - aws-cli/install

      - aws-eks-whitelist-circleci-ip/include:
          cluster-name: eks-cluster-name
          aws-region: eu-central-1
          custom-cidrs: "1.1.1.1/32, 2.2.2.2/32"
          role-arn: arn:aws:iam::1111111111111:role/EksCircleCiRole

      - aws-eks-whitelist-circleci-ip/remove:
          cluster-name: eks-labtwin-staging-cluster
          aws-region: eu-central-1
          keep-cidrs: "1.1.1.1/32"
          role-arn: arn:aws:iam::1111111111111:role/EksCircleCiRole
```

<p class="text-center">
  {% include elements/button.html link="https://github.com/rribeiro1/eks-whitelist-ips" text="Check the Source Code" %}
</p>