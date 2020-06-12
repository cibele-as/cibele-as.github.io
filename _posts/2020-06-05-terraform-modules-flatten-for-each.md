---
title: "Creating reusable modules in Terraform"
tags: [aws, terraform, for, for_each, flatten]
style: fill
color: primary
description: How to leverage the power of flatten function and for_each and create awesome modules in Terraform. 
---

In the previous [post](https://rafaelribeiro.io/blog/terraform-loops-count-foreach) we've covered some aspects of the `count` parameter as well as its common problems and how to use `for_each` to solve them, we walked through a scenario where it was necessary to create a list of users in AWS IAM.

At that point our ficticious company only had one team, but now the company decided to invest in a devops team in order to better manage the infrastructure as code as it is growing faster.

In this post we are going to learn how to create reusable terraform modules 

By combining `for`, `flatten` and `for_each` we can even build more complex modules, that are easy to mantain, let’s leverage the `users module` that we have created previosly and now create a new module to handle `groups` and `policies` resources so that we can define AWS Groups, add users to it and apply policies to each group. What we want to achieve is that: 

``` tf
module “groups” {
  source = “../modules/groups”

  groups = {
    administrators = {
      name     = “Administrators”,
      users    = [“rafael-ribeiro”, “cibele-alves”]
      policies = [
          “AllowAccessTranscribeWebsocket”
        ]
    },
    developers = {
      name     = “Developers”,
      users    = [“caio-myiashiro”]
      policies = [
          “AdministratorAccessPolicy”
        ]
    }
  }
}
```