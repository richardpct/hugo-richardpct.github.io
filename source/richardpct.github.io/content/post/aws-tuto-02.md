---
title: "AWS with Terraform tutorial 02"
date: 2021-03-04T19:42:41Z
draft: False
---

## Purpose

This tutorial takes up the previous one in improving our code by using the
modules. A module in Terraform acts like a funtion in a programming language,
and like a function we can provide some parameters.<br />
It is a good practice to work with, and you know the famous adage in computer 
science? "Don't repeat yourself!".<br />

The source code is available on my [Github repository](https://github.com/richardpct/aws-terraform-tuto02).

## Create modules in Terraform

Here is the new layout of our Terraform files located in our file system:

```
.
├── 01-network
│   ├── backends.tf
│   ├── main.tf
│   ├── outputs.tf
│   └── versions.tf
├── 02-webserver
│   ├── backends.tf
│   ├── main.tf
│   ├── outputs.tf
│   ├── vars.tf
│   └── versions.tf
└── modules
    ├── network
    │   ├── backends.tf
    │   ├── main.tf
    │   ├── outputs.tf
    │   └── vars.tf
    └── webserver
        ├── backends.tf
        ├── main.tf
        ├── outputs.tf
        └── vars.tf
```

The modules are located in the modules directory, they are written as a regular
Terraform code.<br />
`01-network` and `02-webserver` are the modules caller, `01-network` calling
the network module with some parameters and `02-webserver` calling the
webserver module with some parameters.<br />

Let's see how to call a module with some parameters:

#### 01-network/main.tf

```
module "network" {
  source = "../modules/network"

  region         = "eu-west-3"
  vpc_cidr_block = "10.0.0.0/16"
  subnet_public  = "10.0.0.0/24"
}
```

#### 02-webserver/main.tf
```
module "webserver" {
  source = "../modules/webserver"

  region                      = "eu-west-3"
  network_remote_state_bucket = var.bucket
  network_remote_state_key    = var.network_key
  instance_type               = "t2.micro"
  image_id                    = "ami-0ebc281c20e89ba4b" // Amazon Linux 2018
  ssh_public_key              = var.ssh_public_key
}
```

You have just set the `source` variable with the path of your module, then set
the values of some variables.

## Summary

The modules are easy to write and use, so use it for avoiding to duplicate your
code.<br />
In the next tutorial I will show you how to split your code for deploying your
infrastructure in multiple environment: (dev, staging, prod ...) using the
modules.
