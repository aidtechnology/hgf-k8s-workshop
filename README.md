Table of Contents
=================

   * [HGF K8S Workshop](#hgf-k8s-workshop)
      * [Workshop flow](#workshop-flow)
         * [Cluster creation](#cluster-creation)
         * [Development example](#development-example)
         * [Production example](#production-example)
         * [Cleanup](#cleanup)
      * [Extra resources](#extra-resources)
         * [Repositories](#repositories)
         * [Courses](#courses)
         * [FAQ](#faq)


# HGF K8S Workshop

Hyperledger Global Forum workshop on deploying Hyperledger Fabric on Kubernetes in development and production.

You may wish to explore https://github.com/aidtechnology/nephos, which helps automate the deployment of similar examples as presented here.

## Workshop flow

### Cluster creation

In the workshop we demonstrate how to create a managed K8S cluster on Azure:

    export GROUP=hgf-workshop
    export LOCATION=westeurope

    az group create -n $GROUP -l $LOCATION
    az aks create -g $GROUP -n ${GROUP}-aks -s Standard_DS2_v2 --kubernetes-version 1.11.5 --node-count 5
    az aks get-credentials -g $GROUP -n ${GROUP}-aks

Then you can install Helm, using

    kubectl create -f ./helm-rbac.yaml

    helm init --service-account tiller

Finally, add the `incubator` repository, so you are able to install Kafka, etc.

    helm repo add incubator https://kubernetes-charts-incubator.storage.googleapis.com/

    helm repo update

### Development example

We will start with the `dev_example`, using Cryptogen to set up the identities and cryptographic material.

This is sufficient for Development purposes and will use a very simple setup of 1 peer and 1 (solo) orderer.

### Production example

In the second part of the workshop, `prod_example`, we will be using the Fabric CA to provide the identities and cryptographic material.

This uses as production-ready setup implementing Certificate Authorities persisting identities to PostgreSQL, multiple peers and multiple (Kafka-consensus) orderers.

### Cleanup

If you use Azure AKS, you can just delete the resource group and associated AKS cluster in one fell swoop.

    az group delete -n $GROUP

## Extra resources

### Repositories

Our charts can be found at the official Helm Charts repository:

https://github.com/helm/charts

And also on our own open-source repository:

https://github.com/aidtechnology/at-charts

We also have a repository hosting the Fabric CA client Homebrew installer (for OS X):

https://github.com/aidtechnology/homebrew-fabric-ca

### Courses

*Blockchain for Business - An Introduction to Hyperledger Technologies*, where we have contributed the Hyperledger Composer chapter:

https://www.edx.org/course/blockchain-business-introduction-linuxfoundationx-lfs171x-0

*Blockchain for Blockchain Applications* on Packt and Udemy:

https://www.packtpub.com/application-development/hyperledger-blockchain-applications-video

https://www.udemy.com/hyperledger-for-blockchain-applications/

### FAQ

> In progress
