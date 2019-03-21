# Peers and Ledgers

Now we will use the channel we defined. Let's deploy a peer which join the channel.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.5.png "Target network - 05")

P1: a peer of R1 (peer0.member.org1.com)

## Get Ecert for P1

Register P1:

```bash
# Get IP address of CA server
IP=$(docker inspect ca.member.org1.com -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Register P1
./bin/fabric-ca-client register -H $PWD/org1.com/X1/users/Admin@member.org1.com --id.name "peer0.member.org1.com" --id.type peer --id.maxenrollments 1 --id.secret peerpw

```

Enroll P1:

```bash
mkdir -p org1.com/X1/users/peer0.member.org1.com/msp/admincerts
./bin/fabric-ca-client enroll -H $PWD/org1.com/X1/users/peer0.member.org1.com -u http://peer0.member.org1.com:peerpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=member.org1.com
# Copy Admin certs & config.yaml
cp org1.com/X1/msp/admincerts/cert.pem org1.com/X1/users/peer0.member.org1.com/msp/admincerts/
cp org1.com/X1/msp/config.yaml org1.com/X1/users/peer0.member.org1.com/msp/
```

## Run P1

```bash
docker run -d --name peer0.member.org1.com --hostname peer0.member.org1.com \
  --network howto_network \
  -e CORE_VM_ENDPOINT=unix:///host/var/run/docker.sock \
  -e CORE_VM_DOCKER_HOSTCONFIG_NETWORKMODE=howto_network \
  -e CORE_PEER_GOSSIP_USELEADERELECTION=true \
  -e CORE_PEER_GOSSIP_ORGLEADER=false \
  -e CORE_PEER_ID=peer0.member.org1.com \
  -e CORE_PEER_ADDRESS=peer0.member.org1.com:7051 \
  -e CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.member.org1.com:7051 \
  -e CORE_PEER_GOSSIP_BOOTSTRAP=peer0.member.org1.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org1X1 \
  -v /var/run/docker.sock:/host/var/run/docker.sock \
  -v $PWD/org1.com/X1/users/peer0.member.org1.com/msp:/etc/hyperledger/fabric/msp \
  hyperledger/fabric-peer:1.4.0 \
  peer node start
```

## Join P1 into C1 channel

Join P1 into `c1` channel with a block:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.member.org1.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org1X1 \
  -e CORE_PEER_MSPCONFIGPATH=/org1/users/Admin@member.org1.com/msp \
  cli \
  peer channel join -b c1.block
```

Update anchor peers in `c1` channel:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.member.org1.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org1X1 \
  -e CORE_PEER_MSPCONFIGPATH=/org1/users/Admin@member.org1.com/msp \
  cli \
  peer channel update -o orderer.org4.com:7050 -c c1 -f /channel-artifacts/C1R1anchors.tx
```