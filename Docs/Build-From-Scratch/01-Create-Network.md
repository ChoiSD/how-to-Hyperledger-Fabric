# Creating blockchain network

In this guide, we will create a basis of a blockchain network, Ordering service.
A goal of this guide can be conceptually represented by a below picture:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.2.png "Target network - 01")

R4 : org4.com

## Create a self-signed CA certificate

Create a openssl configuration file:

```bash
cat > openssl.cnf <<EOF
dir                                     = .

[ ca ]
default_ca                              = CA_default

[ CA_default ]
default_md                              = md5
preserve                                = no
email_in_dn                             = no
nameopt                                 = default_ca
certopt                                 = default_ca
policy                                  = policy_match

[ policy_match ]
countryName                             = match
stateOrProvinceName                     = match
organizationName                        = match
organizationalUnitName                  = optional
commonName                              = supplied
emailAddress                            = optional

[ req ]
default_md                                = sha256
distinguished_name                      = req_distinguished_name
req_extensions                          = v3_req
x509_extensions                         = v3_ca

[ req_distinguished_name ]
countryName                     = Country Name (2 letter code)
countryName_default             = KR
countryName_min                 = 2
countryName_max                 = 2
stateOrProvinceName             = State or Province Name (full name)
stateOrProvinceName_default     = Seoul
localityName                    = Locality Name (eg, city)
localityName_default            = Gangdong-gu
0.organizationName              = Organization Name (eg, company)
0.organizationName_default      = org4.com
organizationalUnitName          = Organizational Unit Name (eg, section)
#organizationalUnitName_default =
commonName                      = Common Name (eg, your name or your server\'s hostname)
commonName_default              = ca.org4.com
commonName_max                  = 64
emailAddress                    = Email Address
emailAddress_max                = 64


[ v3_ca ]
keyUsage                                = critical, digitalSignature, keyEncipherment, keyCertSign, cRLSign
extendedKeyUsage                        = anyExtendedKeyUsage
basicConstraints                        = CA:TRUE
subjectKeyIdentifier                    = hash

[ v3_req ]
basicConstraints                        = CA:FALSE
subjectKeyIdentifier                    = hash
EOF
```

Generate a private key file and a self-signed certificate using OpenSSL:

```bash
# Create local MSP folders
mkdir -p org4.com/users org4.com/ca; cd org4.com/ca
# Generate serial number
SERIAL=$(xxd -l 16 -p /dev/random)
# Generate ecdsa private key
openssl ecparam -genkey -name prime256v1 -out private.key
# Generate a self-signed CA certificate
openssl req -new -x509 -set_serial 0x${SERIAL} -key private.key -out ca.org4.com-cert.pem -days 3650 -config ../../openssl.cnf
# Transform private key into pkcs8 format
openssl pkcs8 -topk8 -nocrypt -in private.key -out signcert-key.pem
rm -f private.key
mv signcert-key.pem $(openssl x509 -noout -pubkey -in ca.org4.com-cert.pem | openssl asn1parse -strparse 23 -in - | openssl dgst -sha256 | awk '{print $2}')_sk
chmod 600 *_sk
```

## Run Fabric CA server(CA4)

We are going to issue admin and orderer certificate thru Fabric CA server.

Run Fabric CA server:

```bash
PRIVATE=$(ls *_sk)
PUBLIC=$(ls *.pem)
docker run -d --name ca0_org4 --hostname ca0.org4.com \
        --network howto_network \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        hyperledger/fabric-ca:1.3.0 \
        fabric-ca-server start -b admin:admin4org4
```

## Get admin's certificate

Download CA client binary:

```bash
cd ../users
curl https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric-ca/hyperledger-fabric-ca/linux-amd64-1.3.0/hyperledger-fabric-ca-linux-amd64-1.3.0.tar.gz | tar -xz -C ../../
```

Get admin user's certificate from CA server:

```bash
mkdir admin
# Get IP address of CA server
IP=$(docker inspect ca0_org4 -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
export FABRIC_CA_CLIENT_HOME=$PWD/admin
../../bin/fabric-ca-client enroll -u http://admin:admin4org4@${IP}:7054 --csr.cn admin --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org4.com
```

## Get orderer's certificate

Register orderer:

```bash
../../bin/fabric-ca-client register --id.name orderer --id.type orderer --id.maxenrollments 1 --id.secret ordererpw
```

Enroll orderer:

```bash
mkdir orderer
export FABRIC_CA_CLIENT_HOME=$PWD/orderer
../../bin/fabric-ca-client enroll -u http://orderer.ordererpw@${IP}:7054 --csr.cn orderer.org4.com --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org4.com
```

Copy Admin's certificate:

```bash
mkdir orderer/msp/admincerts
cp admin/msp/signcerts/cert.pem orderer/msp/admincerts/
```

## Create a network configuration(NC4)

Write network configuration file:

```bash
cd ../..
cat > configtx.yaml <<EOF
Profiles:
  SampleSoloGenesis:
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
          Rule: "MAJORITY Admins"
        BlockValidation:
          Type: ImplicitMeta
          Rule: "ANY Writers"
      Capabilities:
        V1_1: true
      Organizations:
        - Name: Org4
          ID: Org4MSP
          MSPDir: org4.com/users/orderer/msp
          Policies:
            Readers:
              Type: Signature
              Rule: "OR('Org4MSP.member')"
            Writers:
              Type: Signature
              Rule: "OR('Org4MSP.member')"
            Admins:
              Type: Signature
              Rule: "OR('Org4MSP.admin')"
    Consortiums:
      SampleConsortium:
        Organizations:
EOF
```

Download Hyperledger Fabric tools:

```bash
curl https://nexus.hyperledger.org/content/repositories/releases/org/hyperledger/fabric/hyperledger-fabric/linux-amd64-1.3.0/hyperledger-fabric-linux-amd64-1.3.0.tar.gz | tar -xz
```

Generate a genesis block with NC4:

```bash
export FABRIC_CFG_PATH=$PWD
bin/configtxgen -profile SampleSoloGenesis -channelID syschannel -outputBlock ./genesis.block
```

## Run orderer(O4)

```bash
docker run -d --name orderer --hostname orderer.org4.com \
        --network howto_network \
        -v $PWD/genesis.block:/var/hyperledger/fabric/genesis.block \
        -v $PWD/org4.com/users/orderer/msp:/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LISTENADDRESS=0.0.0.0 \
        -e ORDERER_GENERAL_LOGLEVEL=debug \
        -e ORDERER_GENERAL_GENESISMETHOD=file \
        -e ORDERER_GENERAL_GENESISFILE=/var/hyperledger/fabric/genesis.block \
        -e ORDERER_GENERAL_LOCALMSPDIR=/var/hyperledger/fabric/msp \
        -e ORDERER_GENERAL_LOCALMSPID=Org4MSP \
        hyperledger/fabric-orderer:1.3.0 \
        orderer
```