---
layout: post
title:  "Installing Istio Servicemesh using istioctl - Part 1"
date:   2021-10-28 04:00:00 +0530
category: servicemesh
---

- [Servicemesh](#servicemesh)
   - [Overview](#overview)
   - [Architecture](#architecture)
   - [Pre-requisites](#pre-requisites)
   - [Setup](#setup)
       - [step-1: Installing istio service mesh](#step-1-installing-istio-service-mesh)
       - [step-2: Labeling the namespace](#step-2-labeling-the-namespace)
       - [step-3: deploying sample book info application](#step-3-deploying-sample-book-info-application)
    

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

Execute the below command to start minikube.

```
minikube start --cpus 4 --memory 8192 --addons=ingress
```

status of minikube can be verified by using the below command.

```
minikube status
```

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

```
# This command displays all the components that are deployed
istioctl verify-install
kubectl get ns
kubectl get all -n istio-system
```

three pods will be installed inside `istio-system`

istiod pod -> core component of istio <br/>
istio-ingressgateway pod -> handles incoming requests of servicemesh <br/>
istio-egressgateway pod -> handles requests going out of servicemesh <br/>

services are also installed for respective pods, you can check in the output of `kubectl get all -n istio-system`

## step-2: Labeling the namespace

for demonstration purpose lets create a new namespace called `learning`

```
kubectl create ns learning
kubectl label namespace learning istio-injection=enabled
````

## step-3: deploying sample book info application

lets deploy the sample `bookinfo` application inside `learning` namespace. Execute the below command.

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n learning
```

Output of the above command should be as follows.

![alt text](/assets/images/istio-bookinfo-deployment.png)

if you run `kubectl get pods -n learning` namespace you should see a pod up & running. if you closely observe pod contains 2 containers which can be noticed under `READY` column.

**How did we get 2 containers inside pod**

we have labeled the `learning` namespace with istio-injection so if we create any pod inside this namespace then automatically istio core component(istiod running under istio-system) will inject a new sidecar container(istio-proxy) to pod so you will get 2 containers inside the pod.

istio-proxy - sidecar proxy container

this `sidecar proxy` container will intercept the traffic between 2 microservices. so if we configure any rules at proxy level then microservices interaction will be enabled or disabled based on these proxy rule/configs. you can observe this behavior from the architecture diagram above.

Lets try to access `productpage` endpoint from `ratings` pod. Execute the below command.

```
kubectl -n learning exec $(kubectl get pods -n learning | grep -v -i 'name' | grep -i 'ratings' | awk '{print $1}') -c ratings -- curl -sS http://productpage:9080/productpage
```

As you can see we are successfully able to access the 'productpage' url from ratings pod. however we will not be able to access this endpoint outside the pod/cluster. To makethe application accessible from outside create `ingress gateway` which maps a path to a route at the edge of your mesh.

```
kubectl -n learning apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

Since we can not get a loadbalancer ip in minikube. let us execute the port-forward on `istiod` pod which resides in `istio-system` namespace. execute the below command.

```
kubectl -n istio-system port-forward $(kubectl get pods -n istio-system | grep -v 'NAME' | grep 'istio-ingressgateway' | awk '{print $1}') 8080:8080
```

**Note** The above command will be executed in foreground. let it run and dont kill it. For all the subsequent operations & commmands, open new terminal and navigate to the istio location wherever you have downloaded i.e (cd istio-1.11.3 & export PATH=$PWD/bin:$PATH).

Now let us go to the browser and try to access the endpoint i.e http://localhost:8080/productpage. Screenshot below for reference.

![alt text](/assets/images/product-page-output.png)