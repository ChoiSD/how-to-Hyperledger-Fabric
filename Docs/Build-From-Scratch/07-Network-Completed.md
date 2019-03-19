# Network Completed

Almost done. One thing left is running a peer for R2.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/network.diagram.7.png "Target network - 07")

P2: a peer of R2 (peer0.org2.com)

## Get Ecert for P2

Register P2:

```bash
# Get IP address of CA server
IP=$(docker inspect ca.org2.com -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Register P2
./bin/fabric-ca-client register -H $PWD/org2.com/users/Admin@org2.com --id.name "peer0.org2.com" --id.type peer --id.maxenrollments 1 --id.secret peerpw
```

Enroll P2:

```bash
mkdir -p org2.com/users/peer0.org2.com/msp/admincerts
./bin/fabric-ca-client enroll -H $PWD/org2.com/users/peer0.org2.com -u http://peer0.org2.com:peerpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org2.com
# Copy Admin certs & config.yaml
cp org2.com/msp/admincerts/cert.pem org2.com/users/peer0.org2.com/msp/admincerts/
cp org2.com/msp/config.yaml org2.com/users/peer0.org2.com/msp/
```

## Run P2 & Join it to a channel C1

Run P2:

```bash
docker run -d --name peer0.org2.com --hostname peer0.org2.com \
  --network howto_network \
  -e CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock \
  -e CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=howto_network \
  -e CORE_PEER_GOSSIP_USELEADERELECTION=true \
  -e CORE_PEER_GOSSIP_ORGLEADER=false \
  -e CORE_PEER_ID=peer0.org2.com \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org2.com:7051 \
  -e CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -v /var/run/docker.sock:/host/var/run/docker.sock \
  -v $PWD/org2.com/users/peer0.org2.com/msp:/etc/hyperledger/fabric/msp \
  hyperledger/fabric-peer:1.4.0 \
  peer node start
```

Join a channel:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer channel join -b c1.block
```

## Install a smart contract in P2

Install S5 in P2:
> Don't need to instantiate a smart contract at this time, as we did it already before and that info is written in a block

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer chaincode install -n s5 -v 1.0 -p github.com/chaincodes/
```

## Query a smart contract

Query `A`:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer chaincode query -C c1 -n s5 -c '{"Args":["query","a"]}'
```

Query `B`:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer chaincode query -C c1 -n s5 -c '{"Args":["query","b"]}'
```

## Invoke a smart contract

Transfer `10` from `B` to `A`:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org2.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org2 \
  -e CORE_PEER_MSPCONFIGPATH=/org2/users/Admin@org2.com/msp \
  cli \
  peer chaincode invoke -o orderer.org4.com:7050 -C c1 -n s5 --peerAddresses peer0.org2.com:7051 -c '{"Args":["invoke","b","a","10"]}'
```