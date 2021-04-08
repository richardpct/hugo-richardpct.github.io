---
title: "AWS with Terraform tutorial 06"
date: 2021-04-05T18:33:40Z
draft: false
---

## Purpose
This tutorial takes up the previous one
[aws-with-terraform-tutorial-05](https://richardpct.github.io/post/2021/03/27/aws-with-terraform-tutorial-05/)
by leveraging the high availability feature provided by AWS.
Imagine that your bastion or your webserver are crashing for any reasons, they
will be automatically recreated using autoscaling group, hence your service
will experience a short downtime.<br />
For keeping the same EIP (Elastic IP) when a new instance will replace the old
one, this instance requires to perform an aws command that associates the EIP
with itself.<br />
In addition, I no longer use a Redis server using an EC2, instead I will use 
the Elastic Cache service provided by AWS.

The following figure depicts the infrastructure you will build:

<img src="https://raw.githubusercontent.com/richardpct/images/master/aws-tuto-06/image01.png">

The source code can be found [here](https://github.com/richardpct/aws-terraform-tuto06).

## Configuring the network

#### environments/dev/00-base/main.tf

The following code shows how the subnets are configured:

```
module "base" {
  source = "../../../modules/base"

  region                  = "eu-west-3"
  env                     = "dev"
  vpc_cidr_block          = "10.0.0.0/16"
  subnet_public_bastion_a = "10.0.0.0/24"
  subnet_public_bastion_b = "10.0.1.0/24"
  subnet_public_web_a     = "10.0.2.0/24"
  subnet_public_web_b     = "10.0.3.0/24"
  subnet_private_redis_a  = "10.0.4.0/24"
  subnet_private_redis_b  = "10.0.5.0/24"
  cidr_allowed_ssh        = var.my_ip_address
  ssh_public_key          = var.ssh_public_key
}
```

As you can see later, each service requires to have 2 subnets, one is located in
the availability zone A and the second is located in the availability zone B.

#### modules/base/network.tf

I declare the 2 subnets in which the bastion will be hosted, and each subnet is
located in distinct availability zone:

```
resource "aws_subnet" "public_bastion_a" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = var.subnet_public_bastion_a
  availability_zone = data.aws_availability_zones.available.names[0]

  tags = {
    Name = "subnet_public_bastion_a-${var.env}"
  }
}

resource "aws_subnet" "public_bastion_b" {
  vpc_id            = aws_vpc.my_vpc.id
  cidr_block        = var.subnet_public_bastion_b
  availability_zone = data.aws_availability_zones.available.names[1]

  tags = {
    Name = "subnet_public_bastion_b-${var.env}"
  }
}
```

The webserver subnets and the redis server subnets are created by the same way.

## Creating an IAM role

#### modules/base/iam.tf

When the state of a server is changing to off for any reason, a new public IP
is associated to the server which will replace it. When using a bastion or a
web server, it is handy to keep the same public IP, to accomplish it, the EC2
requires the right to associate the existing EIP by declaring the following IAM
role:

```
resource "aws_iam_role" "role" {
  name = "my_role"

  assume_role_policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Action": "sts:AssumeRole",
      "Principal": {
        "Service": "ec2.amazonaws.com"
      },
      "Effect": "Allow"
    }
  ]
}
EOF
}

resource "aws_iam_policy" "policy" {
  name = "my_policy"

  policy = <<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ec2:AssociateAddress"
      ],
      "Resource": "*"
    }
  ]
}
EOF
}

resource "aws_iam_role_policy_attachment" "attach" {
  role       = aws_iam_role.role.name
  policy_arn = aws_iam_policy.policy.arn
}

resource "aws_iam_instance_profile" "profile" {
  name = "my_profile"
  role = aws_iam_role.role.name
}
```

## Creating the bastion

#### modules/bastion/main.tf

The following code intends to create a bastion by using autoscaling group to
ensure to have one server up and running, if it fails or is deleted for any
reason, a new one is recreated:

```
data "template_file" "user_data" {
  template = file("${path.module}/user-data.sh")

  vars = {
    eip_bastion_id = data.terraform_remote_state.base.outputs.aws_eip_bastion_id
  }
}

resource "aws_launch_configuration" "bastion" {
  name                        = "bastion-${var.env}"
  image_id                    = var.image_id
  user_data                   = data.template_file.user_data.rendered
  instance_type               = var.instance_type
  key_name                    = data.terraform_remote_state.base.outputs.ssh_key
  security_groups             = [data.terraform_remote_state.base.outputs.sg_bastion_id]
  iam_instance_profile        = data.terraform_remote_state.base.outputs.iam_instance_profile_name
  associate_public_ip_address = true

  lifecycle {
    create_before_destroy = true
  }
}

resource "aws_autoscaling_group" "bastion" {
  name                 = "asg_bastion-${var.env}"
  launch_configuration = aws_launch_configuration.bastion.id
  vpc_zone_identifier  = [data.terraform_remote_state.base.outputs.subnet_public_bastion_a_id, data.terraform_remote_state.base.outputs.subnet_public_bastion_b_id]
  min_size             = 1
  max_size             = 1

  tag {
    key                 = "Name"
    value               = "bastion-${var.env}"
    propagate_at_launch = true
  }
}
```

As you can see, in the last resource I use 2 subnets in distinct AZ, if a AZ is
experiencing some issues, the server is recreated in the other AZ.

#### modules/bastion/user-data.sh

The last line intends to associate an existing EIP in order to keep the same
public IP whenever the instance is recreated:

```
#!/bin/bash

exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
sudo yum -y update
sudo yum -y upgrade
INSTANCE_ID="$(curl -s http://169.254.169.254/latest/meta-data/instance-id)"
aws --region eu-west-3 ec2 associate-address --instance-id $INSTANCE_ID --allocation-id ${eip_bastion_id}
```

## Creating the web server

The build of the web server is similar to the bastion server.

## Creating the Redis server

Let's create a Redis server for storing the requests count using the Elastic
Cache service provided by AWS:

```
resource "aws_elasticache_subnet_group" "redis" {
  name       = "subnet-redis-${var.env}"
  subnet_ids = [data.terraform_remote_state.base.outputs.subnet_private_redis_a_id, data.terraform_remote_state.base.outputs.subnet_private_redis_b_id]
}

resource "aws_elasticache_cluster" "redis" {
  cluster_id           = "cluster-redis"
  engine               = "redis"
  node_type            = var.instance_type
  num_cache_nodes      = 1
  parameter_group_name = "default.redis6.x"
  engine_version       = "6.x"
  port                 = 6379
  subnet_group_name    = aws_elasticache_subnet_group.redis.name
  security_group_ids   = [data.terraform_remote_state.base.outputs.sg_database_id]
}
```

## Deploying the infrastructure

Export the following environment variables:

    $ export TF_VAR_region="eu-west-3"
    $ export TF_VAR_bucket="yourbucket-terraform-state"
    $ export TF_VAR_dev_base_key="terraform/dev/base/terraform.tfstate"
    $ export TF_VAR_dev_bastion_key="terraform/dev/bastion/terraform.tfstate"
    $ export TF_VAR_dev_database_key="terraform/dev/database/terraform.tfstate"
    $ export TF_VAR_dev_webserver_key="terraform/dev/webserver/terraform.tfstate"
    $ export TF_VAR_ssh_public_key="ssh-rsa XXX..."
    $ export TF_VAR_my_ip_address=$(curl -s 'https://duckduckgo.com/?q=ip&t=h_&ia=answer' \
    | sed -e 's/.*Your IP address is \([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\) in.*/\1/')

Building:

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

You need to perform `terraform init` once.

## Testing your infrastructure

When your infrastructure is built, get the EIP of your web server by performing
the following command:

    $ aws ec2 describe-addresses --filters "Name=tag:Name,Values=eip_web-dev" \
      --query 'Addresses[*].PublicIp' \
      --output text

Perform the following command until the output matches the EIP of your web
server:

    $ aws ec2 describe-instances --filters "Name=tag-value,Values=webserver-dev" \
      --query 'Reservations[*].Instances[*].NetworkInterfaces[*].PrivateIpAddresses[*].Association.PublicIp' \
      --output text

Then issue the following command several times for increasing the counter:

    $ curl http://ip_public_webserver:8000/cgi-bin/hello.py

It should return the count of requests you have performed.

## Testing the High Availability

Get the instance ID of your web server:

    $ aws ec2 describe-instances --filters "Name=tag-value,Values=webserver-dev" "Name=instance-state-name,Values=running" \
      --query "Reservations[*].Instances[*].InstanceId" \
      --output text

Terminate your instance using its ID:

    $ aws ec2 terminate-instances --instance-ids id-instance_web_server

Then wait until the following command returns the ID instance of your new web
server:

    $ aws ec2 describe-instances --filters "Name=tag-value,Values=webserver-dev" "Name=instance-state-name,Values=running" \
      --query "Reservations[*].Instances[*].InstanceId" \
      --output text

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

In this tutorial you have learned how to use the auto scaling group in order
to ensure one server is up and running, but the downside of this way is
whenever your server is recreated, your service is experiencing a short
downtime.<br />
In the next tutorial I will show you how to improve the method using a load
balancer.
