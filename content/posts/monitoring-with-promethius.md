+++
title = "Monitoring With Prometheus - Part 1"
date = "2021-08-17T12:34:59+01:00"
author = "Shawn V"
authorTwitter = "" #do not include @
cover = "img/p2.jpg"
tags = ["Grafana", "Prometheus"]
keywords = ["Prometheus", "Grafana"]
description = "Installing and configuring Prometheus and Grafana"
showFullContent = false
+++

## Overview

Monitoring your Cardano relay and block producing node `Stake pool` is vitally important if you want to have a successful stake pool.

What are the important things we want to monitor?

- The underlying hardware.
- The network health & bandwidth.
- The cardano nodes themselves.

### Underlying hardware

There is no point monitoring the Cardano nodes unless you know what is going on with your host system. If your host system runs out of Disk space, RAM or has bad network Cardano will fail.
We will want to address these issues before they impact our Cardano nodes.

### Network bandwidth

Network monitoring can get really complex with analyzing traffic patterns, detecting and diagnosing bandwidth hogs, but for now we want to just make sure that we have reliable consistent bandwidth. To do this we need to monitor and look at historical patterns to see if our bandwidth meets our needs.

### Cardano nodes

When your node is up and running you should monitor its behavior. By doing this you can see how everything is running, if your system is performing as it should to keep the node running, and if there is anything that you can do to fine tune system performance.
You can use EKG and live view or you can allow monitoring of the nodes using Prometheus.

    In our case we are going to monitor our Linux server, Cardano relay/producer nodes using 
    Prometheus and Grafana. Prometheus gathers and stores the metrics
    Grafana provides a way to visualize the data with additional functionality like alerting.

![Cardano](/img/monitoring.png)

## 1. Install Grafana

You can find install instructions [here](https://grafana.com/docs/grafana/latest/installation/rpm/)

Create a repo file

```shell
vim /etc/yum.repos.d/grafana.repo
```

Add the following contents to file:

```shell
[grafana]
name=grafana
baseurl=https://packages.grafana.com/oss/rpm
repo_gpgcheck=1
enabled=1
gpgcheck=1
gpgkey=https://packages.grafana.com/gpg.key
sslverify=1
sslcacert=/etc/pki/tls/certs/ca-bundle.crt
```

Install Grafana

```shell
vim /etc/yum.repos.d/grafana.repo
```

```shell
sudo yum install grafana
```

Start the server with init.d
To start the service and verify that the service has started:

```shell
sudo service grafana-server start
```

```shell
sudo service grafana-server status
```

Configure the Grafana server to start on reboot:

```shell
sudo /sbin/chkconfig --add grafana-server
```

### Install details

- Installs binary to `/usr/sbin/grafana-server`
- Copies init.d script to `/etc/init.d/grafana-server`
- Installs default file (environment vars) to `/etc/sysconfig/grafana-server`
- Copies configuration file to `/etc/grafana/grafana.ini`
- Installs systemd service (if systemd is available) name `grafana-server.service`
- The default configuration uses a log file at `/var/log/grafana/grafana.log`
- The default configuration specifies an sqlite3 database at `/var/lib/grafana/grafana.db`

### Install Prometheus

For our purposes Prometheus server within Docker and the Prometheus Node exporter.

Prometheus Node: Will be installed as a service on our linux host machine.
Prometheus Server: will be run in Docker

#### Install `Node Exporter`

Step 1. Get the latest version of `Prometheus Node Exporter` from [here](https://prometheus.io/download/)
````shell
cd /tmp
curl -LO https://github.com/prometheus/node_exporter/releases/download/v1.2.2/node_exporter-1.2.2.linux-amd64.tar.gz
````

Step 2: Unpack the tarball
````shell
tar -xvf node_exporter-1.2.2.linux-amd64.tar.gz
````

Step 3: Move the node export binary to /usr/local/bin
````shell
sudo mv node_exporter-1.2.2.linux-amd64/node_exporter /usr/local/bin/
````

It's important to set Node Exporter up as a service so that it runs in the background and restarts on boot.

Step 1: Create a node_exporter user to run the node exporter service.
````shell
sudo useradd -rs /bin/false node_exporter
````

Step 2: Create a node_exporter service file under systemd.
````shell
sudo vi /etc/systemd/system/node_exporter.service
````

Step 3: Add the following service file content to the `node_exporter.service` file and save it.
````shell
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=node_exporter
Group=node_exporter
Type=simple
ExecStart=/usr/local/bin/node_exporter

[Install]
WantedBy=multi-user.target
````

Step 4: Reload the system daemon and start the node exporter service.
````shell
sudo systemctl daemon-reload
sudo systemctl start node_exporter
````

Step 5: check the node exporter status to make sure it is running in the active state.
````shell
sudo systemctl status node_exporter
````

Step 6: Enable the node exporter service to the system startup.
````shell
sudo systemctl enable node_exporter
````

#### Install `Prometheus Server`

We are going to run in Docker.

We need to create the configuration file `prometheus.yml`

```shell
mkdir ~/prometheus; mkdir docker-config
touch ~/prometheus/docker-config/prometheus.yml
```

To create the configuration file you will need to find the IP addresses of your Cardano nodes that are allocated in Docker. Fortunately we created a network bridge so you can use the command below.

```shell
docker network inspect prod-cardano-bridge
```

Add this to `prometheus.yml`
````shell
global:
   scrape_interval:     15s
   external_labels:
     monitor: 'codelab-monitor'
scrape_configs:
   - job_name: 'relay-1' # To scrape data from the cardano node
     scrape_interval: 5s
     static_configs:
       - targets: ['x.x.x.x:12798']
   - job_name: 'relay-2' # To scrape data from the cardano node
     scrape_interval: 5s
     metrics_path: /metrics
     static_configs:
       - targets: ['x.x.x.x:12798']
   - job_name: 'node' # To scrape data from a node exporter to monitor your linux host metrics.
     scrape_interval: 5s
     static_configs:
       - targets: ['x.x.x.x:9100']
````

Start Prometheus in Docker using the `prometheus.yml` configuration file. We have mapped some ports to allow access.

````shell
docker run --detach --network prod-cardano-bridge\
    --name=prometheus \
    --restart=always \
    -p 9090:9090 \
    -p 9200:9100 \
    -p 9300:9323 \
    -v ~/prometheus/docker-config:/etc/prometheus \
    prom/prometheus
````

Check the logs.
````shell
docker logs -f prometheus
````
