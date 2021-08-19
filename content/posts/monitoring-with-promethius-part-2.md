+++
title = "Monitoring With Prometheus - Part 2"
date = "2021-08-17T12:34:59+01:00"
author = "Shawn V"
authorTwitter = "" #do not include @
cover = "img/p1.jpg"
tags = ["Grafana", "Prometheus"]
keywords = ["Grafana", "Prometheus"]
description = "Monitoring Cardano with Prometheus and Grafana"
showFullContent = false
+++

## Overview

Now that we have successfully installed Prometheus and Grafana we can start using it.

### Look inside

We have two dashboards:

- Prometheus  Port:9090
- Grafana     Port:3000

These two web sites are running on your AWS EC2 server so to access them via a browser you need to do two things.

- (Optional) Allocate an Elastic IP address to your server. [Elastic IP Docs](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/elastic-ip-addresses-eip.html)
- On your EC2 instance there be a `Security Group` where you can manage what Ports are open to the outside world. In here you need to open the two ports `9090` & `3000` to your IP address.

I created a Security Group `Cardano Monitoring` with the port mappings below. Remember to add the Security Group to your EC2 instance.

![Cardano](/img/aws-ports.png)

You should now have access to both sites on the URL below where `x.x.x.x.x` is your Elastic IP address:

http://x.x.x.x.x:9090/targets   (Prometheus)

http://x.x.x.x.x:3000           (Grafana)

### Grafana

- Set up your username and password
- Set up a Prometheus Data Source: Settings > DataSource > Prometheus > `http://localhost:9090` (This is an important step you can't miss it!)
- Add a dashboard.

To add a dashboard you can do it manually but you can also import one by ID
https://grafana.com/grafana/dashboards/14731

Add Dashboard > Import > 14731

This will give you an amazing dashboard for your linux host. A tiny sample below.

![Cardano](/img/g1.png)

### Cardano nodes

Remember when we configured Cardano we edited the file `mainnet-config.json` to include monitoring.
Now we should be able to see that target in http://x.x.x.x.x:9090/targets Prometheus. Its name should be `relay-1`

Steps to adding a new dashboard:

- Create a new dashboard in Grafana
- Using the Prometheus DataSource create a chart.

![Cardano](/img/g2.png)

Now you are on your own.

You can take a look at the JSON from our sample dashboard [github](https://github.com/io-alt-f/grafana-dashboard/blob/main/cardano-dashboard.json) but if you try and use it you will firstly need to create a variable in Grafana for it to work.

![Cardano](/img/g3.png)

Our sample Cardano dashboard.

![Cardano](/img/g4.png)
