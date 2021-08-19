+++
title = "How to set up Cardano relay nodes"
date = "2021-08-16T11:36:15+01:00"
author = ""
authorTwitter = "" #do not include @
cover = "img/cardano-relay.png"
tags = ["cardano", "relay-node", "AWS"]
keywords = ["", ""]
description = "How to set up and run a Cardano relay node"
showFullContent = false
+++

## Intro

First we need to know what is a Cardano Relay node?

    A relay node is essentially a Proxy between the Cardano core network nodes 
    and the internet. 
    If you were going to run commands to query your address for example you 
    would do this via your relay node.

Relay nodes do not have any keys, so they cannot produce blocks.

    I would also suggest starting on `testnet`. My example here is for a production `mainnet` node but just replace `mainnet` with `testnet`.

In this post we are going to cover:

- Hardware considerations
- How to set up a Cardano node on `mainnet` 
- How to run a Relay node on AWS within Docker
- Starting and stopping your Relay node
- Running commands on the Relay node
- Monitoring your Relay node

### Hardware considerations

The official recommendation is

    4 GB of RAM
    24 GB of hard disk space
    1 GB of bandwidth per hour

Officially it is not recommended to run on virtualized hardware like AWS but is this really the best advice?

#### Pros of AWS

- Very quick to get up and running.
- Safe. If you follow AWS recommendations it's a very safe.
- Cost effective. Many ways to reduce costs like dedicated instances.
- Excellent monitoring using Cloudwatch if you do not want to set up Prometheus with Grafana.
- Easy to upgrade. Add more memory, disk space and processing power easily.

#### Cons

- Can be more expensive than dedicated hardware.
- Unpredictable load and memory usage on underlying hardware. This is the biggest downside.

So as you can see I still think that AWS is worth using but it will require monitoring and adjustment once we have more data.

### How to set up a Cardano node on `mainnet`

I am not going to explain how to set up an AWS EC2 instance so if you do not know you will need to read AWS [docs](https://aws.amazon.com/ec2/getting-started/).

`Note`: I am not using AMD processors as some of the utilities we will use later do not have support for it. I choose 64-bit (x86)
with a memory optimized instance  `r5.large` and increased 24GB storage.

Form this point I am assuming you have SSH'd onto your running AWS EC2 instance and are ready to begin running commands.

#### 1. Install docker

Follow the instructions to [*install docker on AWS*](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html)

#### 1.2 Additional docker steps

After following the instructions above you need to exit your EC2 instance and re-login to it. Then enable Docker to auto-start after boot.

````shell
 sudo systemctl enable --now docker
````

Now `docker info` will work.

### 2. Create a `relay` node in docker

![Cardano](/img/cardano-relay.png)


#### 2.1 Make directory structure on your local machine that we are going to use as volumes in docker

I am going to install two relay nodes in one docker container. To just start with one then ignore all the `prodrl2` commands. 

````shell
mkdir /home/ec2-user/cardano;
mkdir /home/ec2-user/cardano/config;
mkdir /home/ec2-user/cardano/config/keys;
mkdir /home/ec2-user/cardano/data;
mkdir /home/ec2-user/cardano/data/prodrl1;
mkdir /home/ec2-user/cardano/data/prodrl2;
mkdir /home/ec2-user/cardano/output;
````

#### 2.1 Download config files

Download the latest config files. Don't rely on the links below to be correct always check [on Cardano](https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/index.html). Replace with testnet links if setting up on the test environment.

`mainnet`

````shell
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-config.json -P /home/ec2-user/cardano/config/
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-byron-genesis.json -P /home/ec2-user/cardano/config/;
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-shelley-genesis.json -P /home/ec2-user/cardano/config/;
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-topology.json  -P /home/ec2-user/cardano/config/
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-alonzo-genesis.json  -P /home/ec2-user/cardano/config/
````

#### 2.2 Create the topology file

Create a file called /home/ec2-user/cardano/config/mainnet-relay-topology.json
Add the JSON to this file and save

```json
{
  "Producers": [
    {
      "addr": "relays-new.cardano-mainnet.iohk.io",
      "port": 3001,
      "valency": 2
    }
  ]
}
```

#### 2.3 Start the node with basic configuration

```properties
docker run --detach \
    --name=prodrl1 \
    -p 3001:3001 \
    -e CARDANO_UPDATE_TOPOLOGY=true \
    -v node-data:/opt/cardano/data/prodrl1 \
    nessusio/cardano-node run
```

What does this command do?
Uses [docker run](https://docs.docker.com/engine/reference/run/) command

    --detach runs in the background
    -p 3001:3001  Maps the Docker containers port to the host port
    -e CARDANO_UPDATE_TOPOLOGY=true Sets an environment variable in the docker container
    -v node-data:/opt/cardano/data  Maps a volume /opt/cardano/data/prodrl1 on the container to a name node-data
    nessusio/cardano-node run  Uses the image nessusio/cardano-node  and calls `run` to start the container


See the docker process

```properties
docker ps
```

Stop the docker process

```properties
docker stop prodrl1
```

Remove the docker process

```properties
docker rm prodrl1
```

### 2.3 Create docker volumes

If you want to monitor your application using Prometheus and Grafana you will need to edit the file `mainnet-config.json` before we copy it.
Uncomment the section

```yaml
  hasPrometheus:
    - "127.0.0.1"
    - 12789
```


Create a volume where we will store the custom configuration and keys

````properties
docker run --name=tmp -v cardano-relay-config:/var/cardano/config centos

docker cp /home/ec2-user/cardano/config/mainnet-relay-topology.json tmp:/var/cardano/config/mainnet-relay-topology.json
docker cp /home/ec2-user/cardano/config/mainnet-byron-genesis.json tmp:/var/cardano/config/mainnet-byron-genesis.json
docker cp /home/ec2-user/cardano/config/mainnet-shelley-genesis.json tmp:/var/cardano/config/mainnet-shelley-genesis.json
docker cp /home/ec2-user/cardano/config/mainnet-alonzo-genesis.json tmp:/var/cardano/config/mainnet-alonzo-genesis.json
docker cp /home/ec2-user/cardano/config/mainnet-config.json tmp:/var/cardano/config/mainnet-config.json

docker rm -f tmp
````

````properties
docker run --name=tmp -v cardano-output:/var/cardano/output centos

docker cp /home/ec2-user/cardano/config/mainnet-relay-topology.json tmp:/var/cardano/config/mainnet-relay-topology.json
docker cp /home/ec2-user/cardano/config/mainnet-byron-genesis.json tmp:/var/cardano/config/mainnet-byron-genesis.json
docker cp /home/ec2-user/cardano/config/mainnet-shelley-genesis.json tmp:/var/cardano/config/mainnet-shelley-genesis.json
docker cp /home/ec2-user/cardano/config/mainnet-alonzo-genesis.json tmp:/var/cardano/config/mainnet-alonzo-genesis.json
docker cp /home/ec2-user/cardano/config/mainnet-config.json tmp:/var/cardano/config/mainnet-config.json

docker rm -f tmp
````

We just created a volume `cardano-relay-config` that links the host and the docker instance file systems.

Useful commands for inspecting volumes:

```shell
docker volume ls
```

Inspect a volume. `Don't skip this it's very useful to see how it maps the container to the host filesystem.`

```shell
docker volume inspect cardano-relay-config
```

#### 2.4 Create a network bridge in docker

As we are building a production system we need to create a network bridge as we do not want to rely on the default docker bridge. We intend to run multiple relay nodes in this environment.

Create the network bridge

```shell
docker network create --driver bridge prod-cardano-bridge
```

Inspect the network bridge

```shell
docker network inspect prod-cardano-bridge
```

#### 2.5 Start cardano relay node with the TOPOLOGY updater and the network bridge

The fully configured node will

- Auto update the topology
- Run using the network bridge *prod-cardano-bridge*
- Map a volume named *`ipc`* for when we use the *cardano-cli* commands
- Map the additional volumes `data`, `config` and `output`
- Auto restart our relay if docker to the host is restarted

```properties
docker run --detach --network prod-cardano-bridge \
    --name=prodrl1 \
    --restart=always \
    -p 3001:3001 \
    -p 12798:12798 \
    -e CARDANO_PORT=3001 \
    -e CARDANO_NETWORK=mainnet \
    -e CARDANO_CONFIG=/var/cardano/config/mainnet-config.json \
    -e CARDANO_UPDATE_TOPOLOGY=true \
    -v ipc:/opt/cardano/ipc \
    -v /home/ec2-user/cardano/data/prodrl1:/opt/cardano/data \
    -v cardano-relay-config:/var/cardano/config  \
    -v /home/ec2-user/cardano/output:/opt/cardano/output \
    nessusio/cardano-node run
```

`Optional` Example on how to start a second Relay node using the network bridge we created earlier.

```properties
docker run --detach --network prod-cardano-bridge \
    --name=prodrl2 \
    --restart=always \
    -p 3002:3002 \
    -e CARDANO_PORT=3002 \
    -e CARDANO_NETWORK=mainnet \
    -e CARDANO_UPDATE_TOPOLOGY=true \
    -v ipc:/opt/cardano/ipc \
    -v /home/ec2-user/cardano/data/prodrl2:/opt/cardano/data \
    -v cardano-relay-config:/var/cardano/config  \
    -v /home/ec2-user/cardano/output:/opt/cardano/output \
    nessusio/cardano-node run
```

#### 2.6 Useful maintenance commands

Tail the Cardano node logs
````shell
docker logs -f prodrl1
````

View the Docker stats like cpu, Memory usage etc
````shell
docker stats
````

View the cardano live view for the node
```shell
docker exec -it prodrl1 gLiveView
```

Access the Docker image in interactive mode using bash
```shell
docker exec -it prodrl1 bash
```

#### 2.7 Sit back and congratulate yourself

Your node needs to sync with the blockchain before it's ready.
This will most likely several hours. Use can use the `gLiveView` to see the status `syncing percentage`

## 3 Running commands on your node

`NOTE`: _If you run commands before syncing is complete you may get errors because of running commands on older era's where they are not supported_. Wait until it is close to 100% completed syncing.

`NOTE`: _--mainnet identifies the Cardano mainnet, for testnets use --testnet-magic 1097911063 instead._

Now we want to run `cardano-cli` commands but the only place Cardano is installed is on our Docker container, and not on our AWS EC2 instance.

So we have two options:

- Access the running docker container and run the commands within the container.
- Use an alias to run `cardano-cli` on the container with the output to a mapped volume. 

We will use the previously mapped *`ipc`* volume and mapping an *`output`* volume we can see the output of the commands created in the docker container on our local host.

#### Create the cardano-cli alias

```shell
alias cardano-cli="docker run -it --rm \
  -v /home/ec2-user/cardano/output:/var/cardano/output \
  -v ipc:/opt/cardano/ipc \
  nessusio/cardano-node cardano-cli"
```

Verify the command is working

```shell
cardano-cli --version 
```

Now for the real reason that we mapped the output volume to `cardano-cli`
Lets run a command outputs a file as the result. `Note` by the time you read this we may have moved to a later era than `--mary-era`

```shell
cardano-cli query protocol-parameters --out-file /var/cardano/local/protocol.json --mary-era --mainnet
```

An example of the above command on `testnet`
```shell
cardano-cli query protocol-parameters --out-file /var/cardano/local/protocol.json --mary-era --testnet-magic 1097911063
```

So what happened?

The `cardano-cli` used our alias `cardano-cli` to run the above command on the Cardano node running in Docker. It output the file to `/var/cardano/local/` which is mapped to the volume on the host machine at `/home/ec2-user/cardano/output`

If we look in `/home/ec2-user/cardano/output` we will fine our `protocol.json` output file.

```shell
ls /home/ec2-user/cardano/output
```

Now we should see `protocol.json`. This will be important later when setting up a Stake Pool.

Remember until we set up monitoring you can use the live view to see the state your your relay node. 

````shell
docker exec -it prodrl1 gLiveView
````

That's it.
