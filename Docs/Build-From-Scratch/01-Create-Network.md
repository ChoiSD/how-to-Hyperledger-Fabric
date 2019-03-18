# Creating blockchain network

In this guide, we will create a basis of a blockchain network, Ordering service.
A goal of this guide can be conceptually represented by a below picture:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.2.png "Target network - 01")

R4 : org4.com

## Create a self-signed CA certificate

Create a openssl configuration file:

```bash
cat > openssl.cnf <<EOF
[ req ]
default_bits            = 256
default_md              = sha256
default_keyfile         = private.key
distinguished_name      = req_distinguished_name
extensions              = v3_user

[ v3_user ]
keyUsage                = critical, digitalSignature, keyEncipherment, keyCertSign, cRLSign
extendedKeyUsage        = anyExtendedKeyUsage
basicConstraints        = CA:TRUE
subjectKeyIdentifier    = hash

[ req_distinguished_name ]
countryName                     = KR
EOF
```

Generate a private key file and a self-signed certificate using OpenSSL:

```bash
# Create local MSP folders
mkdir -p org4.com/{users,ca}
# Generate Self-signed certificate using ECDSA
openssl req -new -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -x509 \
    -days 365 \
    -subj "/C=KR/ST=Seoul/L=Gangdong-gu/O=org4.com/OU=Blockchain/CN=ca.org4.com" \
    -config ./openssl.cnf -extensions v3_user \
    -keyout org4.com/ca/private.key \
    -out org4.com/ca/ca.org4.com-cert.pem
```

## Run Fabric CA server(CA4)

We are going to issue admin and orderer certificate thru Fabric CA server.

Run Fabric CA server:

```bash
PRIVATE=$(ls org4.com/ca/*.key)
PUBLIC=$(ls org4.com/ca/*.pem)
docker run -d --name ca.org4.com--hostname ca.org4.com \
        --network howto_network \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        hyperledger/fabric-ca:1.4.0 \
        fabric-ca-server start -b admin:adminpw
```

## Get admin's certificate

Download CA client binary:

```bash
curl https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric-ca/hyperledger-fabric-ca/linux-amd64-1.4.0/hyperledger-fabric-ca-linux-amd64-1.4.0.tar.gz | tar -xz -C .
```

Get admin user's certificate from CA server:

```bash
mkdir -p org4.com/users/Admin@org4.com/msp/admincerts
# Get IP address of CA server
IP=$(docker inspect ca.org4.com -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
./bin/fabric-ca-client enroll -H $PWD/org4.com/users/Admin@org4.com -u http://admin:adminpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org4.com
# Set Admin Certificate
cp org4.com/users/Admin@org4.com/msp/signcerts/cert.pem org4.com/users/Admin@org4.com/msp/admincerts/
```

## Get orderer's certificate

Register orderer:

```bash
./bin/fabric-ca-client register -H $PWD/org4.com/users/Admin@org4.com --id.name "orderer.org4.com" --id.type peer --id.maxenrollments 1 --id.secret ordererpw
```

Enroll orderer:

```bash
mkdir $PWD/org4.com/users/orderer.org4.com
./bin/fabric-ca-client enroll -H $PWD/org4.com/users/orderer.org4.com -u http://orderer.org4.com:ordererpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org4.com
```

## Make MSP directory

Make MSP structure & Copy proper certificates:

```bash
# Make directories
mkdir -p org4.com/msp/{admincerts,cacerts}
# Copy certificates
cp org4.com/ca/ca.org4.com-cert.pem org4.com/msp/cacerts/
cp org4.com/users/Admin@org4.com/msp/signcerts/cert.pem org4.com/msp/admincerts/
# Copy it to Orderer
cp -R org4.com/msp org4.com/users/orderer.org4.com/
```

## Create a network configuration(NC4)

Write network configuration file:

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
  Kafka:
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
  Capabilities:
    V1_1: true
  Organizations:
###############################################
Profiles:
  HowToDoc1:
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
EOF
```

Download Hyperledger Fabric tools:

```bash
curl https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.4.0/hyperledger-fabric-linux-amd64-1.4.0.tar.gz | tar -xz
```

Generate a genesis block with NC4:

```bash
mkdir channel-artifacts
bin/configtxgen -configPath $PWD -profile HowToDoc1 -channelID syschannel -outputBlock ./channel-artifacts/genesis.block
```

## Run orderer(O4)

```bash
docker run -d --name orderer.org4.com --hostname orderer.org4.com \
        --network howto_network \
        -v $PWD/channel-artifacts/genesis.block:/var/hyperledger/fabric/genesis.block \
        -v $PWD/org4.com/users/orderer.org4.com/msp:/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LISTENADDRESS=0.0.0.0 \
        -e ORDERER_GENERAL_LOGLEVEL=DEBUG \
        -e ORDERER_GENERAL_GENESISMETHOD=file \
        -e ORDERER_GENERAL_GENESISFILE=/var/hyperledger/fabric/genesis.block \
        -e ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LOCALMSPID=Org4 \
        hyperledger/fabric-orderer:1.4.0 \
        orderer
```