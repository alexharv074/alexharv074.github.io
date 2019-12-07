---
layout: post
title: "Adventures in the Terraform DSL, Part IX: Data sources"
date: 2019-10-20
author: Alex Harvey
tags: terraform
---

Part IX of my blog series on the Terraform DSL, where I look at _data sources_.

* ToC
{:toc}

## Introduction

This post is about Terraform _data sources_, also known as _data resources_, a feature that was introduced in Terraform 0.7 in May 2016 written by Martin Atkins. In here, I look at the history, motivation and usage of this important feature.

## Overview

A _data source_ a.k.a. _data resource_ looks and behaves much like an ordinary resource, but presents a read-only view of dynamic data that comes from outside of Terraform.

The data that is made available in this way is called _fetched data_ because it is "fetched" during the refresh stage of the Terraform lifecycle. It should not be confused with _computed data_ that is generated by resources like the `random_id` that we saw earlier in this series. Computed data is created by Terraform during the apply stage of the lifecycle.

## The first data source: terraform_remote_state

Before I get to data sources, I want to distinguish them from the _logical resources_ that they grew out of. To do that I'll look at the very first data source that was added in Terraform, the [terraform_remote_state](https://www.terraform.io/docs/providers/terraform/d/remote_state.html) resource. Here is the example from the Terraform 0.6 docs:

```js
resource "terraform_remote_state" "vpc" {
  backend = "atlas"
  config {
    path = "hashicorp/vpc-prod"
  }
}

resource "aws_instance" "foo" {
  // ...
  subnet_id = "${terraform_remote_state.vpc.output.subnet_id}"
}
```

This was a _logical resource_, a resource that "contributes to Terraform state but does not manage an external resource". Implemented as a logical resource, it was less clear to the reader that, in this case, remote state was a source of data rather than a managed resource.

In Terraform 0.7, this resource was changed to be the first data source. Instead of the resource declaration, it became this:

```js
data "terraform_remote_state" "vpc" {
  backend = "atlas"
  config {
    name = "hashicorp/vpc-prod"
  }
}

resource "aws_instance" "foo" {
  // ...
  subnet_id = data.terraform_remote_state.vpc.subnet_id // 0.12 syntax here.
}
```

The key differences in usage are:

- The block type changed from `resource` to `data`.
- The reference changed from `<TYPE>.<NAME>.<ATTRIBUTE>` to `data.<TYPE>.<NAME>.<ATTRIBUTE>`.

I found it helpful to study the actual [commit](https://github.com/hashicorp/terraform/commit/3eb4a89104ba6c41f305af425ce91f19d4f35f4c) that changed this first data source from a logical resource. It makes it clearer that under the hood, a data source really is just a special resource that is read-only. The data source [still](https://github.com/hashicorp/terraform/blob/3eb4a89104ba6c41f305af425ce91f19d4f35f4c/builtin/providers/terraform/data_source_state.go#L11-L32) returns under the hood a Resource schema:

```go
func dataSourceRemoteState() *schema.Resource {
  return &schema.Resource{
    Read: dataSourceRemoteStateRead,

    Schema: map[string]*schema.Schema{
      "backend": &schema.Schema{
        Type:     schema.TypeString,
        Required: true,
      },

      "config": &schema.Schema{
        Type:     schema.TypeMap,
        Optional: true,
      },

      "output": &schema.Schema{
        Type:     schema.TypeMap,
        Computed: true,
      },
    },
  }
}
```

## Data source examples

Let's look more at how they're actually used. One common use of the data sources is to fetch an AWS AMI via the `aws_ami` data source. Here is how you can fetch an Amazon Linux 2 AMI ID:

```js
data "aws_ami" "amazon_linux_2" {
  most_recent = "true"

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-ebs"]
  }

  owners = ["amazon"]
}
```

The structure of this declaration feels familiar to users of the AWS CLI. I apply that:

```text
▶ terraform apply -auto-approve
data.aws_ami.amazon_linux_2: Refreshing state...

Apply complete! Resources: 0 added, 0 changed, 0 destroyed.

Outputs:

ami_id = ami-0804dc420cb24c62b
```

For AWS users, it is useful to convert some of the AWS data source Terraform declarations into AWS CLI:

```text
▶ aws ec2 describe-images --filters "Name=name,Values=amzn2-ami-hvm-*-x86_64-ebs" \
    --owners amazon --query 'reverse(sort_by(Images, &CreationDate))[0].ImageId'
"ami-0804dc420cb24c62b"
```

Very similar, which is not surprising considering that Terraform and AWS CLI are calling the same AWS API of course.

## Data sources docs

The next thing to know about data sources is how to find the docs. The docs contain a complete list of all data sources for each provider, for each AWS service, etc, and all of their attributes are documented.

The data sources are generally defined in the providers. For AWS, start at the [AWS Provider](https://www.terraform.io/docs/providers/aws/index.html) page:

![Screenshot 1]({{ "/assets/terraform_docs_1.png" | absolute_url }})

Then go down and click on one of the AWS services e.g. ACM:

![Screenshot 2]({{ "/assets/terraform_docs_2.png" | absolute_url }})

And from there all the data sources for that AWS service can be seen, in this case the [aws_acm_certificate](https://www.terraform.io/docs/providers/aws/d/acm_certificate.html) data source that can return the ARN of a certificate in AWS Certificate Manager (ACM).

## Local-only data sources: template_file

Another kind of data source is the "local-only data source", a data source that fetches data from the local machine that is running Terraform, rather than the Cloud or network. I make a special mention of the commonly used [template_file](https://www.terraform.io/docs/providers/template/d/file.html) data source. It is actually deprecated now in favour of the `templatefile()` function - and I will discuss this more in the next part of my series which will be on Terraform's template language - but it is still common to see templates declared like this in Terraform:

```js
data "template_file" "user_data" {
  template = file("${path.module}/template/user_data.sh.tpl")
  vars = {
    foo = var.foo
    bar = var.bar
  }
}
```

That template can then be referenced as:

```js
resource "aws_instance" "web" {
  ami           = "ami-0804dc420cb24c62b"
  instance_type = "t2.micro"
  user_data     = data.template_file.user_data.rendered
}
```

I am slightly disappointed that this is deprecated because, to me, this is cleaner! More on that later.

## Conclusion

Well that is the end of this shorter-than-usual post on data sources a.k.a. data resources. So far, data sources is one of my favourite Terraform features and they do provide a clean way of getting dynamic data from the AWS Cloud and other places. We have seen that they are really just a special kind of resource, distinguished mostly for readability by the `data` declaration, and that these export _fetched data_ only and no _computed data_.

In Part X I will look at Terraform's template language and related template functions some more.

## See also

- Mark Burke, Aug 23 2018, [Get the latest AWS AMI IDs with Terraform](https://letslearndevops.com/2018/08/23/terraform-get-latest-centos-ami/).
- Ken Lucas, Sep 28 2018, [Stack Overflow answer - Fetched vs computed values in Terraform data sources](https://stackoverflow.com/a/52561313/3787051).