# Add Another Consortium

Since now, we built a network (a channel) between two organizations (R1 & R2).
Here is another organization R3. And R2 wants to make another consortium(X2) with R3 without R1.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.4/_images/network.diagram.9.png "Target network - 08")

R3: org3.com

## Create a self-signed CA certificate for R3

Generate a private key and a self-signed certificate for R3:

```bash
# Create MSP folders
mkdir -p org3.com/{users,ca}
# Generate Self-signed CA certificate
openssl req -new -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -x509 \
    -days 365 \
    -subj "/C=KR/ST=Seoul/L=Gangdong-gu/O=org3.com/OU=Blockchain/CN=ca.org3.com" \
    -config ./openssl.cnf -extensions v3_user \
    -keyout org3.com/ca/private.key \
    -out org3.com/ca/ca.org3.com-cert.pem
```

## Run Fabric CA server

```bash
PRIVATE=$(ls org3.com/ca/*.key)
PUBLIC=$(ls org3.com/ca/*.pem)
docker run -d --name ca.org3.com --hostname ca.org3.com \
        --network howto_network \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        hyperledger/fabric-ca:1.4.0 \
        fabric-ca-server start -b admin:adminpw
```

## Get admin's certificate

```bash
mkdir -p org3.com/users/Admin@org3.com/msp/admincerts
# Get IP address of CA server
IP=$(docker inspect ca.org3.com -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
./bin/fabric-ca-client enroll -H $PWD/org3.com/users/Admin@org3.com -u http://admin:adminpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org3.com
# Set Admin Certificate
cp org3.com/users/Admin@org3.com/msp/signcerts/cert.pem org3.com/users/Admin@org3.com/msp/admincerts/
```

## Make MSP directory

```bash
mkdir -p org3.com/msp/{admincerts,cacerts}
# Copy certificates
cp org3.com/ca/ca.org3.com-cert.pem org3.com/msp/cacerts/
cp org3.com/users/Admin@org3.com/msp/signcerts/cert.pem org3.com/msp/admincerts/
# Enable Node OUs
cat > org3.com/msp/config.yaml <<EOF
NodeOUs:
  Enable: true
  ClientOUIdentifier:
    Certificate: "cacerts/ca.org3.com-cert.pem"
    OrganizationalUnitIdentifier: "client"
  PeerOUIdentifier:
    Certificate: "cacerts/ca.org3.com-cert.pem"
    OrganizationalUnitIdentifier: "peer"
EOF
```

## Create a consortium(X2) to a network

We already built a network, which means all configurations are already in a block and cannot be changed by generating a new block.
So we need to update a channel configuration block in the same way as [Adding an Org to a Channel](https://hyperledger-fabric.readthedocs.io/en/release-1.4/channel_update_tutorial.html).

### Retrieve a current configuration block

```bash
# Fetch a config block
docker exec -it \
  -e CORE_PEER_LOCALMSPID=Org4 \
  -e CORE_PEER_MSPCONFIGPATH=/org4/users/orderer.org4.com/msp \
  cli \
  peer channel fetch config config_block.pb -o orderer.org4.com:7050 -c syschannel
# Get out a block from container
docker cp cli:/config_block.pb .
```

### Decode the block & Extract 'config' section

Extract 'config' section:

```bash
./bin/configtxlator proto_decode --input config_block.pb --type common.Block | jq '.data.data[0].payload.data.config' > config.json
```

Extract a base consortium(X1) from the 'config' section:

```bash
jq '.channel_group.groups.Consortiums.groups.X1' config.json > X1.json
```

### Create a new 'config' section

Create a new consortium(X2):

```bash
sed -i 's/Org1X1/Org3/g' X1.json
ADMIN_CERT=$(cat org3.com/msp/admincerts/cert.pem | base64 | tr -d '\n')
ROOT_CERT=$(cat org3.com/msp/cacerts/ca.org3.com-cert.pem | base64 | tr -d '\n')
jq --arg admin ${ADMIN_CERT} --arg root ${ROOT_CERT} '.groups.Org3.values.MSP.value.config.admins[0] = $admin | .groups.Org3.values.MSP.value.config.root_certs[0] = $root | .groups.Org3.values.MSP.value.config.fabric_node_ous.client_ou_identifier.certificate = $root | .groups.Org3.values.MSP.value.config.fabric_node_ous.peer_ou_identifier.certificate = $root' X1.json > X2.json
```

Add a new consortium into a 'config' section:

```bash
X2=$(cat X2.json)
jq --argjson x2 "$X2" '.channel_group.groups.Consortiums.groups.X2 = $x2' config.json > config_updated.json
```

### Encode both 'config' into binary blocks

```bash
./bin/configtxlator proto_encode --input config.json --type common.Config --output config.pb
./bin/configtxlator proto_encode --input config_updated.json --type common.Config --output config_updated.pb
```

### Calculate the delta between two blocks

```bash
./bin/configtxlator compute_update --channel_id syschannel --original config.pb --updated config_updated.pb --output x2_updated.pb
```

### Decode the delta and Wrap it in an envelope message

```bash
./bin/configtxlator proto_decode --input x2_updated.pb --type common.ConfigUpdate | jq . > x2_updated.json
echo '{"payload":{"header":{"channel_header":{"channel_id":"syschannel","type":2}},"data":{"config_update":'$(cat x2_updated.json)'}}}' | jq . > x2_enveloped.json
./bin/configtxlator proto_encode --input x2_enveloped.json --type common.Envelope --output x2_enveloped.pb
```

### Add a new consortium (X2) by updating a channel

Sign and Submit the message:

```bash
# Copy the message into 'cli' container
docker cp x2_enveloped.pb cli:/
# Sign the message with Org1NC4's admin
docker exec -it \
  -e CORE_PEER_LOCALMSPID=Org1NC4 \
  -e CORE_PEER_MSPCONFIGPATH=/org1/NC4/users/Admin@order.org1.com/msp \
  cli \
  peer channel signconfigtx -f x2_enveloped.pb
# Sign the message with Org4's admin
docker exec -it \
  -e CORE_PEER_LOCALMSPID=Org4 \
  -e CORE_PEER_MSPCONFIGPATH=/org4/users/Admin@org4.com/msp \
  cli \
  peer channel signconfigtx -f x2_enveloped.pb
# Submit the message
docker exec -it \
  -e CORE_PEER_LOCALMSPID=Org4 \
  -e CORE_PEER_MSPCONFIGPATH=/org4/users/orderer.org4.com/msp \
  cli \
  peer channel update -f x2_enveloped.pb \
  -o orderer.org4.com:7050 -c syschannel
```