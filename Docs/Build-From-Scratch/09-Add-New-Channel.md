# Add a New Channel

In order to communication between R2 & R3 in private, a channel between them should be built.
Here, we are going to build a new channel (C2) betweene R2 & R3.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/network.diagram.10.png "Target network - 09")

C2: a channel for R2 & R3

## Generate a channel configuration block

Generate a private key and a self-signed certificate for R3:

* Hyperledger Fabric accepts following regex `[a-z][a-z0-9.-]*` only.
* So I will going to use `c2` as a channel name.

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc9 -channelID c2 -outputCreateChannelTx ./channel-artifacts/C2.tx
```

Define a anchor peer for R2:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc9 -channelID c2 -outputAnchorPeersUpdate ./channel-artifacts/C2R2anchors.tx -asOrg Org2
```

Define a anchor peer for R3:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc9 -channelID c2 -outputAnchorPeersUpdate ./channel-artifacts/C2R3anchors.tx -asOrg Org3
```

## Create a channel (C2)

```bash
docker exec -it \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer channel create -o orderer.org4.com:7050 -c c2 -f /channel-artifacts/C2.tx
```