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
countryName                     = Country Name (2 letter code)
countryName_default             = KR
countryName_min                 = 2
countryName_max                 = 2
stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = Seoul
localityName                    = Locality Name (eg, city)
localityName_default            = Gangdong-gu
organizationName                = Organization Name (eg, company)
organizationName_default        = org4.com
organizationalUnitName          = Organizational Unit Name (eg, section)
organizationalUnitName_default  = Blockchain Biz.
commonName                      = Common Name (eg, your name or your server's hostname)
commonName_default              = ca.org4.com
commonName_max                  = 64
EOF
```

Generate a private key file and a self-signed certificate using OpenSSL:

```bash
# Create local MSP folders
mkdir -p org4.com/{users,ca}; cd org4.com/ca
# Generate EC paramter with the group 'prime256v1'
openssl ecparam -out param.out -name prime256v1
# Generate Self-signed CA certificate
openssl req -newkey ec:param.out -nodes -keyout private.key -x509 -days 3650 -out ca.org4.com-cert.pem -extensions v3_user -config ../../openssl.cnf
```

Use pre-configured values in `openssl.cnf`

```bash
Country Name (2 letter code) [KR]:
State or Province Name (full name) [Seoul]:
Locality Name (eg, city) [Gangdong-gu]:
Organization Name (eg, company) [org4.com]:
Common Name (eg, your name or your servers hostname) [ca.org4.com]:
```

Rename private key and Cleanse:

```bash
# Rename private key
mv private.key $(openssl x509 -noout -pubkey -in ca.org4.com-cert.pem | openssl asn1parse -strparse 23 -in - | openssl dgst -sha256 | awk '{print $2}')_sk
rm param.out
```

## Run Fabric CA server(CA4)

We are going to issue admin and orderer certificate thru Fabric CA server.

Run Fabric CA server:

```bash
PRIVATE=$(ls *_sk)
PUBLIC=$(ls *.pem)
docker run -d --name ca0_org4 --hostname ca0.org4.com \
        --network howto_network \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        hyperledger/fabric-ca:1.4.0 \
        fabric-ca-server start -b admin:adminpwOrg4
```

## Get admin's certificate

Download CA client binary:

```bash
cd ../users
curl https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric-ca/hyperledger-fabric-ca/linux-amd64-1.4.0/hyperledger-fabric-ca-linux-amd64-1.4.0.tar.gz | tar -xz -C ../../
```

Get admin user's certificate from CA server:

```bash
mkdir Admin@org4.com
# Get IP address of CA server
IP=$(docker inspect ca0_org4 -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
../../bin/fabric-ca-client enroll -H $PWD/Admin@org4.com -u http://admin:adminpwOrg4@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org4.com
```

## Get orderer's certificate

Register orderer:

```bash
../../bin/fabric-ca-client register -H $PWD/admin --id.name "orderer.org4.com" --id.type peer --id.maxenrollments 1 --id.secret ordererpw
```

Enroll orderer:

```bash
mkdir -p ../orderers/orderer.org4.com
../../bin/fabric-ca-client enroll -H $PWD/../orderers/orderer.org4.com -u http://orderer.org4.com:ordererpw@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org4.com
```

## Make MSP directory

Make MSP structure & Copy proper certificates: 

```bash
# Make directories
cd ..
mkdir -p msp/{admincerts,cacerts}
# Copy certificates
cp ca/ca.org4.com-cert.pem msp/cacerts/
cp users/Admin@org4.com/msp/signcerts/cert.pem msp/admincerts/Admin@org4.com-cert.pem
# Copy it to Orderer
cp -R msp/admincerts orderer/msp/
```

## Create a network configuration(NC4)

Write network configuration file:

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
    V1_3: true
    V1_2: false
    V1_1: false
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
    V1_1: true
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
        Rule: "MAJORITY Admins"
    Capabilities:
      V1_3: true
    Application:
      <<: *ApplicationDefaults
    Orderer:
      <<: *Orderer
      Organizations:
        - *Org4
    Consortiums:
      HowToConsortium:
        Organizations:
EOF
```

Download Hyperledger Fabric tools:

```bash
curl https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.4.0/hyperledger-fabric-linux-amd64-1.4.0.tar.gz | tar -xz
```

Generate a genesis block with NC4:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc1 -channelID syschannel -outputBlock ./genesis.block
```

## Run orderer(O4)

```bash
docker run -d --name orderer --hostname orderer.org4.com \
        --network howto_network \
        -v $PWD/genesis.block:/var/hyperledger/fabric/genesis.block \
        -v $PWD/org4.com/orderer/msp:/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LISTENADDRESS=0.0.0.0 \
        -e ORDERER_GENERAL_LOGLEVEL=debug \
        -e ORDERER_GENERAL_GENESISMETHOD=file \
        -e ORDERER_GENERAL_GENESISFILE=/var/hyperledger/fabric/genesis.block \
        -e ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LOCALMSPID=Org4 \
        hyperledger/fabric-orderer:1.4.0 \
        orderer
```