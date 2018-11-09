# Creating a channel for a consortium

A channel is a primary communications mechanism by which members of a consortium can communicate with each other.
There can be many channels as you want, but let's start with one.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.4.png "Target network - 04")

C1: a channel for R1 & R2

## Create a configuration block for a channel (C1)

Write a configuration file:

```bash
cd ../..
cat > configtx.yaml <<EOF
Profiles:
  C1Channel:
    Consortium: SampleConsortium
    Application:
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
      Organizations:
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
          AnchorPeers:
            - Host: peer0.org1.com
              Port: 7051
        - Name: Org2
          ID: Org2MSP
          MSPDir: org2.com/users/admin/msp
          Policies:
            Readers:
              Type: Signature
              Rule: "OR('Org2MSP.member')"
            Writers:
              Type: Signature
              Rule: "OR('Org2MSP.member')"
            Admins:
              Type: Signature
              Rule: "OR('Org2MSP.admin')"
          AnchorPeers:
            - Host: peer0.org2.com
              Port: 7051
      Capabilities:
        V1_3: true
        V1_2: false
        V1_1: false
EOF
```

Generate a channel(C1) configuration block:

```bash
export FABRIC_CFG_PATH=$PWD
bin/configtxgen -profile C1Channel -channelID C1 -outputCreateChannelTx C1.tx
```

Define a anchor peer for R1:

```bash
bin/configtxgen -profile C1Channel -channelID C1 -outputAnchorPeersUpdate C1R1anchors.tx -asOrg Org1
```

Define a anchor peer for R2:

```bash
bin/configtxgen -profile C1Channel -channelID C1 -outputAnchorPeersUpdate C1R2anshors.tx -asOrg Org2
```

And it's done. The above blocks we've created will be used after starting peers on this network.