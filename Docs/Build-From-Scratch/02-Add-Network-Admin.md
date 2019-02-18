# Adding network administrator

Currently administrators of R4 has administrative rights over the network.
We are going to allow administrators of other organization(R1) to administer the network.
The goal is depicted as:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.2.1.png "Target network - 02")

R1 : org1.com

## Create a self-signed CA certificate for R1

Locate yourself in a path where `openssl.cnf` file exists.
*I assume that you have just done [01-Create-Network.md](https://github.com/ChoiSD/how-to-Hyperledger-Fabric/blob/master/Docs/Build-From-Scratch/01-Create-Network.md)*

Generate a private key and a self-signed certificate for R1:

```bash
# Create MSP folders
mkdir -p org1.com/{ca,users}; cd org1.com/ca
# Generate EC paramter with the group 'prime256v1'
openssl ecparam -out param.out -name prime256v1
# Generate Self-signed CA certificate
openssl req -newkey ec:param.out -nodes -keyout private.key -x509 -days 3650 -out ca.org4.com-cert.pem -extensions v3_user -config ../../openssl.cnf
```

As there are some pre-configured values in `openssl.cnf`, we need to do put some values in this time like below:

```bash
Country Name (2 letter code) [KR]:
Organization Name (eg, company) [org4.com]:org1.com
Organizational Unit Name (eg, section) [Blockchain Biz.]:
Common Name (eg, your name or your servers hostname) [ca.org4.com]:ca.org1.com
```

Rename private key and Cleanse:

```bash
# Rename private key
mv private.key $(openssl x509 -noout -pubkey -in ca.org2.com-cert.pem | openssl asn1parse -strparse 23 -in - | openssl dgst -sha256 | awk '{print $2}')_sk
rm param.out
```

## Run Fabric CA server(CA1)

Run Fabric CA server:

```bash
PRIVATE=$(ls *_sk)
PUBLIC=$(ls *.pem)
docker run -d --name ca0_org1 --hostname ca0.org1.com \
        --network howto_network \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        hyperledger/fabric-ca:1.4.0 \
        fabric-ca-server start -b admin:adminpwOrg1
```

## Get admin's certificate

Get admin user's certificate from CA server:

```bash
cd ../users
mkdir admin
# Get IP address of CA server
IP=$(docker inspect ca0_org1 -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
../../bin/fabric-ca-client enroll -H $PWD/admin -u http://admin:adminpwOrg1@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org1.com
```

Create MSP structure & Copy proper certificates:

```bash
cd ..
mkdir -p msp/{admincerts,tlscacerts}
cp -R users/admin/msp/cacerts msp/
cp users/admin/msp/signcerts/cert.pem msp/admincerts/Admin@org1.com-cert.pem
```

## Allow member of other organization(R1) to administer a network

Write a update network configuration file:

```bash
cd ..
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
  - &Org1
    Name: Org1
    ID: Org1
    MSPDir: org1.com/msp
    Policies:
      Readers:
        Type: Signature
        Rule: "OR('Org1.member')"
      Writers:
        Type: Signature
        Rule: "OR('Org1.member')"
      Admins:
        Type: Signature
        Rule: "OR('Org1.admin')"
###############################################
Capabilities:
  Channel: &ChannelCapabilities
    V1_3: true
  Orderer: &OrdererCapabilities
    V1_1: true
  Application: &ApplicationCapabilities
    V1_3: true
    V1_2: false
    V1_1: false
###############################################
Application: &ApplicationDefaults
  ACLs:
  Organizations:
  Policies:
    Readers:
      Type: ImplicitMeta
      Rule: "ANY Readers"
    Writers:
      Type: ImplicitMeta
      Rule: "ANY Writers"
    Admins:
      Type: ImplicitMeta
      Rule: "MAJORITY Admins"
  Capabilities:
    <<: *ApplicationCapabilities
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
      Rule: "ANY Admins"
    BlockValidation:
      Type: ImplicitMeta
      Rule: "ANY Writers"
  Organizations:
  Capabilities:
    <<: *OrdererCapabilities
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
        Rule: "MAJORITY Admins"
    Capabilities:
      <<: *ChannelCapabilities
    Application:
      <<: *ApplicationDefaults
    Orderer:
      <<: *Orderer
      Organizations:
        - *Org1
        - *Org4
    Consortiums:
      HowToConsortium:
        Organizations:
EOF
```

Generate an update genesis block with updated NC4:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc2 -channelID syschannel -outputBlock ./genesis.block
```

## Run orderer(O4) with updated configuration

```bash
# Stop existing orderer
docker rm -f orderer
# Run orderer
docker run -d --name orderer --hostname orderer.org4.com \
        --network howto_network \
        -v $PWD/genesis.block:/var/hyperledger/fabric/genesis.block \
        -v $PWD/org4.com/users/orderer/msp:/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LISTENADDRESS=0.0.0.0 \
        -e ORDERER_GENERAL_LOGLEVEL=debug \
        -e ORDERER_GENERAL_GENESISMETHOD=file \
        -e ORDERER_GENERAL_GENESISFILE=/var/hyperledger/fabric/genesis.block \
        -e ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LOCALMSPID=Org4 \
        hyperledger/fabric-orderer:1.4.0 \
        orderer
```