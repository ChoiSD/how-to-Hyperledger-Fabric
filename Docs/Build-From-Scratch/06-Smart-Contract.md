# Smart Contract

We are going to install and instantiate smart contract in a peer we deployed in the last chapter.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/network.diagram.6.png "Target network - 06")

S5: a smart contract

> I'm going to use [a simple chaincode in `fabric-samples`](https://github.com/hyperledger/fabric-samples/blob/release-1.4/chaincode/chaincode_example02/go/chaincode_example02.go).
> And, for `A1` in the picture, I'm not going to touch in this guide.

## Make a smart contract

Get a sample chaincode:

```bash
curl https://raw.githubusercontent.com/hyperledger/fabric-samples/release-1.4/chaincode/chaincode_example02/go/chaincode_example02.go -o chaincode/chaincode.go
```

## Install a smart contract in P1

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.member.org1.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org1X1 \
  -e CORE_PEER_MSPCONFIGPATH=/org1/users/Admin@member.org1.com/msp \
  cli \
  peer chaincode install -n s5 -v 1.0 -p github.com/chaincodes/
```

## Instantiate a smart contract in P1

> I'm going to use `"OR ('Org1X1.peer','Org2.peer')"` as an endorsement policy because we only have ONE peer on the network :)

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.member.org1.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org1X1 \
  -e CORE_PEER_MSPCONFIGPATH=/org1/users/Admin@member.org1.com/msp \
  cli \
  peer chaincode instantiate -o orderer.org4.com:7050 -C c1 -n s5 -v 1.0 -c '{"Args":["init","a","100","b","200"]}' -P "OR ('Org1X1.peer','Org2.peer')"
```

## Invoke a smart contract

Transfer `10` from `A` to `B`:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.member.org1.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org1X1 \
  -e CORE_PEER_MSPCONFIGPATH=/org1/users/Admin@member.org1.com/msp \
  cli \
  peer chaincode invoke -o orderer.org4.com:7050 -C c1 -n s5 --peerAddresses peer0.member.org1.com:7051 -c '{"Args":["invoke","a","b","10"]}'
```

## Query a smart contract

Query `A`:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.member.org1.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org1X1 \
  -e CORE_PEER_MSPCONFIGPATH=/org1/users/Admin@member.org1.com/msp \
  cli \
  peer chaincode query -C c1 -n s5 -c '{"Args":["query","a"]}'
```

Query `B`:

```bash
docker exec -it \
  -e CORE_PEER_ADDRESS=peer0.member.org1.com:7051 \
  -e CORE_PEER_LOCALMSPID=Org1X1 \
  -e CORE_PEER_MSPCONFIGPATH=/org1/users/Admin@member.org1.com/msp \
  cli \
  peer chaincode query -C c1 -n s5 -c '{"Args":["query","b"]}'
```