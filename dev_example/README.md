Table of Contents
=================

   * [Development deployment](#development-deployment)
      * [Before starting](#before-starting)
         * [Pre-requisites](#pre-requisites)
      * [Creating](#creating)
         * [Crypto material](#crypto-material)
         * [Fabric Orderer](#fabric-orderer)
         * [Fabric Peer](#fabric-peer)
      * [Deleting](#deleting)

# Development deployment

## Before starting

### Pre-requisites

Before running this tutorial you will need:

1) A Kubernetes (K8S) cluster with at least 1 node (you can get free credits to deploy a managed K8S cluster on AWS, GCP, Azure, etc)
2) Helm (and Tiller) installed on K8S

## Creating

### Crypto Material

#### Cryptogen install

You may need to download the latest cryptogen tools

If you are using a Mac, you can run the following (version 1.1.0 and 1.2.0 are also supported if needed):

    brew tap aidtechnology/homebrew-fabric
    brew install fabric-tools@1.3.0

#### Generation

Generate the necessary crypto-materials with

    cd ./dev_example/config
    cryptogen generate --config ./crypto-config.yaml

You can see what was generated using the command `tree` (install with `brew install tree` if it's missing):

    tree crypto-config

#### Admin materials

If you don't have the following namespaces, you may need to create.

    kubectl create ns orderers
    kubectl create ns peers

We must save the relevant Orderer admin crypto-config files as secrets.

    MSP_DIR=./crypto-config/ordererOrganizations/orderers.svc.cluster.local/users/Admin@orderers.svc.cluster.local/msp

    ORG_CERT=$(ls ${MSP_DIR}/admincerts/*.pem)
    kubectl create secret generic -n orderers hlf--ord-admincert --from-file=cert.pem=$ORG_CERT

    CA_CERT=$(ls ${MSP_DIR}/cacerts/*.pem)
    kubectl create secret generic -n orderers hlf--ord-cacert --from-file=cacert.pem=$CA_CERT

And the Peer admin crypto-config files:

    MSP_DIR=./crypto-config/peerOrganizations/peers.svc.cluster.local/users/Admin@peers.svc.cluster.local/msp

    ORG_CERT=$(ls ${MSP_DIR}/admincerts/*.pem)
    kubectl create secret generic -n peers hlf--peer-admincert --from-file=cert.pem=$ORG_CERT

    ORG_KEY=$(ls ${MSP_DIR}/keystore/*_sk)
    kubectl create secret generic -n peers hlf--peer-adminkey --from-file=key.pem=$ORG_KEY

    CA_CERT=$(ls ${MSP_DIR}/cacerts/*.pem)
    kubectl create secret generic -n peers hlf--peer-cacert --from-file=cacert.pem=$CA_CERT

#### Genesis & Channel

    configtxgen -profile OrdererGenesis -outputBlock ./genesis.block

    configtxgen -profile MyChannel -channelID mychannel -outputCreateChannelTx ./mychannel.tx

    kubectl create secret generic -n orderers hlf--genesis --from-file=genesis.block

    kubectl create secret generic -n peers hlf--channel --from-file=mychannel.tx

### Fabric Orderer

#### Crypto material

Save node identity cryptographic material as secrets:

    MSP_DIR=./crypto-config/ordererOrganizations/orderers.svc.cluster.local/orderers/ord1-hlf-ord.orderers.svc.cluster.local/msp

    NODE_CERT=$(ls ${MSP_DIR}/signcerts/*.pem)
    kubectl create secret generic -n orderers hlf--ord1-idcert --from-file=cert.pem=$NODE_CERT

    NODE_KEY=$(ls ${MSP_DIR}/keystore/*_sk)
    kubectl create secret generic -n orderers hlf--ord1-idkey --from-file=key.pem=$NODE_KEY

#### Helm chart

And install the HLF Orderer Helm chart:

    helm install stable/hlf-ord -n ord1 --namespace orderers -f ../helm_values/ord1.yaml

And check that it is running:

    ORD_POD=$(kubectl get pods -n orderers -l "app=hlf-ord,release=ord1" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs -n orderers $ORD_POD | grep 'Starting orderer'

### Fabric Peer

#### CouchDB

We start by installing the CouchDB database:

    helm install stable/hlf-couchdb -n cdb-peer1 --namespace peers -f ../helm_values/cdb-peer1.yaml

And check that it is running:

    CDB_POD=$(kubectl get pods -n peers -l "app=hlf-couchdb,release=cdb-peer1" -o jsonpath="{.items[*].metadata.name}")

    kubectl logs -n peers $CDB_POD | grep 'Apache CouchDB has started on'

#### Crypto material

Save node identity cryptographic material as secrets:

    MSP_DIR=./crypto-config/peerOrganizations/peers.svc.cluster.local/peers/peer1-hlf-peer.peers.svc.cluster.local/msp

    NODE_CERT=$(ls ${MSP_DIR}/signcerts/*.pem)
    kubectl create secret generic -n peers hlf--peer1-idcert --from-file=cert.pem=$NODE_CERT

    NODE_KEY=$(ls ${MSP_DIR}/keystore/*_sk)
    kubectl create secret generic -n peers hlf--peer1-idkey --from-file=key.pem=$NODE_KEY

#### Helm chart

And install the HLF-Peer Helm Chart:

    helm install stable/hlf-peer -n peer1 --namespace peers -f ../helm_values/peer1.yaml

And check that it is running:

    PEER_POD=$(kubectl get pods -n peers -l "app=hlf-peer,release=peer1" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs -n peers $PEER_POD | grep 'Starting peer'

#### Channel

Create the channel

    kubectl exec -n peers $PEER_POD -- peer channel create -o ord1-hlf-ord.orderers.svc.cluster.local:7050 -c mychannel -f /hl_config/channel/mychannel.tx

Fetch the channel and join it from the orderer:

    kubectl exec -n peers $PEER_POD -- peer channel fetch config /var/hyperledger/mychannel.block -c mychannel -o ord1-hlf-ord.orderers.svc.cluster.local:7050

    kubectl exec -n peers $PEER_POD -- bash -c 'CORE_PEER_MSPCONFIGPATH=$ADMIN_MSP_PATH peer channel join -b /var/hyperledger/mychannel.block'

You should see that now the peer is linked to this channel.

    kubectl exec -n peers $PEER_POD -- peer channel list

## Deleting

We start by deleting the actual Helm deployments:

    helm delete --purge ord1 peer1 cdb-peer1

Then we delete the crypto-material we saved for the orderers:

    kubectl delete secret -n orderers hlf--genesis hlf--ord-admincert hlf--ord-cacert hlf--ord1-idcert hlf--ord1-idkey

And that of the peers:

    kubectl delete secret -n peers hlf--channel hlf--peer-admincert hlf--peer-adminkey hlf--peer-cacert hlf--peer1-idcert hlf--peer1-idkey

Delete crypto material files:

    rm -rf ./config/*MSP ./config/genesis.block ./config/mychannel.tx

Clean up namespaces we used for the development examples

    kubectl delete ns orderers peers
