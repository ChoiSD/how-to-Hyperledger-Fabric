# Creating a channel for a consortium

A channel is a primary communications mechanism by which members of a consortium can communicate with each other.
There can be many channels as you want, but let's start with one.
After this guide, you will get a network like this:
![network pic](https://hyperledger-fabric.readthedocs.io/en/release-1.3/_images/network.diagram.4.png "Target network - 04")

C1: a channel for R1 & R2

## Create a configuration block for a channel (C1)

Write a configuration file:

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
          - *Org1NC4
      X1:
        Organizations:
          - *Org1X1
          - *Org2
  HowToDoc4:
    Consortium: X1
    Application:
      <<: *Applications
      Organizations:
        - *Org1X1
        - *Org2
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

Generate a channel(C1) configuration block:

* Hyperledger Fabric accepts following regex `[a-z][a-z0-9.-]*` only.
* So I will going to use `c1` as a channel name.

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc4 -channelID c1 -outputCreateChannelTx ./channel-artifacts/C1.tx
```

Define a anchor peer for R1:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc4 -channelID c1 -outputAnchorPeersUpdate ./channel-artifacts/C1R1anchors.tx -asOrg Org1X1
```

Define a anchor peer for R2:

```bash
bin/configtxgen -configPath $PWD -profile HowToDoc4 -channelID c1 -outputAnchorPeersUpdate ./channel-artifacts/C1R2anchors.tx -asOrg Org2
```

And it's done. The above blocks we've created will be used after starting peers on this network.