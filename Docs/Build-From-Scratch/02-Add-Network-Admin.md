# Adding network administrator

Currently administrators of R4 has administrative rights over the network.
We are going to allow administrators of other organization(R1) to administer the network.
The goal is depicted as:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.2.1.png "Target network - 02")

R1 : org1.com

## Create a self-signed CA certificate for R1 (Orderer)

Locate yourself in a path where `openssl.cnf` file exists.
*I assume that you have just done [01-Create-Network.md](https://github.com/ChoiSD/how-to-Hyperledger-Fabric/blob/master/Docs/Build-From-Scratch/01-Create-Network.md)*

We need to create two chains of trust for R1(org1.com), as R1 will participate in not only Orderer consortium but also Channel consortium.
In this document, we are going to create a chain of trust for R1 Orderer.

Generate a private key and a self-signed certificate for R1 (Orderer):

```bash
# Create MSP folders
mkdir -p org1.com/NC4/{ca,users}
# Generate Self-signed CA certificate
openssl req -new -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -x509 \
    -days 365 \
    -subj "/C=KR/ST=Seoul/L=Gangdong-gu/O=order.org1.com/OU=Blockchain/CN=ca.order.org1.com" \
    -config ./openssl.cnf -extensions v3_user \
    -keyout org1.com/NC4/ca/private.key \
    -out org1.com/NC4/ca/ca.order.org1.com-cert.pem
```

## Run Fabric CA server(CA1 Orderer)

Run Fabric CA server:

```bash
PRIVATE=$(ls org1.com/NC4/ca/*.key)
PUBLIC=$(ls org1.com/NC4/ca/*.pem)
docker run -d --name ca.order.org1.com --hostname ca.order.org1.com \
        --network howto_network \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        hyperledger/fabric-ca:1.4.0 \
        fabric-ca-server start -b admin:adminpw
```

## Get admin's certificate

Get admin user's certificate from CA server:

```bash
mkdir -p org1.com/NC4/users/Admin@order.org1.com/msp/admincerts
# Get IP address of CA server
IP=$(docker inspect ca.order.org1.com -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
./bin/fabric-ca-client enroll -H $PWD/org1.com/NC4/Admin@order.org1.com -u http://admin:adminpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=order.org1.com
# Set Admin Certificate
cp org1.com/NC4/users/Admin@order.org1.com/msp/signcerts/cert.pem org1.com/NC4/users/Admin@order.org1.com/msp/admincerts/
```

## Make MSP directory

Make MSP structure & Copy proper certificates:

```bash
mkdir -p org1.com/NC4/msp/{admincerts,cacerts}
# Copy certificates
cp org1.com/NC4/ca/ca.order.org1.com-cert.pem org1.com/NC4/msp/cacerts/
cp org1.com/NC4/users/Admin@order.org1.com/msp/signcerts/cert.pem org1.com/NC4/msp/admincerts/
```

## Add administrator of R1 into NC4 consortium

Write a update network configuration file:

```bash
cat > configtx.yaml <<EOF
###############################################
Organizations:
  - &Org4
    Name: Org4
    ID: Org4
    MSPDir: org4.com/msp
    Policies:
      Readers:
        Type: Signature
        Rule: "OR('Org4.member')"
      Writers:
        Type: Signature
        Rule: "OR('Org4.member')"
      Admins:
        Type: Signature
        Rule: "OR('Org4.admin')"
  - &Org1NC4
    Name: Org1NC4
    ID: Org1NC4
    MSPDir: org1.com/NC4/msp
    Policies:
      Readers:
        Type: Signature
        Rule: "OR('Org1NC4.member')"
      Writers:
        Type: Signature
        Rule: "OR('Org1NC4.member')"
      Admins:
        Type: Signature
        Rule: "OR('Org1NC4.admin')"
###############################################
Orderer: &Orderer
  OrdererType: solo
  Addresses:
    - orderer.org4.com:7050
  BatchTimeout: 2s
  BatchSize:
    MaxMessageCount: 10
    AbsoluteMaxBytes: 10 MB
    PreferredMaxBytes: 512 KB
  MaxChannels: 0
  Policies:
    Readers:
      Type: ImplicitMeta
      Rule: "ANY Readers"
    Writers:
      Type: ImplicitMeta
      Rule: "ANY Writers"
    Admins:
      Type: ImplicitMeta
      Rule: "ALL Admins"
    BlockValidation:
      Type: ImplicitMeta
      Rule: "ANY Writers"
  Organizations:
  Capabilities:
    V1_1: true
###############################################
Profiles:
  HowToDoc2:
    Policies:
      Readers:
        Type: ImplicitMeta
        Rule: "ANY Readers"
      Writers:
        Type: ImplicitMeta
        Rule: "ANY Writers"
      Admins:
        Type: ImplicitMeta
        Rule: "ALL Admins"
    Capabilities:
      V1_3: true
    Orderer:
      <<: *Orderer
      Organizations:
        - *Org4
    Consortiums:
      NC4:
        Organizations:
          - *Org4
          - *Org1NC4
EOF
```

Generate an update genesis block with updated NC4:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc2 -channelID syschannel -outputBlock ./channel-artifacts/genesis.block
```

## Run orderer(O4) with updated configuration

```bash
# Stop existing orderer
docker rm -f orderer
# Run orderer
docker run -d --name orderer.org4.com --hostname orderer.org4.com \
        --network howto_network \
        -v $PWD/channel-artifacts/genesis.block:/var/hyperledger/fabric/genesis.block \
        -v $PWD/org4.com/users/orderer.org4.com/msp:/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LISTENADDRESS=0.0.0.0 \
        -e ORDERER_GENERAL_LOGLEVEL=debug \
        -e ORDERER_GENERAL_GENESISMETHOD=file \
        -e ORDERER_GENERAL_GENESISFILE=/var/hyperledger/fabric/genesis.block \
        -e ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LOCALMSPID=Org4 \
        hyperledger/fabric-orderer:1.4.0 \
        orderer
```