---
title: "AWS with Terraform tutorial 03"
date: 2021-03-06T19:34:35Z
draft: false
---

## Purpose

I will introduce you a way to split your infrastructure in distinct environment
by leveraging Terraform modules.<br />
In this tutorial I use the same AWS account to handle multiple environments in
order to simplify the exercise, but in the real life you must use a AWS account
for each environment for increasing isolation.

I take up the previous tutorial by adding some improvements:

* Splitting our infrastructure by environment: Dev and Staging
* Using user-data for automating Nginx installation
* Allowing only my IP own address to connect via SSH within the webserver

Here is an overview of the infrastructure you will build in this tutorial:

<img src="https://raw.githubusercontent.com/richardpct/images/master/aws-tuto-03/image01.png">

The source code is available on my [Github repository](https://github.com/richardpct/aws-terraform-tuto03).

## Create 2 environments: Dev and Staging

Here is the new layout of our Terraform code:

```
.
├── environments
│   ├── dev
│   │   ├── 01-network
│   │   │   ├── backends.tf
│   │   │   ├── main.tf
│   │   │   ├── outputs.tf
│   │   └── 02-webserver
│   │       ├── backends.tf
│   │       ├── main.tf
│   │       ├── outputs.tf
│   │       └── vars.tf
│   └── staging
│       ├── 01-network
│       │   ├── backends.tf
│       │   ├── main.tf
│       │   ├── outputs.tf
│       └── 02-webserver
│           ├── backends.tf
│           ├── main.tf
│           ├── outputs.tf
│           └── vars.tf
└── modules
    ├── network
    │   ├── main.tf
    │   ├── outputs.tf
    │   ├── vars.tf
    │   └── version.tf
    └── webserver
        ├── backends.tf
        ├── main.tf
        ├── outputs.tf
        ├── user-data.sh
        ├── vars.tf
        └── version.tf
```

The code within the module repository is almost the same as the code of the
[first tutorial](https://richardpct.github.io/post/2021/02/20/aws-with-terraform-tutorial-01/),
in which we defined 2 modules: network and webserver.<br />
The code within environments/dev will create the infrastructure in dev
environment and the code within environments/staging will create the staging
one by using the module network and the module webserver.

### Dev environment

#### environments/dev/01-network/main.tf

We build our network stack in Dev environment using the network module with
specific inputs:

```
module "network" {
  source = "../../../modules/network"

  region         = "eu-west-3"
  env            = "dev"
  vpc_cidr_block = "10.0.0.0/16"
  subnet_public  = "10.0.0.0/24"
}
```

#### environments/dev/02-webserver/main.tf

We build our webserver stack in Dev environment using the webserver module with
specific inputs:

```
module "webserver" {
  source = "../../../modules/webserver"

  region                      = "eu-west-3"
  env                         = "dev"
  network_remote_state_bucket = var.bucket
  network_remote_state_key    = var.dev_network_key
  instance_type               = "t2.micro"
  image_id                    = "ami-0ebc281c20e89ba4b"  //Amazon Linux 2018
  ssh_public_key              = var.ssh_public_key
  cidr_allowed_ssh            = var.my_ip_address
}
```

### Staging environment

#### environments/staging/01-network/main.tf

We build our network stack in Staging environment using the network module with
specific inputs:

```
module "network" {
  source = "../../../modules/network"

  region         = "eu-west-3"
  env            = "staging"
  vpc_cidr_block = "10.1.0.0/16"
  subnet_public  = "10.1.0.0/24"
}
```

#### environments/staging/02-webserver/main.tf

We build our webserver stack in Staging environment using the webserver module
with specific inputs:

```
module "webserver" {
  source = "../../../modules/webserver"

  region                      = "eu-west-3"
  env                         = "staging"
  network_remote_state_bucket = var.bucket
  network_remote_state_key    = var.staging_network_key
  instance_type               = "t2.micro"
  image_id                    = "ami-0ebc281c20e89ba4b"  //Amazon Linux 2018
  ssh_public_key              = var.ssh_public_key
  cidr_allowed_ssh            = var.my_ip_address
}
```

For each environment we use distinct VPC and subnet, likewise the webserver
extracts the bucket network key according to the environment.

## Using user-data to automate the installation of Ningx

#### modules/webserver/user-data.sh

The user-data is the script that will be launched after our instance is
started, we use it to update our Linux system and install the Nginx server:

```
#!/bin/bash

index_html=/usr/share/nginx/html/index.html
exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
sudo yum -y update
sudo yum -y upgrade
sudo yum -y install nginx
cat << EOF > $index_html
Hello World!<br />
Environment: ${environment}
EOF
/etc/init.d/nginx start
```

#### modules/webserver/main.tf

We declare a template_file of user-data because we will pass to our script
a variable `var.env`:

```
data "template_file" "user_data" {
  template = file("${path.module}/user-data.sh")

  vars = {
    environment = var.env
  }
}
```

Then we associate this data to our instance resource:

```
resource "aws_instance" "web" {
  ami                    = var.image_id
  user_data              = data.template_file.user_data.rendered
  instance_type          = var.instance_type
  key_name               = aws_key_pair.deployer.key_name
  subnet_id              = data.terraform_remote_state.network.outputs.subnet_public_id
  vpc_security_group_ids = [aws_security_group.webserver.id]

  tags = {
    Name = "web_server-${var.env}"
  }
}
```

## Restrict only your own IP to access to your webserver

In the Terraform code defining the ingress SSH rule, I set the variable
cidr_blocks as following:

```
resource "aws_security_group_rule" "inbound_ssh" {
  type              = "ingress"
  from_port         = 22
  to_port           = 22
  protocol          = "tcp"
  cidr_blocks       = [var.cidr_allowed_ssh]
  security_group_id = aws_security_group.webserver.id
}
```

Where `var.cidr_allowed_ssh` holds the variable `my_ip_address`, hence we must
export the variable `TF_VAR_my_ip_address` as value your own IP in your shell:

    $ export TF_VAR_my_ip_address=$(curl -s 'https://duckduckgo.com/?q=ip&t=h_&ia=answer' \
    | sed -e 's/.*Your IP address is \([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\) in.*/\1/')

## Deploying the infrastructure

We will build our infrastructure in Dev and Staging environment at the same
time.

Export some environment variables:

    $ export TF_VAR_region="eu-west-3"
    $ export TF_VAR_bucket="yourbucket-terraform-state"
    $ export TF_VAR_dev_network_key="terraform/dev/network/terraform.tfstate"
    $ export TF_VAR_dev_webserver_key="terraform/dev/webserver/terraform.tfstate"
    $ export TF_VAR_staging_network_key="terraform/staging/network/terraform.tfstate"
    $ export TF_VAR_staging_webserver_key="terraform/staging/webserver/terraform.tfstate"
    $ export TF_VAR_ssh_public_key="ssh-rsa AAAA..."
    $ export TF_VAR_my_ip_address=$(curl -s 'https://duckduckgo.com/?q=ip&t=h_&ia=answer' \
    | sed -e 's/.*Your IP address is \([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\) in.*/\1/')

Build the Dev environment:

    $ cd environments/dev
    $ cd 01-network
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_dev_network_key}" \
        -backend-config="region=${TF_VAR_region}"
    $ terraform apply
    $ cd ../02-webserver
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_dev_webserver_key}" \
        -backend-config="region=${TF_VAR_region}"
    $ terraform apply

Build the Staging environment:

    $ cd environments/staging
    $ cd 01-network
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_staging_network_key}" \
        -backend-config="region=${TF_VAR_region}"
    $ terraform apply
    $ cd ../02-webserver
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_staging_webserver_key}" \
        -backend-config="region=${TF_VAR_region}"
    $ terraform apply

After finishing your test, destroy your infrastructure in Dev environment:

    $ cd environments/dev
    $ cd 02-webserver
    $ terraform destroy
    $ cd ../01-network
    $ terraform destroy

Do the same with the Staging environment:

    $ cd environments/staging
    $ cd 02-webserver
    $ terraform destroy
    $ cd ../01-network
    $ terraform destroy

## Summary

We have studied how to use the modules for reproducing our infrastructure in
multiple environments.<br />
In the next tutorial I will introduce you the private subnet.
