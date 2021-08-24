+++
title = "Set Up Stake Pool"
date = "2021-08-20T12:12:06+01:00"
author = "Shawn V"
authorTwitter = "" #do not include @
cover = "img/cardano/c1.png"
tags = ["Cardano", "Stake Pool"]
keywords = ["Cardano", "Stake"]
description = "Set up and start your stake pool"
showFullContent = false
+++


## Intro

First we need to know what a Cardano Producing node actually is?

    As a minimum, a stake pool must have one block-producing node and one relay node.
    Block producing nodes hold the keys and certificates that are necessary to issue blocks on the blockchain. 
    Block nodes connect to the network via relay nodes. 

![Cardano](/img/cardano-relay.png)

## Install

Our relay nodes run within Docker on AWS and we plan to do the same with our block-producing node. 

I am just going to repeat the steps here but if you are going to run your block-producing node on the same server as your relay nodes you can skip this section.

### How to set up a Cardano node on `mainnet`

I am not going to explain how to set up an AWS EC2 instance so if you do not know you will need to read AWS [docs](https://aws.amazon.com/ec2/getting-started/).

Form this point I am assuming you have SSH'd onto your running AWS EC2 instance and are ready to begin running commands.

#### 1. Install docker

Follow the instructions to [*install docker on AWS*](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html)

#### 1.2 Additional docker steps

After following the instructions above you need to exit your EC2 instance and re-login to it. Then enable Docker to auto-start after boot.

````shell
 sudo systemctl enable --now docker
````

Now `docker info` will work.

#### 2.1 Make directory structure on your local machine that we are going to use as volumes in docker

We am going to run one block-producing node in docker called `prodb1`

````shell
mkdir ~/cardano;
mkdir ~/cardano/config;
mkdir ~/cardano/data;
mkdir ~/cardano/output;
````

#### 2.1 Download config files

Download the latest config files. Don't rely on the links below to be correct always check [on Cardano](https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/index.html). Replace with testnet links if setting up on the test environment.

`mainnet`

````shell
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-config.json -P ~/cardano/config/
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-byron-genesis.json -P ~/cardano/config/;
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-shelley-genesis.json -P ~/cardano/config/;
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-topology.json  -P ~/cardano/config/
wget https://hydra.iohk.io/job/Cardano/cardano-node/cardano-deployment/latest-finished/download/1/mainnet-alonzo-genesis.json  -P ~/cardano/config/
````

### Configure your topology file to point to our two relay nodes.

Update the `mainnet-topology.json` file to point to our relay nodes.  
Change the x.x.x.x.x in the to your IP address and update the port.

```shell
{
  "Producers": [
    {
      "addr": "x.x.x.x.x",
      "port": 3001,
      "valency": 2
    },
     {
      "addr": "x.x.x.x.x",
      "port": 3002,
      "valency": 2
    }
  ]
}
```

At a later stage you will need to update the relay nodes to point to our block node. This is what we want to achieve.

![Cardano](/img/cardano/c2.png)

If you want to monitor your application using Prometheus and Grafana you will need to edit the file `mainnet-config.json` before we copy it.
Uncomment the section

```yaml
  hasPrometheus:
    - "127.0.0.1"
    - 12789
```

### 2.3 Create docker volumes

Create a volume where we will store the custom configuration and keys

````shell
docker run --name=tmp -v cardano-block-config:/var/cardano/config centos
docker cp ~/cardano/config/kes.skey tmp:/var/cardano/config/kes.skey
docker cp ~/cardano/config/vrf.skey tmp:/var/cardano/config/vrf.skey
docker cp ~/cardano/config/node.cert tmp:/var/cardano/config/node.cert
docker cp ~/cardano/config/mainnet-topology.json tmp:/var/cardano/config/mainnet-topology.json
docker cp ~/cardano/config/mainnet-byron-genesis.json tmp:/var/cardano/config/mainnet-byron-genesis.json
docker cp ~/cardano/config/mainnet-shelley-genesis.json tmp:/var/cardano/config/mainnet-shelley-genesis.json
docker cp ~/cardano/config/mainnet-alonzo-genesis.json tmp:/var/cardano/config/mainnet-alonzo-genesis.json
docker cp ~/cardano/config/mainnet-config.json tmp:/var/cardano/config/mainnet-config.json
docker rm -f tmp
````

We just created a volume `cardano-block-config` that links the host and the docker instance file systems.

Useful commands for inspecting volumes:

```shell
docker volume ls
```

Inspect a volume. `Don't skip this it's very useful to see how it maps the container to the host filesystem.`

```shell
docker volume inspect cardano-block-config
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

#### 2.5 Start cardano block producing node

The fully configured node will

- Auto update the topology
- Run using the network bridge *prod-cardano-bridge*
- Map a volume named *`ipc`* for when we use the *cardano-cli* commands
- Map the additional volumes `data`, `config` and `output`
- Auto restart our relay if docker to the host is restarted

```properties
docker run --detach --network prod-cardano-bridge \
--name=prodb1 \
--restart=always \
-p 3001:3001 \
-p 12789:12789 \
-e CARDANO_BIND_ADDR=0.0.0.0 \
-e CARDANO_PORT=3001 \
-e CARDANO_BLOCK_PRODUCER=true \
-e CARDANO_NETWORK=mainnet \
-e CARDANO_TOPOLOGY=/var/cardano/config/mainnet-topology.json \
-e CARDANO_SHELLEY_KES_KEY=/var/cardano/config/kes.skey \
-e CARDANO_SHELLEY_VRF_KEY=/var/cardano/config/vrf.skey \
-e CARDANO_SHELLEY_OPERATIONAL_CERTIFICATE=/var/cardano/config/node.cert \
-e CARDANO_NODE_SOCKET_PATH=/opt/cardano/ipc \
-e CARDANO_DATABASE_PATH=/opt/cardano/data \
-v cardano-block-config:/var/cardano/config \
-v /home/ec2-user/cardano/data:/opt/cardano/data \
nessusio/cardano-node run
```



Make sure everything is working  
Tail the Cardano node logs
````shell
docker logs -f prodb1
````

View the Docker stats like cpu, Memory usage etc
````shell
docker stats
````

View the cardano live view for the node
```shell
docker exec -it prodb1 gLiveView
```

#### 2.6 Useful maintenance commands

Tail the Cardano node logs
````shell
docker logs -f prodb1
````

View the Docker stats like cpu, Memory usage etc
````shell
docker stats
````

View the cardano live view for the node
```shell
docker exec -it prodb1 gLiveView
```

Access the Docker image in interactive mode using bash
```shell
docker exec -it prodb1 bash
```

### 2.7 Now we start the process of registering our pool with the blockchain


- Register your stake address on the blockchain
- Create the JSON metadata file
- Host the file

#### 2.7.1 Register your stake address on the blockchain

- Create the certificate
- Submit the certificate with a transaction and a fee (deposit 200000) to the blockchain

```shell
cardano-cli stake-address registration-certificate \
--stake-verification-key-file /var/cardano/output/stake.vkey \
--out-file /var/cardano/output/stake.cert
```

For this you need to do the usual transaction flow.

```shell
cardano-cli transaction build-raw --tx-in df938cf3dc586d6200a2eb4fb1623da52c4a47e6a875c3c18ef76a39d1559ecd#0 --tx-out $(cat paymentwithstake.addr)+0 --ttl 0 --fee 0 --out-file /var/cardano/output/tx.raw --certificate-file /var/cardano/output/pool-registration.cert --certificate-file /var/cardano/output/delegation.cert
```

- Calculate fee
```shell
cardano-cli transaction calculate-min-fee  --tx-body-file /var/cardano/output/tx.raw  --tx-in-count 1  --tx-out-count   --witness-count 1  --byron-witness-count 0  --mainnet  --protocol-params-file /var/cardano/output/protocol.json
```

Fee `178173`

- Update the transaction with the fee. This transaction is different as it only has one out and one in.
  
```shell
cardano-cli transaction build-raw --tx-in cb2c126d95c663886489123bd4c0f117084a04cccfce0be0a06a4f7efa38bc09#0 --tx-out $(cat paymentwithstake.addr )+2000000 --tx-out $(cat paymentwithstake.addr)+7821827 --ttl 38095061 --fee 178173 --out-file /var/cardano/output/tx.raw --certificate-file /var/cardano/output/stake.cert
```

- Sign the transaction

```shell
cardano-cli transaction sign --tx-body-file /var/cardano/output/tx.raw --signing-key-file /var/cardano/output/payment.skey --signing-key-file /var/cardano/output/stake.skey --mainnet --out-file /var/cardano/output/tx.signed
```

- Submit the transaction
  
```shell
cardano-cli transaction submit --tx-file /var/cardano/output/tx.signed  --mainnet
```

Ok so now our Stake address is on the blockchain.

#### 2.7.2 Create the JSON metadata file

Create your file using this JSON format. This file needs to be hosted by you so and it has a requirement that the URL can be no longer than 64 characters.

```json
{
"name": "ALT-F decentralized future",
"description": "An alternative decentralized world for an exciting future" ,
"ticker": "ALT-F",
"homepage": "https://alt-f.io/"
}
```

I created this file on my server and ran the following command to generate the hash which is needed in the transaction

```shell
cardano-cli stake-pool metadata-hash --pool-metadata-file  pool-metadata.json
```

`21e4e092d0b04138730f40e18f5068277b4b96dbdf266d118b8ec6f367563fd5`

#### 2.7.2 Host your file

We just host this in AWS S3.  
I am not going to go into details but you create a public bucket and ensure the file is available everywhere. This process seems strange but it helps keep the blockchain decentralized and prevents websites being used as a way to slow down and attach the nodes.

https://altf.s3.eu-west-2.amazonaws.com/pool-metadata.json

This is my URL, it's 59 characters long so meets the requirement of 64 or less.

#### 2.7.3 Build your transaction to register your pool

```shell
cardano-cli stake-pool registration-certificate \
--mainnet \
--cold-verification-key-file /var/cardano/output/cold.vkey \
--vrf-verification-key-file /var/cardano/output/vrf.vkey \
--pool-pledge 30000000000 \
--pool-cost 340000000 \
--pool-margin 0.00 \
--pool-reward-account-verification-key-file /var/cardano/output/stake.vkey \
--pool-owner-stake-verification-key-file /var/cardano/output/stake.vkey \
--pool-relay-ipv4 3.21.219.104 \
--pool-relay-port 3001 \
--metadata-url https://altf.s3.eu-west-2.amazonaws.com/pool-metadata.json \
--metadata-hash 21e4e092d0b04138730f40e18f5068277b4b96dbdf266d118b8ec6f367563fd5 \
--out-file  /var/cardano/output/pool-registration.cert
```


#### 2.7.4 Generate delegation certificate (pledge)

We have to honor our pledge by delegating at least the pledged amount to our pool, so we have to create a delegation certificate to achieve this.

```shell
cardano-cli stake-address delegation-certificate \
--stake-verification-key-file /var/cardano/output/stake.vkey \
--cold-verification-key-file /var/cardano/output/cold.vkey \
--out-file /var/cardano/output/delegation.cert
```

This creates a delegation certificate which delegates fund from all stake addresses associated with key `stake.vkey` to the pool belonging to cold key `cold.vkey`.

#### 2.7.4 Submit the pool certificate and delegation certificate to the blockchain

Finally we need to submit the pool registration certificate and the delegation certificate(s) to the blockchain by including them in one or more transactions. We can use one transaction for multiple certificates, the certificates will be applied in order.
As before, we start by drafting the transaction:

Step 1. First we start by drafting the transaction.

```shell
cardano-cli transaction build-raw --tx-in f30ddef486485c8024c5fdf1a1754033932ca884af003aa3b668414830e7fe61#0 --tx-out $(cat paymentwithstake.addr)+0 --ttl 0 --fee 0 --out-file /var/cardano/output/tx.raw --certificate-file /var/cardano/output/pool-registration.cert --certificate-file /var/cardano/output/delegation.cert
```

Step 2. Calculate the fees

cardano-cli query utxo \
    --address $(cat paymentwithstake.addr) \
    --mainnet > fullUtxo.out

```shell
cardano-cli transaction calculate-min-fee  --tx-body-file /var/cardano/output/tx.raw  --tx-in-count 1  --tx-out-count 1  --witness-count 1  --byron-witness-count 0  --mainnet  --protocol-params-file /var/cardano/output/protocol.json
```

`184817 Lovelace`

Total in my TX = 600000000 
Total amount to pay = 500000000
Fee = 184817

Formula 600000000 - 500000000 - 184817 = 99810739

Step 2. Build the transaction with the fees

```shell
cardano-cli transaction build-raw --tx-in f30ddef486485c8024c5fdf1a1754033932ca884af003aa3b668414830e7fe61#0 --tx-out $(cat paymentwithstake.addr )+499810563 --ttl 38137563 --fee 189437 --out-file /var/cardano/output/tx.raw --certificate-file /var/cardano/output/pool-registration.cert --certificate-file /var/cardano/output/delegation.cert
```

Step 3. Sign the transaction

```shell
cardano-cli transaction sign --tx-body-file /var/cardano/output/tx.raw --signing-key-file /var/cardano/output/payment.skey --signing-key-file /var/cardano/output/stake.skey --signing-key-file /var/cardano/output/cold.skey --mainnet --out-file  /var/cardano/output/tx.signed
```

Step 4. Submit the transaction

```shell
cardano-cli transaction submit --tx-file /var/cardano/output/tx.signed  --mainnet
```

Get the ID

```shell
cardano-cli stake-pool id --cold-verification-key-file /var/cardano/output/cold.vkey
```
