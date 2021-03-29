---
title: "AWS with Terraform tutorial 05"
date: 2021-03-27T20:01:59Z
draft: false
---

## Purpose
This tutorial takes up the previous one
[aws-with-terraform-tutorial-04](https://richardpct.github.io/post/2021/03/16/aws-with-terraform-tutorial-04/)
by adding a bastion server which is the only one that is allowed to connect to
the database and the webserver via SSH for improving the security of your
infrastructure.

Here are the components you will build:

* Creating a VPC with 3 public subnets and a private subnet
* Creating a bastion server in a public subnet which is the only server that
can be reachable from the Internet via SSH, and it is the only that is allowed
to connect to the database and the webserver via SSH
* Creating a web server in a public subnet
* Creating a database server using Redis which stores the count of requests
* Creating a NAT gateway with an Elastic IP (EIP) in a public subnet so that
the database which is in the private subnet is able to reach Internet

Notice: For the exercise I do not use "Elastic Cache" service provided by AWS,
instead I use an EC2 instance and I install Redis manually.

The following figure depicts the infrastructure you will build:

<img src="https://raw.githubusercontent.com/richardpct/images/master/aws-tuto-05/image01.png">

The Terraform code can be found [here](https://github.com/richardpct/aws-terraform-tuto05).

## Configuring the network

#### environments/dev/00-network/main.tf

I could create a public subnet containing all the services (Nat Gateway, web
service and bastion) but it's better to isolate each stack for reducing the
concerns, to do so, I create a first public subnet containing a Nat Gateway, a
second public subnet containing the bastion and a third public subnet
containing a web server:

```
module "network" {
  source = "../../../modules/network"

  region                = "eu-west-3"
  env                   = "dev"
  vpc_cidr_block        = "10.0.0.0/16"
  subnet_public_nat     = "10.0.0.0/24"
  subnet_public_bastion = "10.0.1.0/24"
  subnet_public_web     = "10.0.2.0/24"
  subnet_private        = "10.0.3.0/24"
  cidr_allowed_ssh      = var.my_ip_address
}
```

I will show you only the excerpt code related to the SSH rules, the other rules
are similar to the previous tutorial. The following rules allow your own IP to
connect to the bastion, and the bastion is allow to connect to the web server
and the database:

```
resource "aws_security_group_rule" "bastion_inbound_ssh" {
  type              = "ingress"
  from_port         = local.ssh_port
  to_port           = local.ssh_port
  protocol          = "tcp"
  cidr_blocks       = [var.cidr_allowed_ssh]
  security_group_id = aws_security_group.bastion.id
}

resource "aws_security_group_rule" "bastion_outbound_ssh" {
  type              = "egress"
  from_port         = local.ssh_port
  to_port           = local.ssh_port
  protocol          = "tcp"
  cidr_blocks       = local.anywhere
  security_group_id = aws_security_group.bastion.id
}

resource "aws_security_group_rule" "database_inbound_ssh" {
  type                     = "ingress"
  from_port                = local.ssh_port
  to_port                  = local.ssh_port
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion.id
  security_group_id        = aws_security_group.database.id
}

resource "aws_security_group_rule" "webserver_inbound_ssh" {
  type                     = "ingress"
  from_port                = local.ssh_port
  to_port                  = local.ssh_port
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.bastion.id
  security_group_id        = aws_security_group.webserver.id
}
```

## Creating the bastion

#### modules/bastion/main.tf

The bastion is created like a web server, except that we don't install anything:

```
resource "aws_instance" "bastion" {
  ami                    = var.image_id
  user_data              = <<-EOF
                           #!/bin/bash
                           exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
                           sudo yum -y update
                           sudo yum -y upgrade
                           EOF
  instance_type          = var.instance_type
  key_name               = aws_key_pair.deployer.key_name
  subnet_id              = data.terraform_remote_state.network.outputs.subnet_public_bastion_id
  vpc_security_group_ids = [data.terraform_remote_state.network.outputs.sg_bastion_id]

  tags = {
    Name = "bastion-${var.env}"
  }
}

resource "aws_eip" "bastion" {
  instance = aws_instance.bastion.id
  vpc      = true

  tags = {
    Name = "eip_bastion-${var.env}"
  }
}
```

## Deploying the infrastructure

Export some environment variables:

    $ export TF_VAR_region="eu-west-3"
    $ export TF_VAR_bucket="yourbucket-terraform-state"
    $ export TF_VAR_dev_network_key="terraform/tuto05/dev/network/terraform.tfstate"
    $ export TF_VAR_dev_bastion_key="terraform/tuto05/dev/bastion/terraform.tfstate"
    $ export TF_VAR_dev_database_key="terraform/tuto05/dev/database/terraform.tfstate"
    $ export TF_VAR_dev_webserver_key="terraform/tuto05/dev/webserver/terraform.tfstate"
    $ export TF_VAR_ssh_public_key="ssh-rsa..."
    $ export TF_VAR_my_ip_address=$(curl -s 'https://duckduckgo.com/?q=ip&t=h_&ia=answer' \
    | sed -e 's/.*Your IP address is \([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\) in.*/\1/')

building:

    $ cd environments/dev
    $ cd 00-network
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_dev_network_key}" \
        -backend-config="region=${TF_VAR_region}"
    $ terraform apply
    $ cd ../01-bastion
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_dev_bastion_key}" \
        -backend-config="region=${TF_VAR_region}"
    $ terraform apply
    $ cd ../02-database
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_dev_database_key}" \
        -backend-config="region=${TF_VAR_region}"
    $ terraform apply
    $ cd ../03-webserver
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_dev_webserver_key}" \
        -backend-config="region=${TF_VAR_region}"
    $ terraform apply

## Testing your infrastructure

When your infrastructure is built, wait for a while, then issue the following
command several times for increasing the counter:

    $ curl http://ip_public_webserver:8000/cgi-bin/hello.py

It should return the count of requests you have performed.

## Connecting to the database and the web server via SSH

    $ ssh -J ec2-user@public_ip_bastion ubuntu@private_ip_database
    $ ssh -J ec2-user@public_ip_bastion ec2-user@private_ip_webserver

## Destroying your infrastructure

After finishing your test, destroy your infrastructure:

    $ cd environments/dev
    $ cd 03-webserver
    $ terraform destroy
    $ cd ../02-database
    $ terraform destroy
    $ cd ../01-bastion
    $ terraform destroy
    $ cd ../00-network
    $ terraform destroy

## Summary

We have seen how to improve the security of our infrastructure by using a
bastion for connecting to the servers.<br />
In the next tutorial you will learn how to use the high availability feature
provided by AWS.
