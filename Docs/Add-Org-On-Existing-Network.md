# Adding one Org on an existing Fabric network
This guide is based on [IBM DeveloperWorks - Add an organization to your existing Hyperledger Fabric blockchain network using an easy tool](https://www.ibm.com/developerworks/cloud/library/cl-add-an-organization-to-your-hyperledger-fabric-blockchain/index.html).

Thanks [Bhargav Perepa](https://www.linkedin.com/in/bhargav-perepa/) and [Jason Yellick](https://www.linkedin.com/in/jason-yellick-03653757/) for your wonderful work!

And I'm just going to add some details in more polite way. :)

After completing this lab, you should have one additional Org:

* \*.org3.example.com

## Assumption

- Start this lab on the same path where byfn.sh resides on
- Fabric network is already created and operates ([BYFN sample](https://github.com/hyperledger/fabric-samples/tree/release/first-network))

## Prerequisite
 - Use latest Fabric build >= 1.1.0-preview

## Install jq tool
This lab requires `jq` binaries. Download them from the [jq repository](https://stedolan.github.io/jq/).

### OS X
```
wget https://github.com/stedolan/jq/releases/download/jq-1.5/jq-osx-amd64
chmod +x jq-osx-amd64
sudo mv jq-osx-amd64 /usr/local/bin/jq
```

### Linux
```
wget https://github.com/stedolan/jq/releases/download/jq-1.5/jq-linux64
chmod +x jq-linux64
sudo mv jq-linux64 /usr/local/bin/jq
```

## Create Certificates for Org3

Create a configuration file:
```
cat > crypto-config-add.yaml <<EOF
PeerOrgs:
- Name: Org3
  Domain: org3.example.com
  Template:
    Count: 1
  Users:
    Count: 1
EOF
```

Generate certificates from the configuration file:
```
../bin/cryptogen generate --config=./crypto-config-add.yaml
```

You got Org3's certificates on `./crypto-config/peerOrganizations/org3.example.com/`.

## Running configtxlator
[`configtxlator`](https://hyperledger-fabric.readthedocs.io/en/latest/configtxlator.html) tool is designed to support reconfigure Fabric network.

```
../bin/configtxlator start &
```

## Retrieve a current configuration
Retrieve current configuration at `cli` container:
```
docker exec -it cli peer channel fetch config config_block.pb -o orderer.example.com:7050 -c mychannel --tls --cafile ./crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem

docker cp cli:/opt/gopath/src/github.com/hyperledger/fabric/peer/config_block.pb .
```

## Decode the configuration
Decode fetched configuration file with configtxlator:
```
curl -X POST --data-binary @config_block.pb http://127.0.0.1:7059/protolator/decode/common.Block > config_block.json
```

## Extract `config` section
Extract `config` section from the decoded configuration file:
```
jq .data.data[0].payload.data.config config_block.json > config.json
```

Extract `Org1MSP` section from the config section:
```
jq .channel_group.groups.Application.groups.Org1MSP config.json > Org1MSP.json
```

## Create new configuration
Create `Org3MSP.json` file based on `Org1MSP.json`:
```
ADMIN_CERT=$(cat ./crypto-config/peerOrganizations/org3.example.com/users/Admin\@org3.example.com/msp/signcerts/Admin\@org3.example.com-cert.pem |base64 |tr -d '\n')
ROOT_CERT=$(cat ./crypto-config/peerOrganizations/org3.example.com/ca/ca.org3.example.com-cert.pem  |base64 |tr -d '\n')
TLS_ROOT_CERT=$(cat ./crypto-config/peerOrganizations/org3.example.com/tlsca/tlsca.org3.example.com-cert.pem |base64 |tr -d '\n')
jq --arg admin ${ADMIN_CERT} --arg root ${ROOT_CERT} --arg tls ${TLS_ROOT_CERT} '.values.MSP.value.config.admins[0] = $admin | .values.MSP.value.config.root_certs[0] = $root | .values.MSP.value.config.tls_root_certs[0] = $tls' Org1MSP.json > Org3MSP.json
sed -i 's/Org1MSP/Org3MSP/g' Org3MSP.json
```

Put a content of `Org3MSP.json` after `Org2MSP` in config.json file:
```
ORG3=$(cat Org3MSP.json)
jq --argjson org3 "$ORG3" '.channel_group.groups.Application.groups.Org3MSP = $org3' config.json > updated_config.json
```

## Encode both the original and modified configurations
Via configtxlatr, encode both configuration file, `config.json` & `updated_config.json`:
```
curl -X POST --data-binary @config.json http://127.0.0.1:7059/protolator/encode/common.Config > config.pb
curl -X POST --data-binary @updated_config.json http://127.0.0.1:7059/protolator/encode/common.Config > updated_config.pb
```

## Send them to configtxlator to compute the config update delta
Compute the config update delta:
```
curl -X POST -F original=@config.pb -F updated=@updated_config.pb http://127.0.0.1:7059/configtxlator/compute/update-from-configs -F channel=mychannel > config_update.pb
```

## Decode the config update and wrap it up into a config update envelope
Decode the config update file into JSON:
```
curl -X POST --data-binary @config_update.pb http://127.0.0.1:7059/protolator/decode/common.ConfigUpdate > config_update.json
```

Create an envelope for the config update message:
```
echo '{"payload":{"header":{"channel_header":{"channel_id":"mychannel", "type":2}},"data":{"config_update":'$(cat config_update.json)'}}}' > config_update_as_envelope.json
```

## Create the new config transaction
Encode the enveloped message into a protobuf format:
```
curl -X POST --data-binary @config_update_as_envelope.json http://127.0.0.1:7059/protolator/encode/common.Envelope > config_update_as_envelope.pb
```

Copy the new transaction on `cli` container:
```
docker cp config_update_as_envelope.pb cli:/opt/gopath/src/github.com/hyperledger/fabric/peer/
```

## Update the channel by submitting the new signed config transaction

Sign the configuration update transaction on all MSPs:

Add Org1MSP's signature to the new transaction
```
docker exec -it \
-e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt \
-e CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key \
-e CORE_PEER_LOCALMSPID=Org1MSP \
-e CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt \
-e CORE_PEER_TLS_ENABLED=true \
-e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp \
cli \
peer channel signconfigtx -f config_update_as_envelope.pb \
-o orderer.example.com:7050 --tls --cafile ./crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

Add Org2MSP's signature to the new transaction
```
docker exec -it \
-e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/ca.crt \
-e CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/server.key \
-e CORE_PEER_LOCALMSPID=Org2MSP \
-e CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer0.org2.example.com/tls/server.crt \
-e CORE_PEER_TLS_ENABLED=true \
-e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin@org2.example.com/msp \
cli \
peer channel signconfigtx -f config_update_as_envelope.pb \
-o orderer0.example.com:7050 --tls --cafile ./crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

Submit the updated transaction:
```
docker exec -it \
-e CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/ca.crt \
-e CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.key \
-e CORE_PEER_LOCALMSPID=Org1MSP \
-e CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/peers/peer0.org1.example.com/tls/server.crt \
-e CORE_PEER_TLS_ENABLED=true \
-e CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org1.example.com/users/Admin@org1.example.com/msp \
cli \
peer channel update -f config_update_as_envelope.pb \
-o orderer.example.com:7050 -c mychannel --tls --cafile ./crypto/ordererOrganizations/example.com/orderers/orderer.example.com/msp/tlscacerts/tlsca.example.com-cert.pem
```

Now adding Org3 is all done!

## Run a peer0.org3 node
Create a compose file for peer0.org3:
```
cat > docker-compose-peer0-org3.yaml <<EOF
version: '2'

networks:
  byfn:

services:
  peer0.org3.example.com:
    container_name: peer0.org3.example.com
    extends:
      file: base/peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer0.org3.example.com
      - CORE_PEER_ADDRESS=peer0.org3.example.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer0.org3.example.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer0.org3.example.com:7051
      - CORE_PEER_LOCALMSPID=Or3MSP
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls:/etc/hyperledger/fabric/tls
    ports:
      - 11051:7051
      - 11053:7053
    networks:
      - byfn
EOF
```

Start peer0.org3:
```
docker-compose -f docker-compose-peer0-org3.yaml up -d
```

## Execute a chaincode on peer2.org2:

Exeucte cli container's shell:
```
docker exec -it cli bash
```

On the shell, run following commands:
```
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/ca.crt
CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/server.key
CORE_PEER_LOCALMSPID=Org3MSP
CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/peers/peer0.org3.example.com/tls/server.crt
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org3.example.com/users/Admin\@org3.example.com/msp
CORE_PEER_ADDRESS=peer0.org3.example.com:7051

CHANNEL_NAME=mychannel
peer channel join -b ${CHANNEL_NAME}.block
peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```
