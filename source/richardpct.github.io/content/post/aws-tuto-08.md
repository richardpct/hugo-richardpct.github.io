---
title: "AWS with Terraform tutorial 08"
date: 2021-04-23T18:55:17Z
draft: false
---

## Purpose

This tutorial takes up the previous one
[aws-terraform-tuto07](https://richardpct.github.io/post/2021/04/17/aws-with-terraform-tutorial-07/)
by adding autoscaling policy in order to scale our web server according to a
metric that you chose, for our example I chose to add a web server if the CPU
Load is higher than a threshold that I have defined.

The following figure depicts the infrastructure you will build:

<img src="https://raw.githubusercontent.com/richardpct/images/master/aws-tuto-08/image01.png">

The source code can be found [here](https://github.com/richardpct/aws-terraform-tuto08).

## Configuring the autoscaling policy

#### modules/webserver/main.tf

I only show the relevant excerpt on how to configure the autoscaling policy:

```
resource "aws_autoscaling_group" "web" {
  name                 = "asg_web-${var.env}"
  launch_configuration = aws_launch_configuration.web.id
  vpc_zone_identifier  = [data.terraform_remote_state.base.outputs.subnet_private_web_a_id, data.terraform_remote_state.base.outputs.subnet_private_web_b_id]
  target_group_arns    = [data.terraform_remote_state.base.outputs.alb_target_group_web_arn]
  health_check_type    = "ELB"

  min_size             = 2
  max_size             = 3

  tag {
    key                 = "Name"
    value               = "webserver-${var.env}"
    propagate_at_launch = true
  }
}

resource "aws_autoscaling_policy" "web" {
  name                   = "autoscaling_policy_web-${var.env}"
  policy_type            = "TargetTrackingScaling"
  autoscaling_group_name = aws_autoscaling_group.web.name

  target_tracking_configuration {
    predefined_metric_specification {
      predefined_metric_type = "ASGAverageCPUUtilization"
    }

    target_value = 40.0
  }
}
```

`min_size = 2` intends to have at least 2 web servers up and running, and
`max_size = 3` intends to have at maximum 3 web servers in any case.<br />
`predefined_metric_type = "ASGAverageCPUUtilization"` intends to use the
CPU Load Metric for scaling the web servers.<br />
`target_value = 40.0` intends to scale up when the average CPU Load of all our
web server is higher than 40% and when the average is lower than 40% the
autoscaling will scale down the service.

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

## Testing the automation of autoscaling

Chose one of the 2 running instances and connect to it, then install a package
stress in order to burn the CPU:

    $ ssh -J ec2-user@IP_public_bastion ec2-user@IP_private_instance
    $ sudo su -
    # yum install stress
    # stress --cpu 1

Wait for a while, then a third web server will be created and the Load Balancer
will register it, you now have three healthy instances:

    $ aws elbv2 describe-target-health \
        --target-group-arn arn:aws:elasticloadbalancing:eu-west-3:xxxxxxxxxxxx:targetgroup/xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx

If you make some requests to the service, you will notice that the three
servers will serve your requests in turn:

    $ curl http://ARN_load_balancer/cgi-bin/hello.py

For testing the scale down when there is no longer high load, just stop the
stress process by pressing `CTRL-C` in your terminal, then a web server will be
terminated and you have now 2 web servers up and running.

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

