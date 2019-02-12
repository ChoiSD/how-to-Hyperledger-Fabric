# Defining a consortium

Now we have a base network.
Let's create a consortium which will make use of this network.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.3.png "Target network - 03")

R2 : org2.com

## Create a self-signed CA certificate for R2

Locate yourself in a path where `openssl.cnf` file exists.
*I assume that you have just done [02-Add-Network-Admin.md](https://github.com/ChoiSD/how-to-Hyperledger-Fabric/blob/master/Docs/Build-From-Scratch/02-Add-Network-Admin.md)*

Generate a private key and a self-signed certificate for R2:

```bash
# Create MSP folders
mkdir -p org2.com/{ca,users}; cd org2.com/ca
# Generate EC paramter with the group 'prime256v1'
openssl ecparam -out param.out -name prime256v1
# Generate Self-signed CA certificate
openssl req -newkey ec:param.out -nodes -keyout private.key -x509 -days 3650 -out ca.org2.com-cert.pem -extensions v3_user -config ../../openssl.cnf
# Rename private key
mv private.key $(openssl x509 -noout -pubkey -in ca.org2.com-cert.pem | openssl asn1parse -strparse 23 -in - | openssl dgst -sha256 | awk '{print $2}')_sk
rm param.out
```

As there are some pre-configured values in `openssl.cnf`, we need to do put some values in this time like below:

```bash
Country Name (2 letter code) [KR]:
Organization Name (eg, company) [org4.com]:org2.com
Organizational Unit Name (eg, section) [Blockchain Biz.]:
Common Name (eg, your name or your servers hostname) [ca.org4.com]:ca.org2.com
```

## Run Fabric CA server(CA2)

Run Fabric CA server:

```bash
PRIVATE=$(ls *_sk)
PUBLIC=$(ls *.pem)
docker run -d --name ca0_org2 --hostname ca0.org2.com \
        --network howto_network \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        hyperledger/fabric-ca:1.4.0 \
        fabric-ca-server start -b admin:adminpwOrg2
```

## Get admin's certificate

Get admin user's certificate from CA server:

```bash
cd ../users
mkdir admin
# Get IP address of CA server
IP=$(docker inspect ca0_org2 -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
export FABRIC_CA_CLIENT_HOME=$PWD/admin
../../bin/fabric-ca-client enroll -u http://admin:adminpwOrg2@${IP}:7054 --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org2.com
```

Create MSP sturcture & Copy proper certificates:

```bash
cd ..
mkdir -p msp/{admincerts,tlscacerts}
cp -R users/admin/msp/cacerts msp/
cp /users/admin/msp/signcerts/cert.pem msp/adminverts/Admin@org2.com-cert.pem
```

## Create a consortium(X1) to a network

Write a update network configuration file:

```bash
cd ../..
cat > configtx.yaml <<EOF
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
      Capabilities:
        V1_1: true
      Organizations:
        - Name: Org4
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
        - Name: Org1
          ID: Org1MSP
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
    Consortiums:
      HowToConsortium:
        Organizations:
          - Name: Org1
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
          - Name: Org2
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
EOF
```

Generate an updated genesis block:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc3 -channelID syschannel -outputBlock ./genesis.block
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