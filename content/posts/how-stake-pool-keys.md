+++
title = "How to set up a Stake Pool. (Part 1 \"Keys\")"
date = "2021-08-17T08:53:41+01:00"
author = ""
authorTwitter = "" #do not include @
cover = ""
tags = ["Cardano", "Staking"]
keywords = ["Cardano", "Stake pool"]
description = "Create your Cardano Stake Pool keys"
showFullContent = false
+++

## 1. Overview

The official course can be found here [`cardano foundation`](https://cardano-foundation.gitbook.io/stake-pool-course/stake-pool-guide/system-setup)

The goals for document.

- Create all the Keys needed to run a Stake Pool.
- Explain what all those keys are.
- Show you how to convert your keys you have created in Daedalus or Yori wallets to use when creating your Stake Pool. (Optional)

IMPORTANT for mainnet you need to keep your keys in cold storage. We are going to create mainnet keys in this document.

At the end of this we will have the following keys

| Key  | Description |
| -----|-------------|
|  cold.vkey    |    Cold verification key [here]({{< ref "#1-cold-keys" >}})      |
|   vrf.vkey   |   VRF verification key  [here]({{< ref "#2-vrf-key-pair" >}})      |
|  vrf.skey    |   VRF signing key          |
|  kes.vkey    |   KES verification key  [here]({{< ref "#3-kes-key-pair" >}})         |
|   kes.skey   |   KES signing key          |
|  payment.vkey     | Payment verification key            |
|  payment.skey   |   Payment signing key          |
|  stake.vkey    |    Staking verification key         |
|  stake.skey    |   Staking signing key          |
|  stake.addr    |    Registered stake address         |
|  paymentwithstake.addr    |   Funded address linked to stake          |
|  cold.skey    |    Cold signing key         |
|  cold.counter    |  Issue counter           |
|  node.cert    |    Operational certificate         |



 
    These "Node" keys represent the security of the blockchain and consist of the following keys
    - Cold Key pair
    - VRF Key pair
    - KES Key pair
    - Operational Certificate



You should have your alias `cardano-cli` set. If not see here [here]({{< ref "how-relay-node.md#create-the-cardano-cli-alias" >}} "Alias")

## 1. Cold Keys

```shell
cardano-cli node key-gen \
--cold-verification-key-file /var/cardano/output/cold.vkey \
--cold-signing-key-file /var/cardano/output/cold.skey \
--operational-certificate-issue-counter-file /var/cardano/output/cold.counter
```

## 2. VRF Key pair

```shell
cardano-cli node key-gen-VRF \
--verification-key-file /var/cardano/output/vrf.vkey \
--signing-key-file /var/cardano/output/vrf.skey
```

## 3. KES Key pair

    To create an operational certificate for a block-producing node, you need a Key Evolving Signature (KES) key pair, which authenticates who you are.
    A KES key can only evolve for a certain number of periods and becomes useless afterwards. 
    After the set number of periods has passed, the node operator must generate a new KES key pair, issue a new operational node certificate with that new key pair, and restart the node with the new certificate.

```shell
cardano-cli node key-gen-KES \
--verification-key-file /var/cardano/output/kes.vkey \
--signing-key-file /var/cardano/output/kes.skey
```

## 3. Operational certificates

    These are operators’ offline key pairs that include a certificate counter for new certificates. 
    It is the responsibility of the operator to manage both the hot (online), and cold (offline) keys for the pool. 
    Cold keys must be secure and should not reside on a device with internet connectivity. It is recommended to keep multiple backups of cold keys.

Firstly, we need to know the slots per KES period, we get it from the genesis file:

```shell
cat ~/cardano/config/mainnet-shelley-genesis.json | grep KESPeriod
```

`slotsPerKESPeriod": 129600`

So one period lasts `129600` slots.

Now we need the tip of the blockchain so we query using our Cardano node.

```shell
cardano-cli query tip --testnet-magic 1097911063
```


### 1 Create your stake keypairs

 cardano-cli stake-address key-gen \
 --verification-key-file /var/cardano/output/stake.vkey \
 --signing-key-file /var/cardano/output/stake.skey

### 1.1.2 Create your stake address

Now, we can create our stake address. This address CAN'T receive payments but will receive the rewards from participating in the protocol. We will save this address in the file stake.addr

```properties
cardano-cli stake-address build \
 --stake-verification-key-file /var/cardano/output/stake.vkey \
 --out-file /var/cardano/output/stake.addr \
 --mainnet
 ```

### 1.1.3 Regenerate payment address

 Now that we have a stake address, it is time to regenerate a payment address. This time we use both the stake verification key and payment verification key to build the address. With this, both addresses will be linked together and associated with one another.

```properties
cardano-cli address build \
 --payment-verification-key-file /var/cardano/output/payment.vkey \
 --stake-verification-key-file /var/cardano/output/stake.vkey \
 --out-file /var/cardano/output/paymentwithstake.addr \
 --mainnet
 ```

 ## 2.1 Register stake address in the blockchain

Before, we created our payment keys and address, which allow us to control our funds; we also created our stake keys and stake address, which allow us to participate in the protocol by delegating our stake or by creating a stake pool.

We need to register our stake key in the blockchain. To achieve this, we:

- Create a registration certificate
- Determin the deposit required
- Determine the Time-to-Live (TTL)
- Calculate the fees and deposit
- Submit the certificate to the blockchain with a transaction

### 2.1.1 Create a registration certificate

```properties
cardano-cli stake-address registration-certificate \
--stake-verification-key-file /var/cardano/output/stake.vkey \
--out-file /var/cardano/output/stake.cert
```

Once the certificate has been created, we must include it in a transaction to post it to the blockchain.

### 2.1.2 Determin the deposit required

```properties
cardano-cli query protocol-parameters --out-file /var/cardano/output/protocol.json --mainnet
```

The deposit amount can be found in the protocol.json under stakeAddressDeposit

`2000000`

### 2.1.3 Determin TTL

```properties
cardano-cli query tip --mainnet
```

### 2.1.4 Get transaction hash to use  

```properties
cardano-cli query utxo --address $(cat paymentLive.addr  ) --mainnet
```

### 2.1.5 Draft transaction

```properties
cardano-cli transaction build-raw --tx-in 4eac8d1b1c581fca71048ea05e67c55b2c8650f24603299d57422e99930d9339#0 --tx-out $(cat paymentwithstake.addr)+0 --ttl 0 --fee 0 --out-file /var/cardano/output/tx.raw --certificate-file /var/cardano/output/stake.cert
```

### 2.1.5 Calculate the minimum fees

```properties
cardano-cli transaction calculate-min-fee  --tx-body-file tx.raw  --tx-in-count 1  --tx-out-count 1  --witness-count 1  --byron-witness-count 1  --testnet-magic 1097911063  --protocol-params-file protocol.json
```

### 2.1.5 Build the transaction

```properties
cardano-cli transaction build-raw --tx-in 4eac8d1b1c581fca71048ea05e67c55b2c8650f24603299d57422e99930d9339#0 --tx-out $(cat paymentwithstake.addr )+9303023103 --ttl 33337142 --fee 176897 --out-file /var/cardano/output/tx.raw --certificate-file /var/cardano/output/stake.cert
```






properties

















Follow the instructions to install docker on AWS
https://docs.aws.amazon.com/AmazonECS/latest/developerguide/docker-basics.html

### 1.2 Additional steps

&nbsp;&nbsp;Enable Docker to auto-start after boot

&nbsp;&nbsp;`sudo systemctl enable --now docker`

## 2. Create a relay node in docker

### 2.1 Make directory structure on your local machine that we are going to use 

mkdir /home/ec2-user/cardano
mkdir /home/ec2-user/cardano/config
mkdir /home/ec2-user/cardano/config/keys
mkdir /home/ec2-user/cardano/data
mkdir /home/ec2-user/cardano/output
mkdir /home/ec2-user/cardano/tmp

cd /home/ec2-user/cardano/tmp

### 2.1 Optional, get the latest config files and store them in temp
Check the latest URL [cardano-config](https://cardanoupdates.com/docs/b8164e68-d89e-4574-bc78-97f66889e784)

### 2.2 Create the topology file in /home/ec2-user/cardano/config/mainnet-relay-topology.json
We will be making changes to this file a bit later

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

### 2.3 Start the node with basic configuration

```shell
docker run --detach \
    --name=prodrl1 \
    -p 3001:3001 \
    -e CARDANO_UPDATE_TOPOLOGY=true \
    -v node-data:/opt/cardano/data \
    nessusio/cardano-node run
```

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

Create a volume where we will store the custom configuration and keys

```properties
docker run --name=tmp -v cardano-relay-config:/var/cardano/config centos
```

```properties
docker cp /home/ec2-user/cardano/config/mainnet-relay-topology.json tmp:/var/cardano/config/mainnet-relay-topology.json
```

```properties
docker rm -f tmp
```

We just created a volume cardano-relay-config that links the host and the docker instance file systems

See the available volumes

```properties
docker volume ls
```

See the available volumes

```properties
docker volume inspect cardano-relay-config
```

### 2.4 Create a network bridge in docker

As we are building a production system we need to create a network bridge as we do not want to relay on the default docker bridge. We intend to run multiple relay nodes in this environment

Create the network bridge

```properties
docker network create --driver bridge prod-cardano-bridge
```

Inspect the network bridge

```properties
docker network inspect prod-cardano-bridge
```

### 2.5 Start cardano relay node with the TOPOLOGY updater

CARDANO_UPDATE_TOPOLOGY=true

```properties
docker run --detach --network prod-cardano-bridge \
    --name=prodrl1 \
    --restart=always \
    -p 3001:3001 \
    -e CARDANO_PORT=3001 \
    -e CARDANO_NETWORK=mainnet \
    -e CARDANO_UPDATE_TOPOLOGY=true \
    -v ipc:/opt/cardano/ipc \
    -v /home/ec2-user/cardano/data:/opt/cardano/data \
    -v cardano-relay-config:/var/cardano/config  \
    -v /home/ec2-user/cardano/output:/opt/cardano/output \
    nessusio/cardano-node run
```

View logs

```properties
docker logs -f prodrl1
docker stats
docker exec -it prodrl1 gLiveView
docker exec -it prodrl1 bash
```

### 2.7 Allow your node to sycn

Your node needs to sycn with the block chain. This will take several hours. Use the `gLiveView` to see the status : syncing percentage




cardano-cli address key-gen --verification-key-file payment.vkey --signing-key-file payment.skey


alias cardano-cli="docker run -it --rm \
  -v /home/ec2-user/cardano/output:/var/cardano/local \
  -v ipc:/opt/cardano/ipc \
  nessusio/cardano-node cardano-cli"



cardano-cli address key-gen --verification-key-file /var/cardano/local/payment.vkey --signing-key-file /var/cardano/local/payment.skey

cardano-cli address build --payment-verification-key-file /var/cardano/local/payment.vkey --out-file /var/cardano/local/payment.addr --mainnet

cardano-cli query utxo  --address $(cat payment.addr) --mainnet





