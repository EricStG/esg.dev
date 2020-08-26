---
title: "State management with Terraform"
date: 2019-09-21T19:57:59-04:00
summary: An horror story about state management in Terraform
tags:
- Terraform
---
[Terraform](https://www.terraform.io/) is a great tool. While I'm not super found of [YAML](https://yaml.org/), the syntax allows for organizing your infrastructure in a clean way.

As as they say, cleanliness is Godliness.

However, there is something that I was was more explicit in the guides and tutorials I was following.

## State management in Terraform

Terraform keeps track of whatever action was applied through it's [state](https://www.terraform.io/docs/state/index.html). By default, this state is stored on your local drive where your templates are stored.

This is what happened to me:

1. Setup the terraform templates
2. Plan and apply them
3. Do some tweaks, add some stuff
4. Plan & apply
5. Repeat 3-4 a bunch of times

Now I'm happy with the template, everything is good. Let's throw that in an Azure DevOps pipeline.

This, my friends, is where things went horribly terrible wrong.

Since the state was on my local drive, Terraform tried recreating my resources every time the template was applied. And it was applied successfully somehow, too! The result was that now I had a bunch of copies of the same resource. Route 53 zones with the same names, S3 buckets too. It wasn't great.

Lesson learned! Setup a [backend](https://www.terraform.io/docs/backends/index.html).

Terraform allows to store the state in share space, such as S3 buckets. Note that there's a new-ish offering [Terraform Cloud](https://www.terraform.io/docs/cloud/index.html) which will allow you to not only store the state, but manage your configuration and provisioning, but that's a story for another time...

I decided to use S3 to host the state since I was planning to host the site on AWS anyway. All I had to do was create a new S3 bucket in my AWS account and modify my Terraform configuration to include a `backend` section.

Here's how it looks:

```HCL
terraform {
  backend "s3" {
    bucket = "blog-terraform-state" # The name of the bucket to use
    key    = "terraform.tfstate" # The name of the file within the bucket
    region = "us-east-1" # The AWS region to store the bucket in
  }
}
```

Once I had the backend properly configured, it didn't matter if I applied the configuration locally or if I did through a remote pipeline. They all shared the same state, and this lived in harmony forever and ever.
