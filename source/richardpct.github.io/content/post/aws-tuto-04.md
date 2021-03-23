---
title: "AWS with Terraform tutorial 04"
date: 2021-03-16T20:24:25Z
draft: true
---

## Purpose

I will show you how to build a scenario more complex than the previous one
using a public and a private subnet in the same VPC. The scenario takes up this
tutorial
[Scenario 2: VPC with Public and Private Subnets (NAT)](https://docs.aws.amazon.com/vpc/latest/userguide/VPC_Scenario2.html?shortFooter=true),
here are the components you will build:

* Creating a VPC with a public and a private subnet
* Creating a webserver in the public subnet
* Creating a database server using Redis which stores the count of requests,
only the webserver is allowed to connect to the database server using firewall rules called "Security Group"
* Creating a NAT gateway with an Elastic IP (EIP) located in the public subnet
so that the database which is in the private subnet can reach Internet

Notice: For this exercise I do not use "Elastic Cache" service provided by AWS,
instead I use an EC2 instance where I install Redis.

The following figure depicts the infrastructure you will build:

<img src="https://raw.githubusercontent.com/richardpct/images/master/aws-tuto-04/image01.png">

A public subnet is an area where it can reach Internet and Internet can reach
it, whereas a private subnet is an area where it can reach Internet via a NAT
gateway but Internet can't reach it.<br />

The Terraform code can be found [here](https://github.com/richardpct/aws-terraform-tuto04).

## Network configuration

I will show you how to configure the network stack including the building of
the subnets and the firewall rules, I only show the relevant excerpt

#### modules/network/main.tf

Create the public subnet and their components (I remind you a public subnet is
a subnet which its default route is the Internet Gateway):

```
# Create the Internet Gateway
resource "aws_internet_gateway" "my_igw" {
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "my_igw-${var.env}"
  }
}

# Create the public subnet
resource "aws_subnet" "public" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = var.subnet_public

  tags = {
    Name = "subnet_public-${var.env}"
  }
}

# Create a custom route table
resource "aws_route_table" "route" {
  vpc_id = aws_vpc.my_vpc.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.my_igw.id
  }

  tags = {
    Name = "custom_route-${var.env}"
  }
}

# Associate the subnet with the default route table
resource "aws_route_table_association" "public" {
  subnet_id      = aws_subnet.public.id
  route_table_id = aws_route_table.route.id
}

# Create an Elastic IP for the Nat Gateway
resource "aws_eip" "nat" {
  vpc = true

  tags = {
    Name = "eip_nat-${var.env}"
  }
}

# Create a Nat Gateway in the public subnet
resource "aws_nat_gateway" "gw" {
  allocation_id = aws_eip.nat.id
  subnet_id     = aws_subnet.public.id

  tags = {
    Name = "nat_gw-${var.env}"
  }
}
```

Create the private subnet (I remind you a private subnet is a subnet which its
default route is the Nat Gateway):

```
# Create a private subnet
resource "aws_subnet" "private" {
  vpc_id     = aws_vpc.my_vpc.id
  cidr_block = var.subnet_private

  tags = {
    Name = "subnet_private-${var.env}"
  }
}

# Use the default route table for the private subnet
resource "aws_default_route_table" "route" {
  default_route_table_id = aws_vpc.my_vpc.default_route_table_id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_nat_gateway.gw.id
  }

  tags = {
    Name = "default_route-${var.env}"
  }
}

# Associate the private subnet with the default route table
resource "aws_route_table_association" "private" {
  subnet_id      = aws_subnet.private.id
  route_table_id = aws_default_route_table.route.id
}
```

#### modules/network/sg.tf

Create a security group for the database stack and for the webserver stack:

```
resource "aws_security_group" "database" {
  name   = "sg_database-${var.env}"
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "database_sg-${var.env}"
  }
}

resource "aws_security_group" "webserver" {
  name   = "sg_webserver-${var.env}"
  vpc_id = aws_vpc.my_vpc.id

  tags = {
    Name = "webserver_sg-${var.env}"
  }
}
```

Only the webserver is allowed to make requests to the database on the Redis
port:

```
resource "aws_security_group_rule" "database_inbound_redis" {
  type                     = "ingress"
  from_port                = local.redis_port
  to_port                  = local.redis_port
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.webserver.id
  security_group_id        = aws_security_group.database.id
}
```

Set some rules so that the database can update its system and softwares on
Internet:

```
resource "aws_security_group_rule" "database_outbound_http" {
  type              = "egress"
  from_port         = local.http_port
  to_port           = local.http_port
  protocol          = "tcp"
  cidr_blocks       = local.anywhere
  security_group_id = aws_security_group.database.id
}

resource "aws_security_group_rule" "database_outbound_https" {
  type              = "egress"
  from_port         = local.https_port
  to_port           = local.https_port
  protocol          = "tcp"
  cidr_blocks       = local.anywhere
  security_group_id = aws_security_group.database.id
}
```

Only your own IP is allowed to connect to the webserver via SSH:

```
resource "aws_security_group_rule" "webserver_inbound_ssh" {
  type              = "ingress"
  from_port         = local.ssh_port
  to_port           = local.ssh_port
  protocol          = "tcp"
  cidr_blocks       = [var.cidr_allowed_ssh]
  security_group_id = aws_security_group.webserver.id
}
```

Anyone can make HTTP requests to the webserver:

```
resource "aws_security_group_rule" "webserver_inbound_http" {
  type              = "ingress"
  from_port         = local.webserver_port
  to_port           = local.webserver_port
  protocol          = "tcp"
  cidr_blocks       = local.anywhere
  security_group_id = aws_security_group.webserver.id
}
```

Set some rules so that the webserver can update its system and softwares on
Internet:

```
resource "aws_security_group_rule" "webserver_outbound_http" {
  type              = "egress"
  from_port         = local.http_port
  to_port           = local.http_port
  protocol          = "tcp"
  cidr_blocks       = local.anywhere
  security_group_id = aws_security_group.webserver.id
}

resource "aws_security_group_rule" "webserver_outbound_https" {
  type              = "egress"
  from_port         = local.https_port
  to_port           = local.https_port
  protocol          = "tcp"
  cidr_blocks       = local.anywhere
  security_group_id = aws_security_group.webserver.id
}
```

The webserver is allowed to connect to the database:

```
resource "aws_security_group_rule" "webserver_outbound_redis" {
  type                     = "egress"
  from_port                = local.redis_port
  to_port                  = local.redis_port
  protocol                 = "tcp"
  source_security_group_id = aws_security_group.database.id
  security_group_id        = aws_security_group.webserver.id
}
```

## Redis configuration

Only the user-data script deserves to be showed here:

#### modules/database/user-data.sh

```
#!/bin/bash

exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
sudo apt-get update
sudo apt-get -y upgrade
sudo apt-get -y install redis
sudo sed -i -e 's/^\(bind 127.0.0.1 ::1\)/#\1/' /etc/redis/redis.conf
sudo sed -i -e 's/# \(requirepass\) foobared/\1 ${database_pass}/' /etc/redis/redis.conf
sudo systemctl restart redis
```

I allow all network interfaces to be listened on Redis port then I set the
password.

## Webserver configuration

Likewise only the user-data script deserves to be showed here:

#### modules/webserver/user-data.sh

```
#!/bin/bash

exec > >(tee /var/log/user-data.log|logger -t user-data -s 2>/dev/console) 2>&1
sudo yum -y update
sudo yum -y upgrade
sudo yum -y install python38
sudo pip-3.8 install redis
sudo useradd www -s /sbin/nologin
mkdir -p /var/lib/www/cgi-bin

cat << EOF > /var/lib/www/cgi-bin/hello.py
#!/usr/bin/env python3

import redis

r = redis.Redis(
                host='${database_host}',
                port=6379,
                password='${database_pass}')
r.set('count', 0)
count = r.incr(1)

print("Content-type: text/html")
print("")
print("<html><body>")
print("<p>Hello World!<br />counter: " + str(count) + "<br />env: ${environment}</p>")
print("</body></html>")
EOF

chmod 755 /var/lib/www/cgi-bin/hello.py
cd /var/lib/www
sudo -u www python3 -m http.server 8000 --cgi
```

I use the module http.server of Python 3 for spinning up a web server,
/var/lib/www/cgi-bin/hello.py is the cgi script which is executed.<br />
I use the redis module for connecting to the Redis server, and whenever the cgi
script is called, the count field will be incremented by 1.<br />
The webserver serves the HTTP requests on port 8000.

## Deploying the infrastructure

Export some environment variables:

    $ export TF_VAR_region="eu-west-3"
    $ export TF_VAR_bucket="yourbucket-terraform-state"
    $ export TF_VAR_dev_network_key="terraform/dev/network/terraform.tfstate"
    $ export TF_VAR_dev_database_key="terraform/dev/database/terraform.tfstate"
    $ export TF_VAR_dev_webserver_key="terraform/dev/webserver/terraform.tfstate"
    $ export TF_VAR_ssh_public_key="ssh-rsa XXX..."
    $ export TF_VAR_dev_database_pass="pass_redis"
    $ export TF_VAR_my_ip_address=$(curl -s 'https://duckduckgo.com/?q=ip&t=h_&ia=answer' \
    | sed -e 's/.*Your IP address is \([0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\.[0-9]\{1,3\}\) in.*/\1/')

building:

    $ cd environments/dev
    $ cd 01-network
    $ terraform init \
        -backend-config="bucket=${TF_VAR_bucket}" \
        -backend-config="key=${TF_VAR_dev_network_key}" \
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

After finishing your test, destroy your infrastructure:

    $ cd environments/dev
    $ cd 03-webserver
    $ terraform destroy
    $ cd 02-database
    $ terraform destroy
    $ cd ../01-network
    $ terraform destroy

## Summary

We have studied how to isolate some servers such as a database using a private
subnet.<br />
In the next tutorial we will add a bastion in our infrastructure.
