---
title: "AWS with Terraform tutorial 07"
date: 2021-04-17T18:17:44Z
draft: false
---

## Purpose

This tutorial takes up the previous one
[aws-terraform-tuto06](https://richardpct.github.io/post/2021/04/05/aws-with-terraform-tutorial-06/)
by adding an ALB (Application Load Balancer) in front of the 2 web servers for
sharing the load between 2 web servers and having a short downtime when a web
server is failing.

The following figure depicts the infrastructure you will build:

<img src="https://raw.githubusercontent.com/richardpct/images/master/aws-tuto-07/image01.png">

The source code can be found [here](https://github.com/richardpct/aws-terraform-tuto07).

## Configuring the network

#### environments/dev/00-base/main.tf

The following code shows how the subnets are configured:

```
module "base" {
  source = "../../../modules/base"

  region                  = "eu-west-3"
  env                     = "dev"
  vpc_cidr_block          = "10.0.0.0/16"
  subnet_public_lb_a      = "10.0.0.0/24"
  subnet_public_lb_b      = "10.0.1.0/24"
  subnet_public_nat_a     = "10.0.2.0/24"
  subnet_public_nat_b     = "10.0.3.0/24"
  subnet_public_bastion_a = "10.0.4.0/24"
  subnet_public_bastion_b = "10.0.5.0/24"
  subnet_private_web_a    = "10.0.6.0/24"
  subnet_private_web_b    = "10.0.7.0/24"
  subnet_private_redis_a  = "10.0.8.0/24"
  subnet_private_redis_b  = "10.0.9.0/24"
  cidr_allowed_ssh        = var.my_ip_address
  ssh_public_key          = var.ssh_public_key
}
```

As you can see each service is held into 2 subnets, the first is located in the
Availability Zone A and the second in Availability Zone B.<br />
The load balancer, the Nat gateway and the bastion run in the public subnet
whereas the web server, redis server run in the private subnet.

#### modules/base/network.tf

We create a Nat Gateway in each Availability Zone, the private services located
in the AZ-A will use the Nat Gateway in AZ-A, likewise the private services
located in the AZ-B will use the Nat Gateway in AZ-B, thus if a AZ is
unavailable the service still works:

```
resource "aws_eip" "nat_a" {
  vpc = true

  tags = {
    Name = "eip_nat_a-${var.env}"
  }
}

resource "aws_eip" "nat_b" {
  vpc = true

  tags = {
    Name = "eip_nat_b-${var.env}"
  }
}

resource "aws_nat_gateway" "nat_gw_a" {
  allocation_id = aws_eip.nat_a.id
  subnet_id     = aws_subnet.public_nat_a.id

  tags = {
    Name = "nat_gw_a-${var.env}"
  }
}

resource "aws_nat_gateway" "nat_gw_b" {
  allocation_id = aws_eip.nat_b.id
  subnet_id     = aws_subnet.public_nat_b.id

  tags = {
    Name = "nat_gw_b-${var.env}"
  }
}
```

I remind you that the public subnets use a default route to the Internet
Gateway whereas the private subnet use a default route to the Nat Gateway:

```
resource "aws_route_table_association" "public_lb_a" {
  subnet_id      = aws_subnet.public_lb_a.id
  route_table_id = aws_route_table.route.id
}

resource "aws_route_table_association" "public_lb_b" {
  subnet_id      = aws_subnet.public_lb_b.id
  route_table_id = aws_route_table.route.id
}

resource "aws_route_table_association" "public_nat_a" {
  subnet_id      = aws_subnet.public_nat_a.id
  route_table_id = aws_route_table.route.id
}

resource "aws_route_table_association" "public_nat_b" {
  subnet_id      = aws_subnet.public_nat_b.id
  route_table_id = aws_route_table.route.id
}

resource "aws_route_table_association" "public_bastion_a" {
  subnet_id      = aws_subnet.public_bastion_a.id
  route_table_id = aws_route_table.route.id
}

resource "aws_route_table_association" "public_bastion_b" {
  subnet_id      = aws_subnet.public_bastion_b.id
  route_table_id = aws_route_table.route.id
}

resource "aws_route_table_association" "private_web_a" {
  subnet_id      = aws_subnet.private_web_a.id
  route_table_id = aws_route_table.route_nat_a.id
}

resource "aws_route_table_association" "private_web_b" {
  subnet_id      = aws_subnet.private_web_b.id
  route_table_id = aws_route_table.route_nat_b.id
}
```

Except the Nat Gateway, only the bastion requires an Elastic IP, the web
servers don't need it anymore because a public IP will automatically assign to
the Load Balancer:

```
resource "aws_eip" "nat_a" {
  vpc = true

  tags = {
    Name = "eip_nat_a-${var.env}"
  }
}

resource "aws_eip" "nat_b" {
  vpc = true

  tags = {
    Name = "eip_nat_b-${var.env}"
  }
}

resource "aws_eip" "bastion" {
  vpc = true

  tags = {
    Name = "eip_bastion-${var.env}"
  }
}
```

## Creating the Load Balancer

#### modules/base/alb.tf

We create our Application Load Balancer assigned in 2 public subnets:

```
resource "aws_lb" "web" {
  name               = "alb-web-${var.env}"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb_web.id]
  subnets            = [aws_subnet.public_lb_a.id, aws_subnet.public_lb_b.id]
}
```

We state the behavior of our Load Balancer, it forwards the requests to port
8000 of the web servers and check the health of our service by using the page
located at /cgi-bin/ping.py (you will see later how this script is created),
and the Load Balancer receives the requests in port 80:

```
resource "aws_lb_target_group" "web" {
  port     = 8000
  protocol = "HTTP"
  vpc_id   = aws_vpc.my_vpc.id

  health_check {
    healthy_threshold   = 2
    unhealthy_threshold = 2
    timeout             = 3
    interval            = 30
    path                = "/cgi-bin/ping.py"
  }
}

resource "aws_lb_listener" "web" {
  load_balancer_arn = aws_lb.web.arn
  port              = 80
  protocol          = "HTTP"

  default_action {
    target_group_arn = aws_lb_target_group.web.arn
    type             = "forward"
  }
}
```

## Configuring the Web Servers

#### modules/webserver/main.tf

We create an autoscaling group associated with our Load Balancer to ensure that
we have 2 servers up and running, if a server is failing for any reasons, it
will stop receiving the requests, afterwards it will be replaced by a new one:

```
resource "aws_autoscaling_group" "web" {
  name                 = "asg_web-${var.env}"
  launch_configuration = aws_launch_configuration.web.id
  vpc_zone_identifier  = [data.terraform_remote_state.base.outputs.subnet_private_web_a_id, data.terraform_remote_state.base.outputs.subnet_private_web_b_id]
  target_group_arns    = [data.terraform_remote_state.base.outputs.alb_target_group_web_arn]
  health_check_type    = "ELB"

  min_size             = 2
  max_size             = 2

  tag {
    key                 = "Name"
    value               = "webserver-${var.env}"
    propagate_at_launch = true
  }
}
```

#### modules/webserver/user-data.sh

I added a page located at /cgi-bin/ping.py for checking the health of the web
servers, in addition I display the instance Id:

```
#!/bin/bash

set -x

exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
sudo yum -y update
sudo yum -y upgrade
sudo yum -y install python38
sudo pip-3.8 install redis
sudo useradd www -s /sbin/nologin
mkdir -p /var/lib/www/cgi-bin
INSTANCE_ID="$(curl -s http://169.254.169.254/latest/meta-data/instance-id)"

cat << EOF > /var/lib/www/cgi-bin/hello.py
#!/usr/bin/env python3

import redis

r = redis.Redis(
                host='${database_host}',
                port=6379)
r.set('count', 0)
count = r.incr(1)

print("Content-type: text/html")
print("")
print("<html><body>")
print("<p>Hello World!<br />counter: " + str(count) + "<br />env: ${environment}</p>")
print("Id: $INSTANCE_ID")
print("</body></html>")
EOF

cat << EOF > /var/lib/www/cgi-bin/ping.py
#!/usr/bin/env python

import redis

r = redis.Redis(
                host='${database_host}',
                port=6379)
r.set('count', 0)
count = r.incr(1)

print("Content-type: text/html")
print("")
print("<html><body>")
print("<p>Hello World!<br />counter: " + str(count) + "<br />env: ${environment}</p>")
print("Id: $INSTANCE_ID")
print("</body></html>")
EOF

cat << EOF > /var/lib/www/cgi-bin/ping.py
#!/usr/bin/env python3

print("Content-type: text/html")
print("")
print("<html><body>")
print("<p>ok</p>")
print("</body></html>")
EOF

chmod 755 /var/lib/www/cgi-bin/hello.py
chmod 755 /var/lib/www/cgi-bin/ping.py
cd /var/lib/www
sudo -u www python3 -m http.server 8000 --cgi
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

When your infrastructure is built, get the DNS name of your Load Balancer by
performing the following command:

    $ aws elbv2 describe-load-balancers --names alb-web-dev \
        --query 'LoadBalancers[*].DNSName' \
        --output text

Get the ARN of your Load Balancer:

    $ aws elbv2 describe-load-balancers --names alb-web-dev \
        --query 'LoadBalancers[*].LoadBalancerArn' \
        --output text

Get the ARN of your Target Groups by providing the Load Balancer ARN:

    $ aws elbv2 describe-target-groups \
        --load-balancer-arn arn:aws:elasticloadbalancing:eu-west-3:xxxxxxxxxxxx:loadbalancer/app/alb-web-dev/xxxxxxxxxxxxxxxx \
        --query 'TargetGroups[*].TargetGroupArn' \
        --output text

Perform the following command by providing the Target Group ARN until you have
2 healthy instances:

    $ aws elbv2 describe-target-health \
        --target-group-arn arn:aws:elasticloadbalancing:eu-west-3:xxxxxxxxxxxx:targetgroup/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

Then issue the following command several times for increasing the counter:

    $ curl http://ARN_load_balancer:8000/cgi-bin/hello.py

It should return the count of requests you have performed.

## Testing the High Availability

Chose one of the 2 running instances and connect to it, then kill the web
service process:

    $ ssh -J ec2-user@IP_public_bastion ec2-user@IP_private_instance
    $ sudo su -
    # pkill python3

Wait for a while, then the Load Balancer will deregister the unhealthy instance,
you now have only one healthy instance:

    $ aws elbv2 describe-target-health \
        --target-group-arn arn:aws:elasticloadbalancing:eu-west-3:xxxxxxxxxxxx:targetgroup/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

If you make some requests to the service, you will notice it is still up and
running because the Load Balancer have stopped to forward to the failed server,
only the healthy server is used:

    $ curl http://ARN_load_balancer:8000/cgi-bin/hello.py

Wait for a while, then you will have 2 healthy instances because a new one has
replaced the failed instance.

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

