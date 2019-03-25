# Join a peer to multiple channels

Make existing peer (P2) to join a new channel (C2).
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/network.diagram.12.png "Target Network - 11")

## Join P2 to a channel C2

Join a channel:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer channel join -b c2.block
```

Update anchor peers in `c2` channel:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer channel update -o orderer.org4.com:7050 -c c2 -f /channel-artifacts/C2R2anchors.tx
```

## Install & Instantiate a smart contract

Install S6 in P2:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer chaincode install -n s6 -v 1.0 -p github.com/chaincodes/
```