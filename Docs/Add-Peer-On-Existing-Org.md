# Adding one peer on an existing organization

In this lab, you will add one additional peer to an existing organization. This lab will leverage CloudFlare's PKI toolkit, [cfssl](https://github.com/cloudflare/cfssl), to generate TLS certificates to secure communication between other peer nodes.

After completing this lab, you should have one additional peer node:

* peer2.org2.example.com

> I am not sure this is the right way. But it works!!

## Assumption

- Start this lab on the same path where byfn.sh resides on
- Fabric network is already created and operates

## Install cfssl
This lab requires the `cfssl` and `cfssljson` binaries. Download them from the [cfssl repository](https://pkg.cfssl.org).

### OS X

```
wget https://pkg.cfssl.org/R1.2/cfssl_darwin-amd64
chmod +x cfssl_darwin-amd64
sudo mv cfssl_darwin-amd64 /usr/local/bin/cfssl
```

```
wget https://pkg.cfssl.org/R1.2/cfssljson_darwin-amd64
chmod +x cfssljson_darwin-amd64
sudo mv cfssljson_darwin-amd64 /usr/local/bin/cfssljson
```

### Linux

```
wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
chmod +x cfssl_linux-amd64
sudo mv cfssl_linux-amd64 /usr/local/bin/cfssl
```

```
wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
chmod +x cfssljson_linux-amd64
sudo mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
```

## Set up a CA configuration

Create a CA configuration file:
```
mkdir pki; cd pki
cat > ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "17520h"
        },
        "profiles": {
            "signcert": {
                "expiry": "17520h",
                "usages": [
                    "signing"
                ]
            },
            "tls": {
                "expiry": "17520h",
                "usages": [
                    "signing",
                    "key encipherment",
                    "server auth",
                    "client auth"
                ]
            }
        }
    }
}
EOF
```

## Create TLS & Signcert certificates

Create a TLS certificate signing request:

```
mkdir tls; tls
cat > peer2-tls.json <<EOF
{
    "CN": "peer2.org2.example.com",
    "hosts": [
        "peer2.org2.example.com",
        "peer2"
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "US",
            "ST": "California",
            "L": "San Francisco"
        }
    ]
}
EOF
```

Generate a TLS certificate and a private key:
```
cfssl gencert \
  -ca ../../crypto-config/peerOrganizations/org2.example.com/tlsca/tlsca.org2.example.com-cert.pem \
  -ca-key ../../crypto-config/peerOrganizations/org2.example.com/tlsca/*_sk \
  -config ../ca-config.json \
  -profile=tls peer2-tls.json | cfssljson -bare tls
```

Create a Signcert certificate signing request:
```
mkdir ../signcert; cd ../signcert
cat > peer2-signcert.json <<EOF
{
    "CN": "peer2.org2.example.com",
    "hosts": [
      "peer2.org2.example.com",
      "peer2"
    ],
    "key": {
        "algo": "ecdsa",
        "size": 256
    },
    "names": [
        {
            "C": "US",
            "ST": "California",
            "L": "San Francisco"
        }
    ]
}
EOF
```

Generate a Signcert certificate and a private key:
```
cfssl gencert \
  -ca ../../crypto-config/peerOrganizations/org2.example.com/ca/ca.org2.example.com-cert.pem \
  -ca-key ../../crypto-config/peerOrganizations/org2.example.com/ca/*_sk \
  -config ../ca-config.json \
  -profile=signcert peer2-signcert.json | cfssljson -bare signcert
```

## Move generated certificates and private keys to proper locations

Create a new path for peer2.org2.example.com:
```
cp -R crypto-config/peerOrganizations/org2.example.com/peers/peer1.org2.example.com/ \
  crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/
```

Move a TLS certificate and private key to proper location:
```
cd ../tls
mv tls.pem \
  ../../crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls/server.crt
chmod u+x tls-key.pem
mv tls-key.pem \
  ../../crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls/server.key
```

Move a Signcert certificate and private key to proper location:
```
cd ../signcert
chmod u+x signcert-key.pem
mv signcert-key.pem $(openssl x509 -in signcert.pem -noout -text |grep -A1 'Subject Key' | tail -1 | sed 's/://g;s/\ //g' | tr '[:upper:]' '[:lower:]')_sk
rm ../../crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/msp/keystore/*_sk
mv signcert.pem \
  ../../crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/msp/signcerts/peer2.org2.example.com-cert.pem
rm ../../crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/msp/keystore/*_sk
mv *_sk \
  ../../crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/msp/keystore/
```

## Run a peer2.org2 node

Create a compose file for peer2.org2:
```
cd ../
cat > docker-compose-peer2-org2.yaml <<EOF
version: '2'

networks:
  byfn:

services:

  peer2.org2.example.com:
    container_name: peer2.org2.example.com
    extends:
      file: base/peer-base.yaml
      service: peer-base
    environment:
      - CORE_PEER_ID=peer2.org2.example.com
      - CORE_PEER_ADDRESS=peer2.org2.example.com:7051
      - CORE_PEER_GOSSIP_EXTERNALENDPOINT=peer2.org2.example.com:7051
      - CORE_PEER_GOSSIP_BOOTSTRAP=peer2.org2.example.com:7051
      - CORE_PEER_LOCALMSPID=Org2MSP
    volumes:
        - /var/run/:/host/var/run/
        - ./crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/msp:/etc/hyperledger/fabric/msp
        - ./crypto-config/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls:/etc/hyperledger/fabric/tls
    ports:
      - 11051:7051
      - 11053:7053
    networks:
      - byfn
EOF
```

Start peer2.org2:
```
docker-compose -f docker-compose-peer2-org2.yaml up -d
```

## Execute a chaincode on peer2.org2:

Exeucte cli container's shell:
```
docker exec -it cli bash
```

On the shell, run following commands:
```
CORE_PEER_TLS_ROOTCERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls/ca.crt
CORE_PEER_TLS_KEY_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls/server.key
CORE_PEER_LOCALMSPID=Org2MSP
CORE_PEER_TLS_CERT_FILE=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/peers/peer2.org2.example.com/tls/server.crt
CORE_PEER_MSPCONFIGPATH=/opt/gopath/src/github.com/hyperledger/fabric/peer/crypto/peerOrganizations/org2.example.com/users/Admin\@org2.example.com/msp
CORE_PEER_ADDRESS=peer2.org2.example.com:7051

CHANNEL_NAME=<channel name you use>
peer channel join -b ${CHANNEL_NAME}.block
peer chaincode install -n mycc -v 1.0 -p github.com/hyperledger/fabric/examples/chaincode/go/chaincode_example02
peer chaincode query -C $CHANNEL_NAME -n mycc -c '{"Args":["query","a"]}'
```
