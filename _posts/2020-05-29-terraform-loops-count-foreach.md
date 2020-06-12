---
title: "Terraform loops: from count to for_each"
tags: [aws, terraform, for_each, count]
style: fill
color: light
description: Terraform users from version 0.11 may have dynamically created some resources using the count statement, very handy to apply DRY principle and scale resources by simply increment an element in a list or number.
---

Terraform users from version **0.11** may have dynamically created some resources using the `count` statement, very handy to apply [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle and scale resources by simply increment an element in a list or number. 

However, one of the problems we face using `count` is related to {% include elements/highlight.html text="ordering." %} Count maintains an array of numeric index (list) to perform it's operations. If there is a change in the order of the list, terraform will re-create the list again, we will get into that later. 

![alt text](https://media.giphy.com/media/U3CAeMs6h2SEU/giphy.gif)

Let's walk through a scenario to understand it better. 

We want to create a list of users in AWS which represents our team members, for the sake of simplicity, the team is composed of 3 developers. Thus, we probably would start with that:

``` hcl
resource "aws_iam_user" "joao" {
  name = "joao-neves"
}

resource "aws_iam_user" "andre" {
  name = "andre-veron"
}

resource "aws_iam_user" "caio" {
  name = "caio-miyashiro"
}
```

The company is going very well and now 6 developers joint to the team, then you think, maybe creating more 6 users manually would not be a bad idea, however, the scenario describes a simple resource from AWS that requires only the name of the user, imagine another resource which requires dozens of parameters. 

After exploring the alternatives, we find the `count` parameter and decide to extract this to a module `users`.

## **Using `count`**:

We create a module where we will receive a list of users in the variable `users` and then, we get the length of users and assign to count so that we can create user resources dynamically:

``` hcl
# Module users - modules/users.tf

variable "users" {
    type = list(string)
    description = List of developers
}

resource "aws_iam_user" "users" {
  count = length(var.users)
  
  name = var.users[count.index]
}
```

We can re-create users, now using the new module, note that now we can easily increment the list as soon as a new member joins the team:

``` hcl
# Applying the module users - production/users.tf

module "users" {
  source = "../modules/users"

  users  = [
      "joao-neves",
      "andre-veron",
      "caio-miyashiro"
      # rest of the team
  ]
}
```

Typing `terraform plan` we can check that Terraform will create resources as expected:

``` bash
Terraform will perform the following actions:

  # module.users.aws_iam_user.users[0] will be created
  + resource "aws_iam_user" "users" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "joao-neves"
      + path          = "/"
      + unique_id     = (known after apply)
    }

  # module.users.aws_iam_user.users[1] will be created
  + resource "aws_iam_user" "users" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "andre-veron"
      + path          = "/"
      + unique_id     = (known after apply)
    }

  # module.users.aws_iam_user.users[2] will be created
  + resource "aws_iam_user" "users" {
      + arn           = (known after apply)
      + force_destroy = false
      + id            = (known after apply)
      + name          = "caio-miyashiro"
      + path          = "/"
      + unique_id     = (known after apply)
    }

Plan: 3 to add, 0 to change, 0 to destroy.
```

And `terraform apply`:

``` bash
# module.users.aws_iam_user.users[2]: Creating...
# module.users.aws_iam_user.users[1]: Creating...
# module.users.aws_iam_user.users[0]: Creating...
# module.users.aws_iam_user.users[1]: Creation complete after 1s [id=andre-veron]
# module.users.aws_iam_user.users[2]: Creation complete after 1s [id=caio-miyashiro]
# module.users.aws_iam_user.users[0]: Creation complete after 1s [id=joao-neves]
```

Awesome, isn't it? well, until one of the developers leaves the company! 💀

It turns out that João is out of the team and now we have to remove him from the list of users, it should be easy to remove, right? well, maybe not.  

``` hcl
# Applying the module users - production/users.tf

module "users" {
  source = "../modules/users"

  # Removed joao-neves from the list
  users  = [
      "andre-veron",
      "caio-miyashiro"
      # rest of the team
  ]
}
```

After plan our infrastructure again, we would expect to terraform only destroy one resource, however, we can see that not only one resource will be destroyed but also two will be updated. 🤔

``` bash
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  ~ update in-place
  - destroy

Terraform will perform the following actions:

  # module.users.aws_iam_user.users[0] will be updated in-place
  ~ resource "aws_iam_user" "users" {
        arn           = "arn:aws:iam::11111111111:user/joao-neves"
        force_destroy = false
        id            = "joao-neves"
      ~ name          = "joao-neves" -> "andre-veron"
        path          = "/"
        tags          = {}
        unique_id     = "AIDAYMLLNMYWLLJ36G36G"
    }

  # module.users.aws_iam_user.users[1] will be updated in-place
  ~ resource "aws_iam_user" "users" {
        arn           = "arn:aws:iam::11111111111:user/andre-veron"
        force_destroy = false
        id            = "andre-veron"
      ~ name          = "andre-veron" -> "caio-miyashiro"
        path          = "/"
        tags          = {}
        unique_id     = "AIDAYMLLNMYWCXE2REZKI"
    }

  # module.users.aws_iam_user.users[2] will be destroyed
  - resource "aws_iam_user" "users" {
      - arn           = "arn:aws:iam::11111111111:user/caio-miyashiro" -> null
      - force_destroy = false -> null
      - id            = "caio-miyashiro" -> null
      - name          = "caio-miyashiro" -> null
      - path          = "/" -> null
      - tags          = {} -> null
      - unique_id     = "AIDAYMLLNMYWKD2Y3JLT2" -> null
    }

Plan: 0 to add, 2 to change, 1 to destroy.
```

In fact, {% include elements/highlight.html text="Removing elements from lists will force terraform to recreate the lists again." %} Well, here is where the `for_each` statement comes into play, let's see how it solves the problem.

## **Using `for_each`**:

The `for_each` statement was introduced in Terraform 0.12.6 and one of its benefits is to solve problems we had using `count` described previously, the drawback is that `for_each` does not work with lists, so we should convert it to a map or a set of strings.

Let's change our module implementation then.

``` hcl
variable "users" {
  type = map(string) # Type is a map of strings now.
}

resource "aws_iam_user" "users" {
  for_each = var.users
  name = each.value # for each entry we could either use the key or the value.
}
```

And the implementation would be something like that: 

``` hcl
module "users" {
  source = "../../modules/users"

  users = {
    joao  = "joao-neves" ,
    andre = "andre-veron" ,
    caio  = "caio-myiashiro" 
  }
}
```

Now, if we want to remove only the first developer from this list, we should have this output from Terraform:

``` bash
An execution plan has been generated and is shown below.
Resource actions are indicated with the following symbols:
  - destroy

Terraform will perform the following actions:

  # module.users.aws_iam_user.users["joao"] will be destroyed
  - resource "aws_iam_user" "users" {
      - arn           = "arn:aws:iam::11111111111:user/joao-neves" -> null
      - force_destroy = false -> null
      - id            = "joao-neves" -> null
      - name          = "joao-neves" -> null
      - path          = "/" -> null
      - tags          = {} -> null
      - unique_id     = "AIDAYMLLNMYWN4OKLNJVE" -> null
    }

Plan: 0 to add, 0 to change, 1 to destroy.
```

From now on, only the resources removed from the list will be destroyed 😃

![alt text](https://media.giphy.com/media/111ebonMs90YLu/giphy.gif)

In the next post, we are going to combine `for_each`, `for`, and `flatten` to create more complex modules that are easily maintainable and readable. 

See ya!