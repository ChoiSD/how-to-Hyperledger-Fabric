# Add another Peer

Now we are going to add another peer for R3 and join C2 channel.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/network.diagram.11.png "Target network - 10")

P3: a peer of R3 (peer0.org3.com)

## Get Ecert for P3

Register P3:

```bash
# Get IP address of CA server
IP=$(docker inspect ca.org3.com -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Register P3
./bin/fabric-ca-client register -H $PWD/org3.com/users/Admin@org3.com --id.name "peer0.org3.com" --id.type peer --id.maxenrollments 1 --id.secret peerpw
```

Enroll P3:

```bash
./bin/fabric-ca-client enroll -H $PWD/org3.com/users/peer0.org3.com -u http://peer0.org3.com:peerpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org3.com
# Copy Admin certs & config.yaml
cp -R org3.com/msp/admincerts org3.com/users/peer0.org3.com/msp/
mv org3.com/users/peer0.org3.com/msp/cacerts/*.pem org3.com/users/peer0.org3.com/msp/cacerts/ca.org3.com-cert.pem
cp org3.com/msp/config.yaml org3.com/users/peer0.org3.com/msp/
```

## Run P3 & Join it to a channel C2

Run P3:

```bash
docker run -d --name peer0.org3.com --hostname peer0.org3.com \
  --network howto_network \
  -v /var/run/docker.sock:/host/var/run/docker.sock \
  -v $PWD/org3.com/users/peer0.org3.com/msp:/etc/hyperledger/fabric/msp \
  -e CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock \
  -e CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=howto_network \
  -e CORE_PEER_GOSSIP_USELEADERELECTION=true \
  -e CORE_PEER_GOSSIP_ORGLEADER=false \
  -e CORE_PEER_ID=peer0.org3.com \
  -e CORE_PEER_ADDRESS=peer0.org3.com:7051 \
  -e CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org3.com:7051 \
  -e CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org3.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org3 \
  hyperledger/fabric-peer:1.4.0 \
  peer node start
```

Join a channel:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org3.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org3 \
  -e CORE_PEER_MSPCONFIGPATH=/org3/users/Admin@org3.com/msp \
  cli \
  peer channel join -b c2.block
```

Update anchor peers in `c2` channel:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org3.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org3 \
  -e CORE_PEER_MSPCONFIGPATH=/org3/users/Admin@org3.com/msp \
  cli \
  peer channel update -o orderer.org4.com:7050 -c c2 -f /channel-artifacts/C2R3anchors.tx
```

## Install & Instantiate a smart contract

Install S6 in P3:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org3.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org3 \
  -e CORE_PEER_MSPCONFIGPATH=/org3/users/Admin@org3.com/msp \
  cli \
  peer chaincode install -n s6 -v 1.0 -p github.com/chaincodes/
```

Instantiate S5:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.org3.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org3 \
  -e CORE_PEER_MSPCONFIGPATH=/org3/users/Admin@org3.com/msp \
  cli \
  peer chaincode instantiate -o orderer.org4.com:7050 -C c2 -n s6 -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR ('Org2.peer','Org3.peer')"
```