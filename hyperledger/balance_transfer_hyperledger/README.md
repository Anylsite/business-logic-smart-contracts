# Guide to run setups

### Prerequisites 
* Clone https://github.com/AnyLedger/fabric-samples.git to your local
* You need to have 
    * Go - 1.10.x
    * Docker >= 17.06.2-ce
    * [Node.js](https://nodejs.org/) >= 8.9.x
    * npm 
    * cURL 
For more information on prerequisites : [Hyperledger Fabric Prerequisites](https://hyperledger-fabric.readthedocs.io/en/latest/prereqs.html)

* Fabric Images
    ```sh
    $ cd fabric-samples/scripts
    ```
    Make sure bootstrap.sh has `VERSION` set to `1.2.0 `
    ```sh
    $ ./bootstrap.sh
    ```
    This command downloads all the docker images and tools required to run Hyperledger Fabric.
    Once it successfully downloads everything required, proceed with individual example to run.

### How to run **first-network**?
```sh
$ cd fabric-samples/first-network
$ ./byfn.sh up
```

This generates all the required artifacts and brings up the simple network with 1 orderer, 4 peers of 2 different orgs with respective couchDBs and 1 cli container. Also, it runs end-to-end steps to verify whether network has all correct configurations. 

```sh
$ docker ps
```

This will list all the containers running on the machine. You can see `first-network` components running here.
You can find more detailed information on 'first-network' here : https://hyperledger-fabric.readthedocs.io/en/latest/build_network.html#

```sh
$ cd fabric-samples/first-network
$ ./byfn.sh down
```
Run this command once you're done using 'first-network'.
It removes all the containers and artifacts generated so that we can try other samples.
Also, we'll be using 'first-network' to in our next section to get the crypto material.

### Getting familiar Javascript SDK and running e2e setup using 'balance-transfer' example

This exercise mainly focus on below parts.
  - How to create a HLF network from scratch and setup channel for peers
  - How to use Javascript SDK to interact with HLF network
  - How to make whole sample cloud-deployable. 
  - How to add a new peer to already running network

### How to make whole sample cloud-deployable.

All projects under 'fabric-samples' runs on a bridge network when you run any docker-compose file which makes it little difficult to deploy different containers (read peers, orderers and other components) in different machines. We'll be using an overlay network so that network management becomes easy.

```sh
$ docker network create --attachable --driver overlay byfn-ov
```

This creates an overlay network named 'byfn-ov'. 
We'll be using this network for all of containers. You can inspect updated `fabric-samples/balance-transfer/artifacts/docker-compose.yaml` to see how we actually use this network.

Combining overlay network with [docker swarm](https://docs.docker.com/engine/swarm/#whats-next) makes the deployment possible across different machine over the internet.

### How to create a HLF network from scratch and setup channel for peers

We'll be using 'first-network' sample to generate crypto-material for our new network.

HLF network configs are mentioned in `fabric-samples/first-network/configtx.yaml` and `fabric-samples/first-network/crypto-config.yaml`

Open a new terminal window and let's call it **terminal_1**
```sh
$ cd fabric-samples/first-network
$ ./byfn.sh generate
```

This command uses 'cryptogen' tool to generate necessary certificates for orderer (orderer.example.com), 2 orgs (org1.example.com and org2.example.com), genesis block config and channel config transaction.

Open a new terminal window and let's call it **terminal_2**
```sh
$ cd fabric-samples/balance-transfer
$ rm -rf ./artifacts/channel/crypto-config/*
$ rm -rf ./fabric-client-kv-*
```
Here we clean up any existing artifacts from `crypto-config` and `fabric-client-kv-ORGNAME` folders. More on `fabric-client-kv' under Javascript SDK section.

Switch to **terminal_1**
```sh
$ cp -r crypto-config/* ../balance-transfer/artifacts/channel/crypto-config/
$ cp channel-artifacts/channel.tx ../balance-transfer/artifacts/channel/mychannel.tx 
$ cp channel-artifacts/genesis.block ../balance-transfer/artifacts/channel/genesis.block
```

With this step, we copied our newly created artifacts to 'balance-transfer' sample so that we can use it to run the network.

There are minor changes required to existing `balance-transfer/artifacts/docker-compose.yaml` and `balance-transfer/artifacts/network-config.yaml`. **Don't miss this.**

1. change private keys for each CA.
    Under `balance-transfer/artifacts/docker-compose.yaml`, for `ca.org1.example.com`
    set `FABRIC_CA_SERVER_CA_KEYFILE` and `FABRIC_CA_SERVER_TLS_KEYFILE` as guided below.
    Copy file name for a filename ending with `_sk`  from `balance-transfer/artifacts/channel/crypto-config/peerOrganizations/org1.example.com/ca/` and replace at appropriate place for above ENV variables.
    
    Follow the **similar** process for `ca.org1.example.com` as well.

2. change private key for Admin of each org
    Under `balance-transfer/artifacts/network-config.yaml`, for `adminPrivateKey` path under `organizations` section
    Copy file name for a filename ending with `_sk`  from `balance-transfer/artifacts/channel/crypto-config/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp/keystore` and replace at appropriate place for `adminPrivateKey` path variable.
    
    Follow the **similar** process for all `adminPrivateKey` with respective private keys.

That was quite a trouble!! 
But now we're ready to start our network.

switch to **terminal_2**,
```sh
$ docker-compose -f artifacts/docker-compose.yaml up -d 
```

This brings up all the components of our network up.
But without any `channel` or `smart-contract`, this network is of no use yet. That's what we'll accomplish in next step.

### How to use Javascript SDK to interact with HLF network

There are different ways to interact with HLF network.
 - use cli tool with different commands
 - use Go,Javascript,python or Java [SDK](https://hyperledger-fabric.readthedocs.io/en/release-1.2/getting_started.html#hyperledger-fabric-sdks)
 
'balance-transfer' example demonstrate the use of Javascript SDK. The project contains a simple node server which exposes different APIs such as 'create-channel', 'join-channel', 'install-chaincode', 'invoke-chaincode', 'get-tx-info', 'get-block-info' and many others.

To use this API server, we need to provide it out network configuration such as what are the peers, orderers endpoint, keys for admin, channel information etc. All this information is part of `network-config.yaml`. 
If network is running on any remote machine, we need to change endpoints in `network-config.yaml`. As of now our network is on the same machine and hence our endpoint is `localhost:port`

Open a new terminal window and let's call it **terminal_3**
```sh
$ cd fabric-samples/balance-transfer
$ npm install 
$ npm start
```

This brings up node server on 'localhost:4000'. 
Now we'll use the different APIs to setup channel and use chaincode (smart-contract).

switch to **terminal_2**
```sh
$ ./setup-channel.sh
```
It creates a channel named 'mychannel' and adds org1 & org2 to this channel using javascript SDK on back-end.

Now let's play around with some chaincode.
We'll be using DeviceManager.go chaincode here.

make sure you have DeviceManger.go at `fabric-samples/balance-transfer/artifacts/src/github.com/device_manager/go/` path. 
All smart contracts are under ['smart-contracts' repo](https://github.com/AnyLedger/smart-contracts)

Give a unique name to the chaincode to be installed. 
Change value for `CC_NAME` to desired unique name inside `balance-transfer/install-instantiate-cc.sh`
```sh
$ ./install-instantiate-cc.sh
```
It installs and instantiates chaincode on org1 and org2's peers. You can see 2 new docker containers running with name starting from `dev-peer*`. These are chaincode containers for repective peer of respective org.

Change value for `CC_NAME` inside `balance-transfer/invoke-on-1-query-on-2.sh` to the name set in **above step**.
```sh
$ ./invoke-on-1-query-on-2.sh
```

This step invokes our chaincode on org1's peer to store IPFS hash against deviceId. 
We're storing 'QmWWQSuPMS6aXCbZKpEjPHPUZN2NjB3YrhJTHsV4X3vb2t' against 'device1'.
We're also querying this information from peer0 of org2. You can see the output on cmd which shows the IPFS hash stored for 'device1'.

You can play around with chaincode invoke/query using `install-instantiate-cc.sh` and `invoke-on-1-query-on-2.sh` or [Postman](https://www.getpostman.com/) once you're familiar with all the APIs involved.

**fabric-client-kv- Folders**
Each API requires some identity from user to perform the action. Identity is nothing but the valid certificates for user or admin. Here we use 2 users, Jim for Org1 and Barry for Org2. We enroll this users which requests certificates from CA and stores it to `fabric-client-kv-ORGNAME` directory. Once successfully enrolled for subsequent requests, Node server loads certificates from this location.

###  How to add a new peer to already running network

To add a new peer, we need to get certificates for peer identity of a org. This can be achieved either by requesting certs from CA for that org or we can use cryptogen tool to provision one more peer from existing set of certs of any org.

We'll go back to 'first-network' project as we've generated our crypto materials using the config files under this project.

Open `fabric-samples/first-network/crypto-config.yaml` and increase the `count` under Peer template of the org you want to add new peer to. 
switch to **terminal_1**,

```sh
$ ../bin/cryptogen extend --config=./crypto-config.yaml
```

This will add new peer's artifacts under particular org's peer folder. 
For example, If you changed `count` for org1 to 3, `first-network/crypto-config/peerOrganizations/org1.example.com/peers` will have 3 peers under it.

Copy this newly generated artifacts to 'balance-transfer' using below or **similar** command.
```sh
$ cp -r ./crypto-config/peerOrganizations/org1.example.com/peers/peer2.org1.example.com ../balance-transfer/artifacts/channel/crypto-config/peerOrganizations/org1.example.com/peers/
```

Now that we have required artifacts for new peer, we can bring this peer up. In order to do this, we need to write a docker service for this peer.
`balance-transfer/artifacts/docker-compose-new-peer.yaml` can be used a template for this purpose. 
Make necessary changes such as service name, Port mappings, Environment variable etc(**if required**).

switch to **terminal_2**
```sh
$ docker-compose -f artifacts/docker-compose-new-peer.yaml up -d 
```

Once this peer is up, we can add it to 'mychannel' or any other channel.
We'll add this peer to 'mychannel' and we'll use existing API to perform this task.

But before doing that, we need add this peer to `network-config.yaml`.
There are 3 placed where we need to change here. Use existing config as reference and change accordingly.
    - `peers` section under `channel`
    - `peers` section under `organizations`
    - `peers` section where `url`, `eventUrl` etc. is defined. Refer to `docker-compose-new-peer.yaml` for port mappings.

You need to restart Node server which is running from **terminal_3** so that new changes are reflected in connection profile of Node server.

Open `balance-transfer/join-channel-new-peer.sh`and make necessary changes to 'join-channel' API call (**if required**).
```sh
$ ./join-channel-new-peer.sh
```

This will add new peer to existing 'mychannel' channel.

More Information on different API or on this example can be found at : https://github.com/hyperledger/fabric-samples/blob/release-1.2/README.md