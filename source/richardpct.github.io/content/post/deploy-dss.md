---
title: "Deploy DSS on GCP"
date: 2021-09-04T08:48:49Z
draft: true
---

## Purpose

I will show you how to deploy DSS on GCP using Terraform and monitored by
Prometheus, the following tasks will be built:

* Creating 2 VM on a common network
* Installing Prometheus on one of the machine
* Monitor the basic system metrics of the second machine with it
* Install DSS on the second machine
* Add a probe to monitor whether DSS service is alive or not
* Configure a basic alert if DSS service stops responding

## Requirements

### Create a SSH key

    $ ssh-keygen -t rsa -b 4096

By default the pair key is created in ~/.ssh/id_rsa and ~/.ssh/id_rsa.pub, we
will use the public key for GCP and acces to it by using SSH.

### Create a service account

Open to your GCP console then go to `IAM & Admin -> Service Accounts` with
project editor role, create a service account, then create your key as json
type and save it on your own computer.

### Install the latest Terraform

For this Exercise I use the latest Terraform version, that is 1.0.6

## Configure the Prometheus server

The build of the Prometheus server is described in the prometheus.tf file.<br />
The script that is launched at the bootstrap is startup_prometheus.sh, it
defines all the steps for configuring the Prometheus server, these steps are
as follow:

* Install Prometheus
* Install Blackbox Exporter in order to check the health of DSS using a http
request to http://public_ip-DSS:11000/dip/api/get-configuration
* Acces to the Node exporter of the DSS server in order to get some basic
Metrics
* Add an alert if the DSS service is down

The prometheus configuration (/opt/prometheus/prometheus-2.29.2.linux-amd64/prometheus.yml)
looks like as follow:

```
# my global config
global:
  scrape_interval: 15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

# Alertmanager configuration
alerting:
  alertmanagers:
    - static_configs:
        - targets:
          # - alertmanager:9093

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  - "alert.rules.yml"
  # - "second_rules.yml"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: "prometheus"

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ["localhost:9090"]
  - job_name: "dss"
    static_configs:
      # Access to the Node Exporter of DSS server
      - targets: ["10.132.0.14:9100"]
  # Blackbox Exporter
  - job_name: 'blackbox'
    metrics_path: /probe
    params:
      module: [http_2xx]
    static_configs:
      - targets:
        # Check the health of the DSS service
        - http://10.132.0.14:11000/dip/api/get-configuration
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        # Expose the Probe
        replacement: 127.0.0.1:9115
```

Notice that the DSS IP is computed by Terraform.

## Configure the DSS server

The build of the Prometheus server is described in the dss.tf file.<br />
The script that is launched at the bootstrap is startup_dss.sh, it
defines all the steps for configuring the DSS server, these steps are as
follow:

* Install the Node Exporter in order to expose some basics metrics
* Install DSS

## Build the infrastructure

    $ git clone https://github.com/richardpct/deploy-dss-gcp.git
    $ cd deploy-dss-gcp

Copy your credentials in this directory with the name credentials.json

    $ terraform init
    $ terraform apply

The last command returns the public IP of our DSS and Prometheus servers, you
can connect to them by using SSH and check the installation in
/var/log/startup.log file.

## Check your infrastructure

### DSS
Open your browser at http://public_ip_DSS:11000

### Prometheus
Open your browser at http://public_ip_prometheus:9090

### Probe that check the DSS service
open your browser at http://public_ip_prometheus:9115

#### Check the alert of the DSS service
open your browser at http://public_ip_prometheus:9090/alerts

## Destroy the infrastructure

    $ terraform destroy

