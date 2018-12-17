Table of Contents
=================

   * [Production deployment](#production-deployment)
      * [Before starting](#before-starting)
         * [Important note](#important-note)
         * [Pre-requisites](#pre-requisites)
         * [Customisation](#customisation)
      * [Creating](#creating)
         * [Fabric CA](#fabric-ca)
         * [Genesis and Channel](#genesis-and-channel)
         * [Kafka for Ordering service](#kafka-for-ordering-service)
         * [Fabric Orderer](#fabric-orderer)
         * [Fabric Peer](#fabric-peer)
      * [Deleting](#deleting)

# Production deployment

## Before starting

### Important note

While we call this a `production` deployment, this is not *quite* a production deployment because in production:

1. you will have more peers
2. you will have more orderers
3. you will have multiple organisations

However, this is a good starting point for a production deployment.

### Pre-requisites

Before running this tutorial you will need:

1) A Kubernetes (K8S) cluster with at least 4 nodes (you can get free credits to deploy a managed K8S cluster on AWS, GCP, Azure, etc)
2) Helm (and Tiller) installed on K8S
3) An `nginx-ingress` installation (using the Helm chart)
4) A `cert-manager` installation (using the Helm chart)
5) A domain name for your components (e.g. the Certificate Authority), connected to your `nginx-ingress` IP address - you can obtain one for free or $1.00 at many Domain Name Registrars.

#### NGINX Ingress controller

You can install the ingress controller by running this command:

    helm install stable/nginx-ingress -n nginx-ingress --namespace ingress-controller

#### Certificate manager

You can install the certificate manager, to ensure you can auto-generate the TLS certificates

    helm install stable/cert-manager -n cert-manager --namespace cert-manager

Then we need to add the Staging and Production cluster issuers

    kubectl create -f ./extra/certManagerCI_staging.yaml

    kubectl create -f ./extra/certManagerCI_production.yaml

### Customisation

#### Domain Name

Currently, the `helm_values` files for the CA reference the following CA Domain Name: `ca.hgf.aidtech-test.xyz` in  `/helm_values/ca.yaml`

Since you won't have access to this, you should set this domain name to one you've obtained/purchased, and which is pointing to the `nginx-ingress` IP address.

Alternatively, you may not use the Ingress at all and disable it, and instead use the CA through port-forwarding from the Kubernetes cluster to your local machine. For this you will need to adapt the instructions provided to your own use-case.

## Creating

### K8S namespaces

Create the required namespaces:

    kubectl create ns cas orderers peers

### Fabric CA

#### Installing

Install the Fabric CA chart (it automatically creates a postgresql database)

    helm install stable/hlf-ca -n ca --namespace cas -f ./helm_values/ca.yaml

Get pod for CA release

    CA_POD=$(kubectl get pods -n cas -l "app=hlf-ca,release=ca" -o jsonpath="{.items[0].metadata.name}")

Check if server is ready

    kubectl logs -n cas $CA_POD | grep "Listening on"

#### Fabric CA Identity

Check that we don't have a certificate

    kubectl exec -n cas $CA_POD -- cat /var/hyperledger/fabric-ca/msp/signcerts/cert.pem

    kubectl exec -n cas $CA_POD -- bash -c 'fabric-ca-client enroll -d -u http://$CA_ADMIN:$CA_PASSWORD@$SERVICE_DNS:7054'

Check that ingress works correctly

    CA_INGRESS=$(kubectl get ingress -n cas -l "app=hlf-ca,release=ca" -o jsonpath="{.items[0].spec.rules[0].host}")

    curl https://$CA_INGRESS/cainfo

#### Org Admin Identities

##### Register

###### Orderer Organisation

Get identity of ord-admin (this should not exist at first)

    kubectl exec -n cas $CA_POD -- fabric-ca-client identity list --id ord-admin

Register Orderer Admin if the previous command did not work

    kubectl exec -n cas $CA_POD -- fabric-ca-client register --id.name ord-admin --id.secret OrdAdm1nPW --id.attrs 'admin=true:ecert'

###### Peer Organisation

Get identity of ord-admin (this should not exist at first)

    kubectl exec -n cas $CA_POD -- fabric-ca-client identity list --id peer-admin

Register Peer Admin if the previous command did not work

    kubectl exec -n cas $CA_POD -- fabric-ca-client register --id.name peer-admin --id.secret PeerAdm1nPW --id.attrs 'admin=true:ecert'

##### Enroll

###### Orderer Organisation

Enroll the Organisation Admin identity (typically we would use a more secure password than `OrdAdm1nPW`, etc.)

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client enroll -u https://ord-admin:OrdAdm1nPW@$CA_INGRESS -M ./OrdererMSP

Copy the signcerts to admincerts

    mkdir -p ./config/OrdererMSP/admincerts

    cp ./config/OrdererMSP/signcerts/* ./config/OrdererMSP/admincerts


###### Peer Organisation

Enroll the Organisation Admin identity (typically we would use a more secure password than `PeerAdm1nPW`, etc.)

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client enroll -u https://peer-admin:PeerAdm1nPW@$CA_INGRESS -M ./PeerMSP

Copy the signcerts to admincerts

    mkdir -p ./config/PeerMSP/admincerts

    cp ./config/PeerMSP/signcerts/* ./config/PeerMSP/admincerts

##### Save Crypto Material

###### Orderer Organisation

Create a secret to hold the admin certificate:

    ORG_CERT=$(ls ./config/OrdererMSP/admincerts/cert.pem)

    kubectl create secret generic -n orderers hlf--ord-admincert --from-file=cert.pem=$ORG_CERT

Create a secret to hold the admin key:

    ORG_KEY=$(ls ./config/OrdererMSP/keystore/*_sk)

    kubectl create secret generic -n orderers hlf--ord-adminkey --from-file=key.pem=$ORG_KEY

Create a secret to hold the admin key CA certificate:

    CA_CERT=$(ls ./config/OrdererMSP/cacerts/*.pem)

    kubectl create secret generic -n orderers hlf--ord-ca-cert --from-file=cacert.pem=$CA_CERT

###### Peer Organisation

Create a secret to hold the admincert:

    ORG_CERT=$(ls ./config/PeerMSP/admincerts/cert.pem)

    kubectl create secret generic -n peers hlf--peer-admincert --from-file=cert.pem=$ORG_CERT

Create a secret to hold the admin key:

    ORG_KEY=$(ls ./config/PeerMSP/keystore/*_sk)

    kubectl create secret generic -n peers hlf--peer-adminkey --from-file=key.pem=$ORG_KEY

Create a secret to hold the CA certificate:

    CA_CERT=$(ls ./config/PeerMSP/cacerts/*.pem)

    kubectl create secret generic -n peers hlf--peer-ca-cert --from-file=cacert.pem=$CA_CERT

### Genesis and channel

    cd ./config

Create Genesis block and Channel

    configtxgen -profile OrdererGenesis -outputBlock ./genesis.block

    configtxgen -profile MyChannel -channelID mychannel -outputCreateChannelTx ./mychannel.tx

Save them as secrets

    kubectl create secret generic -n orderers hlf--genesis --from-file=genesis.block

    kubectl create secret generic -n peers hlf--channel --from-file=mychannel.tx

Get back to where we were before...

    cd ..

### Kafka for Ordering service

Install Kafka chart (use special values to ensure 4 Kafka brokers and that Kafka messages don't disappear)

    helm install incubator/kafka -n kafka-hlf --namespace orderers -f ./helm_values/kafka-hlf.yaml

### Fabric Orderer

For each orderer set the `NUM` environmental variable and follow the below instructions (in this example, either 1 or 2):

    export NUM=1

#### Crypto material

Register orderer with CA (typically we would use a more secure password than `ord1_pw`, etc.)

    kubectl exec -n cas $CA_POD -- fabric-ca-client register --id.name ord${NUM} --id.secret ord${NUM}_pw --id.type orderer

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client enroll -d -u https://ord${NUM}:ord${NUM}_pw@$CA_INGRESS -M ord${NUM}_MSP

Save the Orderer certificate in a secret

    NODE_CERT=$(ls ./config/ord${NUM}_MSP/signcerts/*.pem)

    kubectl create secret generic -n orderers hlf--ord${NUM}-idcert --from-file=cert.pem=${NODE_CERT}

Save the Orderer private key in another secret

    NODE_KEY=$(ls ./config/ord${NUM}_MSP/keystore/*_sk)

    kubectl create secret generic -n orderers hlf--ord${NUM}-idkey --from-file=key.pem=${NODE_KEY}

#### Helm charts

Install orderers

    helm install stable/hlf-ord -n ord${NUM} --namespace orderers -f ./helm_values/ord${NUM}.yaml

Get logs from orderer to check it's actually started

    ORD_POD=$(kubectl get pods -n orderers -l "app=hlf-ord,release=ord${NUM}" -o jsonpath="{.items[0].metadata.name}")

    kubectl logs -n orderers $ORD_POD | grep 'completeInitialization'

> Repeat all above steps for Orderer 2, etc.

### Fabric Peer

For each peer set the `NUM` environmental variable and follow the below instructions (in this example, either 1 or 2):

    export NUM=1

#### CouchDB Helm Chart

Install CouchDB chart

    helm install stable/hlf-couchdb -n cdb-peer${NUM} --namespace peers -f ./helm_values/cdb-peer${NUM}.yaml

Check that CouchDB is running

    CDB_POD=$(kubectl get pods -n peers -l "app=hlf-couchdb,release=cdb-peer${NUM}" -o jsonpath="{.items[*].metadata.name}")

    kubectl logs -n peers $CDB_POD | grep 'Apache CouchDB has started on'

#### Crypto material

Register orderer with CA (typically we would use a more secure password than `peer1_pw`, etc.)

    kubectl exec -n cas $CA_POD -- fabric-ca-client register --id.name peer${NUM} --id.secret peer${NUM}_pw --id.type peer

    FABRIC_CA_CLIENT_HOME=./config fabric-ca-client enroll -d -u https://peer${NUM}:peer${NUM}_pw@$CA_INGRESS -M peer${NUM}_MSP

Save the Peer certificate in a secret

    NODE_CERT=$(ls ./config/peer${NUM}_MSP/signcerts/*.pem)

    kubectl create secret generic -n peers hlf--peer${NUM}-idcert --from-file=cert.pem=${NODE_CERT}

Save the Peer private key in another secret

    NODE_KEY=$(ls ./config/peer${NUM}_MSP/keystore/*_sk)

    kubectl create secret generic -n peers hlf--peer${NUM}-idkey --from-file=key.pem=${NODE_KEY}

#### Peer Helm Chart

Install Peer

    helm install stable/hlf-peer -n peer${NUM} --namespace peers -f ./helm_values/peer${NUM}.yaml

Get Peer pod:

    PEER_POD=$(kubectl get pods -n peers -l "app=hlf-peer,release=peer${NUM}" -o jsonpath="{.items[0].metadata.name}")

And check that Peer is running

    kubectl logs -n peers $PEER_POD | grep 'Starting peer'

> Repeat all above steps for Peer 2, etc.

#### Channels

Create channel (do this only once in Peer 1)

    kubectl exec -n peers $PEER_POD -- peer channel create -o ord1-hlf-ord.orderers.svc.cluster.local:7050 -c mychannel -f /hl_config/channel/mychannel.tx

Fetch and join channel

    kubectl exec -n peers $PEER_POD -- peer channel fetch config /var/hyperledger/mychannel.block -c mychannel -o ord1-hlf-ord.orderers.svc.cluster.local:7050

    kubectl exec -n peers $PEER_POD -- bash -c 'CORE_PEER_MSPCONFIGPATH=$ADMIN_MSP_PATH peer channel join -b /var/hyperledger/mychannel.block'

> Repeat above 2 commands (`fetch` & `join`) for Peer 2, etc.

Check which channels the peer has joined:

    kubectl exec $PEER_POD -n peers -- peer channel list

## Deleting

Delete helm deployments

    helm delete --purge ca kafka-hlf ord1 ord2 cdb-peer1 peer1 cdb-peer2 peer2

Delete stateful sets (in case Helm does not fully delete them)

    kubectl delete statefulset -n orderers kafka-log kafka-hlf-zookeeper kafka-hlf

Delete Persistent Volume Claims

    kubectl delete pvc -n cas data-ca-postgresql-0

    kubectl delete pvc -n orderers data-kafka-hlf-zookeeper-0 data-kafka-hlf-zookeeper-1 data-kafka-hlf-zookeeper-2 datadir-kafka-hlf-0 datadir-kafka-hlf-1 datadir-kafka-hlf-2 datadir-kafka-hlf-3

Delete secrets on K8S:

    kubectl delete secret -n orderers hlf--ord-admincert hlf--ord-adminkey hlf--ord-ca-cert hlf--genesis hlf--ord1-idcert hlf--ord2-idcert hlf--ord1-idkey hlf--ord2-idkey

    kubectl delete secret -n peers hlf--peer-admincert  hlf--peer-adminkey hlf--peer-ca-cert hlf--channel hlf--peer1-idcert hlf--peer2-idcert hlf--peer1-idkey hlf--peer2-idkey

Delete crypto material files:

    rm -rf ./config/*MSP ./config/genesis.block ./config/mychannel.tx

Clean up namespaces we used for the production examples

    kubectl delete ns cas orderers peers
