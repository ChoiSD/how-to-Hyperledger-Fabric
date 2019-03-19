# Defining a consortium

Now we have a base network.
Let's create a consortium which will make use of this network.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.3.png "Target network - 03")

R2 : org2.com

## Create a self-signed CA certificate for R1 (member) & R2

Locate yourself in a path where `openssl.cnf` file exists.
*I assume that you have just done [02-Add-Network-Admin.md](https://github.com/ChoiSD/how-to-Hyperledger-Fabric/blob/master/Docs/Build-From-Scratch/02-Add-Network-Admin.md)*

In order to make a consortium, we need to create another chain of trust for R1 member.

Generate a private key and a self-signed certificate for R1 (Member):

```bash
# Create MSP folders
mkdir -p org1.com/X1/{users,ca}
# Generate Self-signed CA certificate
openssl req -new -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -x509 \
    -days 365 \
    -subj "/C=KR/ST=Seoul/L=Gangdong-gu/O=member.org1.com/OU=Blockchain/CN=ca.member.org1.com" \
    -config ./openssl.cnf -extensions v3_user \
    -keyout org1.com/X1/ca/private.key \
    -out org1.com/X1/ca/ca.member.org1.com-cert.pem
```

Generate a private key and a self-signed certificate for R2:

```bash
# Create MSP folders
mkdir -p org2.com/{users,ca}
# Generate Self-signed CA certificate
openssl req -new -newkey ec -pkeyopt ec_paramgen_curve:prime256v1 -nodes -x509 \
    -days 365 \
    -subj "/C=KR/ST=Seoul/L=Gangdong-gu/O=org2.com/OU=Blockchain/CN=ca.org2.com" \
    -config ./openssl.cnf -extensions v3_user \
    -keyout org2.com/ca/private.key \
    -out org2.com/ca/ca.org2.com-cert.pem
```

## Run Fabric CA server(CA1 Member & CA2)

Run Fabric CA server (CA1 member):

```bash
PRIVATE=$(ls org1.com/X1/ca/*.key)
PUBLIC=$(ls org1.com/X1/ca/*.pem)
docker run -d --name ca.member.org1.com --hostname ca.member.org1.com \
        --network howto_network \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        hyperledger/fabric-ca:1.4.0 \
        fabric-ca-server start -b admin:adminpw
```

Run Fabric CA server (CA2):

```bash
PRIVATE=$(ls org2.com/ca/*.key)
PUBLIC=$(ls org2.com/ca/*.pem)
docker run -d --name ca.org2.com --hostname ca0.org2.com \
        --network howto_network \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        hyperledger/fabric-ca:1.4.0 \
        fabric-ca-server start -b admin:adminpw
```

## Get admin's certificate

Get admin user's certificate from CA server (CA1 member):

```bash
mkdir -p org1.com/X1/users/Admin@member.org1.com/msp/admincerts
# Get IP address of CA server
IP=$(docker inspect ca.member.org1.com -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
./bin/fabric-ca-client enroll -H $PWD/org1.com/X1/users/Admin@member.org1.com -u http://admin:adminpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=member.org1.com
# Set Admin Certificate
cp org1.com/X1/users/Admin@member.org1.com/msp/signcerts/cert.pem org1.com/X1/users/Admin@member.org1.com/msp/admincerts/
```

Get admin user's certificate from CA server (CA2):

```bash
mkdir org2.com/users/Admin@org2.com
# Get IP address of CA server
IP=$(docker inspect ca.org2.com -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
./bin/fabric-ca-client enroll -H $PWD/org2.com/users/Admin@org2.com -u http://admin:adminpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org2.com
# Set Admin Certificate
cp org2.com/users/Admin@org1.com/msp/signcerts/cert.pem org2.com/users/Admin@org2.com/msp/admincerts/

```

## Make MSP directory

Make MSP structure & Copy proper certificates for R1 Member:

```bash
mkdir -p org1.com/X1/msp/{admincerts,cacerts}
# Copy certificates
cp org1.com/X1/ca/ca.member.org1.com-cert.pem org1.com/X1/msp/cacerts/
cp org1.com/X1/ca/ca.member.org1.com-cert.pem org1.com/X1/users/Admin@member.org1.com/msp/cacerts/
cp org1.com/X1/users/Admin@member.org1.com/msp/signcerts/cert.pem org1.com/X1/msp/admincerts/
# Enable Node OUs
cat > org1.com/X1/msp/config.yaml <<EOF
NodeOUs:
  Enable: true
  ClientOUIdentifier:
    Certificate: "cacerts/ca.member.org1.com-cert.pem"
    OrganizationalUnitIdentifier: "client"
  PeerOUIdentifier:
    Certificate: "cacerts/ca.member.org1.com-cert.pem"
    OrganizationalUnitIdentifier: "peer"
EOF
```

Make MSP structure & Copy proper certificates for R2:

```bash
mkdir -p org2.com/msp/{admincerts,cacerts}
# Copy certificates
cp org2.com/ca/ca.org2.com-cert.pem org2.com/msp/cacerts/
cp org2.com/ca/ca.org2.com-cert.pem org2.com/users/Admin@org2.com/msp/cacerts/
cp org2.com/users/Admin@org2.com/msp/signcerts/cert.pem org2.com/msp/admincerts/
# Enable Node OUs
cat > org2.com/msp/config.yaml <<EOF
NodeOUs:
  Enable: true
  ClientOUIdentifier:
    Certificate: "cacerts/ca.org2.com-cert.pem"
    OrganizationalUnitIdentifier: "client"
  PeerOUIdentifier:
    Certificate: "cacerts/ca.org2.com-cert.pem"
    OrganizationalUnitIdentifier: "peer"
EOF
```

## Create a consortium(X1) to a network

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
  - &Org1X1
    Name: Org1X1
    ID: Org1X1
    MSPDir: org1.com/X1/msp
    Policies:
      Readers:
        Type: Signature
        Rule: "OR('Org1X1.member')"
      Writers:
        Type: Signature
        Rule: "OR('Org1X1.member')"
      Admins:
        Type: Signature
        Rule: "OR('Org1X1.admin')"
    AnchorPeers:
      - Host: peer0.member.org1.com
        Port: 7051
  - &Org2
    Name: Org2
    ID: Org2
    MSPDir: org2.com/msp
    Policies:
      Readers:
        Type: Signature
        Rule: "OR('Org2.member')"
      Writers:
        Type: Signature
        Rule: "OR('Org2.member')"
      Admins:
        Type: Signature
        Rule: "OR('Org2.admin')"
    AnchorPeers:
      - Host: peer0.org2.com
        Port: 7051
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
  HowToDoc3:
    Policies:
      Readers:
        Type: ImplicitMeta
        Rule: "ANY Readers"
      Writers:
        Type: ImplicitMeta
        Rule: "ANY Writers"
      Admins:
        Type: ImplicitMeta
        Rule: "MAJAORITY Admins"
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
      X1:
        Organizations:
          - *Org1X1
          - *Org2
EOF
```

Generate an updated genesis block:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc3 -channelID syschannel -outputBlock ./channel-artifacts/genesis.block
```