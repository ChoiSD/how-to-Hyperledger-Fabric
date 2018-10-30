# Adding network administrator
Currently administrators of R4 has administrative rights over the network.
We are going to allow administrators of other organization(R1) to administer the network.
The goal is depicted as:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.2.1.png "Target network - 02")

R1 : org1.com

## Create a self-signed CA certificate

Locate yourself in a path where `openssl.cnf` file exists.
*I assume that you have just done [01-Create-Network.md](https://github.com/ChoiSD/how-to-Hyperledger-Fabric/blob/master/Docs/Build-From-Scratch/01-Create-Network.md)*

Generate a private key and a self-signed certificate for R1:
```
# Create MSP folders
mkdir -p org1.com/ca org1.com/users; cd org1.com/ca
# Generate serial number
SERIAL=$(xxd -l 16 -p /dev/random)
# Generate ecdsa private key
openssl ecparam -genkey -name prime256v1 -out private.key
# Generate a self-signed CA certificate
openssl req -new -x509 -set_serial 0x${SERIAL} -key private.key -out ca.org1.com-cert.pem -days 3650 -config ../../openssl.cnf
```

As there are some pre-configured values in `openssl.cnf`, we need to do put some values in this time like below:
```
Country Name (2 letter code) [KR]:
State or Province Name (full name) [Seoul]:
Locality Name (eg, city) [Gangdong-gu]:
Organization Name (eg, company) [org4.com]:org1.com
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) [ca.org4.com]:ca.org1.com
Email Address []:
```

Transfrom private key into pkcs8 format:
```
openssl pkcs8 -topk8 -nocrypt -in private.key -out signcert-key.pem
rm -f private.key
mv signcert-key.pem $(openssl x509 -noout -pubkey -in ca.org1.com-cert.pem | openssl asn1parse -strparse 23 -in - | openssl dgst -sha256 | awk '{print $2}')_sk
chmod 600 *_sk
```

## Run Fabric CA server(CA1)

Run PostgreSQL server:
```
docker volume create ca_org1_data
docker run -d --name postgres_org1 \
        --network howto_network \
        -v ca_org1_data:/var/lib/postgresql/data \
        -e POSTGRES_USER=caorg1 \
        -e POSTGRES_PASSWORD=secretfororg1 \
        -e POSTGRES_DB=fabric \
        postgres:9.5
```

Run Fabric CA server:
```
PRIVATE=$(ls *_sk)
PUBLIC=$(ls *.pem)
docker run -d --name ca0_org1 --hostname ca0.org1.com \
        --network howto_network \
        -v $PWD/$PRIVATE:/etc/hyperledger/fabric-ca-server/private.key \
        -v $PWD/$PUBLIC:/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_CERTFILE=/etc/hyperledger/fabric-ca-server/public.pem \
        -e FABRIC_CA_SERVER_CA_KEYFILE=/etc/hyperledger/fabric-ca-server/private.key \
        -e FABRIC_CA_SERVER_DB_TYPE=postgres \
        -e FABRIC_CA_SERVER_DB_DATASOURCE="host=postgres_org1 port=5432 user=caorg1 password=secretfororg1 dbname=fabric sslmode=disable" \
        hyperledger/fabric-ca:1.3.0 \
        fabric-ca-server start -b admin:admin4org1
```

## Get admin's certificate

Get admin user's certificate from CA server:
```
cd ../users
mkdir admin
# Get IP address of CA server
IP=$(docker inspect ca0_org1 -f '{{.NetworkSettings.Networks.howto_network.IPAddress}}')
# Get certificate
export FABRIC_CA_CLIENT_HOME=$PWD/admin
../../bin/fabric-ca-client enroll -u http://admin:admin4org1@${IP}:7054 --csr.cn admin --csr.names C=KR,ST=Seoul,L=Gangdong-gu,O=org1.com
# Create missing directories
mkdir admin/msp/admincerts
cp admin/msp/signcerts/cert.pem admin/msp/admincerts/
```

## Allow member of other organization(R1) to administer a network

Write a update network configuration file:
```
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
        - Name: Org1
          ID: Org1MSP
          MSPDir: org1.com/users/admin/msp
          Policies:
            Readers:
              Type: Signature
              Rule: "OR('Org1MSP.member')"
            Writers:
              Type: Signature
              Rule: "OR('Org1MSP.member')"
            Admins:
              Type: Signature
              Rule: "OR('Org1MSP.admin')"
    Consortiums:
      SampleConsortium:
        Organizations:
EOF
```

Generate an update genesis block with updated NC4:
```
export FABRIC_CFG_PATH=$PWD
bin/configtxgen -profile SampleSoloGenesis -channelID syschannel -outputBlock ./genesis.block.updated
```

## Run orderer(O4) with updated configuration

```
# Stop existing orderer
docker rm -f orderer
# Run orderer
docker run -d --name orderer --hostname orderer.org4.com \
        --network howto_network \
        -v $PWD/genesis.block.updated:/var/hyperledger/fabric/genesis.block \
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