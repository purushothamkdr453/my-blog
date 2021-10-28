---
layout: post
title:  "Installing Istio Servicemesh"
date:   2021-09-18 12:41:25 +0530
category: servicemesh
---

- [Servicemesh](#servicemesh)
   - [Overview](#overview)
   - [Architecture](#architecture)
   - [Pre-requisites](#pre-requisites)
   - [Setup](#setup)
       - [step-1: Installing istio service mesh](#step-1-installing-istio-service-mesh)
       - [step-2: Labeling the namespace](#step-2-labeling-the-namespace)
       - [step-3: testing](#step-3-testing)
    

## Servicemesh

## Overview

`Servicemesh` is a dedicated infrastructure layer for facilitating communications between services or microservices using proxy.

Servicemesh provides the following benefits

- observability
- traffic management
- security

Some of the popular servicemesh solutions are

- istio
- linkerd
- consul 
- aws app mesh
- traefikmesh

`istio` is more popular compared to others, so we will be installing `istio`.

## Architecture

![alt text](/assets/images/istio-architecture.png)

## Pre-requisites

`helm` & `kubernetes cluster` are pre-requisites so install them prior proceeding with other steps.

to install `helm` refer to [this](https://helm.sh/docs/intro/install/)

you can install any kubernetes cluster i.e `managed kubernetes` or `kubeadm` or `minikube`. 

## Setup

checking the status of minkube. Execute the below command.

minikube status

![alt text](/assets/images/minikube-status.png)

kubectl get nodes

![alt text](/assets/images/kubectl-getnodes.png)

### step-1: Installing istio service mesh

There are multiple ways to install the istio. in this article we will see how to install istio using `istioctl`. 

**Download istioctl**

Execute the below commands

```
curl -L https://istio.io/downloadIstio | ISTIO_VERSION=1.11.3 TARGET_ARCH=x86_64 sh -
cd istio-1.11.3
export PATH=$PWD/bin:$PATH
```

After running the above commands `istioctl` binary is available for you. you can verify it by executing the command `istioctl --help`

Now lets install istio using istioctl command

```
istioctl install --set profile=demo -y
```

![alt text](/assets/images/istio-install.png)

**Note** the above command creates new namespace `istio-system` and deploys istio components inside that namespace. you can verify that by executing the below commands.

kubectl get ns
kubectl get all -n istio-system

three pods will be installed inside `istio-system`

istiod pod -> core component of istio
istio-ingressgateway pod -> handles incoming requests of servicemesh
istio-egressgateway pod -> handles requests going out of servicemesh

services are also installed for respective pods, you can check in the output of `kubectl get all -n istio-system`

## step-2: Labeling the namespace

for demonstration purpose lets create a new namespace called `learning`

```
kubectl create ns learning
```

lets label this namespace with `istio-injection`

```
kubectl label namespace learning istio-injection=enabled
````

you can verify the labels of namespace by executing below command.

```
kubectl get ns learning --show-labels
```

## step-3: testing

lets test the functionality of istio by deploying a sample pod inside `learning` namespace.

```
kubectl run webapp --image nginx -n learning
```

if you run `kubectl get pods -n learning` namespace you should see a pod up & running. if you closely observe this pod contains 2 containers which can be noticed under `READY` column.

**How did we get 2 containers inside pod**

we have labeled the `learning` namespace with istio-injection so if we create any pod inside this namespace then automatically istio core component(istiod running under istio-system) will inject a new sidecar container to pod so you will get 2 containers inside the pod.

```
kubectl describe pod webapp -n learning
```

nginx - main container
istio-proxy - sidecar proxy container

this `sidecar proxy` container will intercept the traffic between 2 microservices. so if we configure any rules at proxy level then microservices interaction will be enabled or disabled based on these proxy rule/configs. you can observe this behavior from the architecture diagram above.


