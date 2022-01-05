---
layout: post
title:  "Istio Security Authorization - Part 2"
date:   2021-12-19 07:10:00 +0530
category: servicemesh
---

- [Security](#Security)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Creating test namespace](#creating-test-namespace)
   - [Deploying httpbin to test namespace](#deploying-httpbin-to-test-namespace)
   - [Peer authentication policy](#peer-authentication-policy)


## Security

## Overview

In this blog we will look into authorization part of istio security.

## Pre-requisites

Makesure you have followed all the instructions outlined in [istio setup](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/12/18/Installing-istio-servicemesh-using-istioctl.html) and [traffic management part-1](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/12/18/Istio-traffic-management-part-1.html).

Dont touch the port-forward command which was executed( mentioned in part-1). Open new terminal and navigate to the path where istioctl package was downloaed. Execute the below commads.

```
cd istio-1.11.3
export PATH=$PWD/bin:$PATH
```

## Creating test namespace

create a new namespace i.e `test` and enable istio-injection.

```
kubectl create ns test
kubectl label ns test istio-injection=enabled
```

## Deploying httpbin to test namespace

deploying nginx to test namespace. execute the below command.

```
kubectl -n test apply -f samples/httpbin/httpbin.yaml
```

Lets try to access the productpage application from nginx pod. Execute the below command. when you execute the below command you will get a successful response.

```
kubectl -n test exec -it $(kubectl -n test get pods | grep -vi 'name' | grep 'httpbin' | awk '{print $1}') -c istio-proxy -- curl http://productpage.learning:9080
```

## Peer authentication policy

lets create a file named  `strict-mtls-to-bookinfo.yaml` with a below content. This policy applies to all the workloads in `learning` namespace.

```
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls-to-bookinfo
  namespace: learning
spec:
  mtls:
    mode: STRICT
```

lets apply the policy.

```
kubectl apply -f strict-mtls-to-bookinfo.yaml
```

Verifying the created peer authentication policy.

```
kubectl get peerauthentication -n learning
```

Now if you try to access the bookinfo application from nginx pod, you will get an error.

```
kubectl -n test exec -it $(kubectl -n test get pods | grep -vi 'name' | grep 'httpbin' | awk '{print $1}') -c istio-proxy  -- curl http://productpage.learning:9080
```

Error you will get is as follows.

![alt text](/assets/images/istio-security-authetication-mtls-error.png)

Lets try to fix this

create a file named `strict-mtls-to-nginx.yaml` with the below content.

```
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls-to-nginx
  namespace: test
spec:
  selector:
    matchLabels:
      app: frontend
  mtls:
    mode: STRICT
```

```
kubectl apply -f strict-mtls-to-nginx.yaml
```

Lets try to fix this

create a file named `enable-acccess-to-httpbin.yaml` with the below content.

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: enable-acccess-to-httpbin
 namespace: learning
spec:
 action: ALLOW
 rules:
 - from:
   - source:
       principals: ["cluster.local/ns/test/sa/frontend"]
```

lets apply the pollicy.

```
kubectl apply -f enabling-mtls-on-httpbin.yaml
```

lets verify the created peerauthentication policy.

```
kubectl get peerauthentication -n test
```


