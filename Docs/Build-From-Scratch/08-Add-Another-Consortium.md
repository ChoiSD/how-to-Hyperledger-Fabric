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
cp org3.com/ca/ca.org3.com-cert.pem org3.com/users/Admin@org3.com/msp/cacerts/
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
  - &Org3
    Name: Org3
    ID: Org3
    MSPDir: org3.com/msp
    Policies:
      Readers:
        Type: Signature
        Rule: "OR('Org3.member')"
      Writers:
        Type: Signature
        Rule: "OR('Org3.member')"
      Admins:
        Type: Signature
        Rule: "OR('Org3.admin')"
    AnchorPeers:
      - Host: peer0.org3.com
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
Application: &Applications
  ACLs:
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
  Organizations:
  Capabilities:
    V1_3: true
###############################################
Profiles:
  HowToDoc8:
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
      X2:
        Organizations:
          - *Org2
          - *Org3

  HowToDoc9:
    Consortium: X2
    Application:
      <<: *Applications
      Organizations:
        - *Org2
        - *Org3
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
EOF
```

## Re-generate a genesis block

Before generating a genesis block, stop a orderer first:

```bash
docker stop orderer.org4.com
```

Then generate a new genesis block:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc8 -channelID syschannel -outputBlock ./channel-artifacts/genesis.block
```

Restart the orderer:

```bash
docker restart orderer.org4.com
```