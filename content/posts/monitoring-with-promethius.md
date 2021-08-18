+++
title = "Monitoring With Prometheus"
date = "2021-08-17T12:34:59+01:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["", ""]
keywords = ["", ""]
description = "Monitoring Cardano with Prometheus and Grafana"
showFullContent = false
+++

# Requirements

TODO do a diagram

- Install Grafana


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

- Installs binary to /usr/sbin/grafana-server
- Copies init.d script to /etc/init.d/grafana-server
- Installs default file (environment vars) to /etc/sysconfig/grafana-server
- Copies configuration file to /etc/grafana/grafana.ini
- Installs systemd service (if systemd is available) name grafana-server.service
- The default configuration uses a log file at /var/log/grafana/grafana.log
- The default configuration specifies an sqlite3 database at /var/lib/grafana/grafana.db

### Install Prometheus

Install prometheus in Docker for ease of use. Some of the port mappings are for feeding in downstream data which we will use later.

````shell
docker run --detach --network prod-cardano-bridge\
    --name=prometheus \
    --restart=always \
    -p 9090:9090 \
    -p 9200:9100 \
    -p 9300:9323 \
    -v /home/ec2-user/prometheus/docker-config:/etc/prometheus \
    prom/prometheus
````


Tail the prometheus logs to check it's working correctly.
````shell
docker logs -f prometheus
````

### Install Prometheus Node Exporter

Node exporter can't be run in docker as this is the service that will push the metrics from the host linux server into Prometheus and eventually into Grafana.

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

Step 3: Add the following service file content to the service file and save it.
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

Step 4: Reload the system daemon and star the node exporter service.
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