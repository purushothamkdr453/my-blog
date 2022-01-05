---
layout: post
title:  "Istio Security part 2 - Authorization"
date:   2021-12-19 07:10:00 +0530
category: servicemesh
---

- [Security](#Security)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Application](#application)
   - [Restricting access to bookinfo app from frontend namespace](#restricting-access-to-bookinfo-app-from-frontend-namespace)
   - [Testing](#testing)


## Security

## Overview

In this blog we will look into authorization part of istio security. Istio Authorization Policy enables access control on workloads in the mesh.

## Pre-requisites

Makesure you have the followed all the instructions specified in [previous blog](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/12/19/istio-security-part-1-authentication.html).

## Application

![alt text](/assets/images/bookinfo-application.png)

## Restricting access to bookinfo app from frontend namespace

lets restrict the access to bookinfo only from `frontend namespace pods`.

create a file named `restict-to-bookinfo-from-frontend.yaml` with below content.

```
apiVersion: security.istio.io/v1beta1
kind: AuthorizationPolicy
metadata:
 name: restirct-access-from-frontend 
 namespace: learning
spec:
 selector:
   matchLabels:
     app: productpage
 action: ALLOW
 rules:
 - from:
   - source:
       namespaces: [ "frontend" ]
   to:
   - operation:
       methods: ["GET"]
```

lets apply the file.

```
kubectl apply -f restict-to-bookinfo-from-frontend.yaml
```

lets verify the authorization policy created using below command.

```
kubectl get authorizationpolicy -n learning
```

lets create namespace `authtest`.

```
kubectl create ns authtest
```

create a file named `frontend.yaml` with below content.

```
---
apiVersion: v1
kind: ServiceAccount
metadata:
  name: frontend
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  labels:
    tier: frontend
spec:
  replicas: 1
  selector:
    matchLabels:
      tier: frontend
  template:
    metadata:
      labels:
        tier: frontend
    spec:
      serviceAccountName: frontend
      containers:
      - name: frontend
        image: purushothamkdr453/frontend:v1
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: 250m
---
apiVersion: v1
kind: Service
metadata:
  name: frontend
spec:
  selector:
    tier: frontend
  ports:
    - protocol: TCP
      port: 80
      targetPort: 80
```

lets deploy the above application to `authtest` namespace.

```
kubectl -n authtest apply -f <(istioctl kube-inject -f frontend.yaml)
```

you can verify the deployed application using below command.

```
kubectl get pods -n authtest
```

## Testing

lets try to access the `book info app` from `authtest` namespaces pod.

**From authtest namespace app**

```
kubectl -n authtest exec $(kubectl get pods -n authtest | grep -v 'NAME' | awk '{print $1}') -c frontend -- curl -sS http://productpage.learning:9080/productpage
```

the above command throws an error i.e `RBAC: access denied`. This is expected because it voilates the condition specified in the above authorization policy.

**From learning namespace app**

```
kubectl -n frontend exec $(kubectl get pods -n frontend | grep -v 'NAME' | awk '{print $1}') -c frontend -- curl -sS http://productpage.learning:9080/productpage
```

The above command provides successful response. as it satisfies the condition specified in authorization policy that we created earlier.

