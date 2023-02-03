---
layout: post
title:  "Argocd Installation"
date:   2023-02-03 04:00:00 +0530
category: argocd
tags: argocd kubernetes devops gitops cd
---

- [argocd](#argocd)
   - [Overview](#overview)
   - [pre-requisites](#pre-requisites)
   - [Installation](#installation)
       - [Using Kubectl](#using-kubectl)
       - [Using kustomize](#using-kustomize)

## argocd

## Overview

`argocd` is a declarative & continuous delivery tool for kubernetes. Argocd follows `Gitops principles`. Following are the Gitops principles.

## pre-requisites

- kubernetes cluster

## Installation

Argocd can be installed in many ways i.e using kustomize or kubectl. We will discuss about all types of installation methods below.

### Using kubectl


```
kubectl create namespace argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/v2.5.10/manifests/ha/install.yaml
```

wait until all the pods move into running state. you can verify it by executing the below command.

```
kubectl get pods -n argocd
```

Execute the port-forward command on `argocd-server` to access the argocd web gui.

```
kubectl -n argocd port-forward svc/argocd-server 8080:443
```

Now try to access the web gui at - https://localhost:8080

![alt text](/assets/images/argocd-webgui.png)

Username is `admin`
<br/>
password - initial password is stored in secret which can be accessed using below command.

```
kubectl get secret argocd-initial-admin-secret -o json -n argocd | jq -r .data.password | base64 -d
```

Enter username and password in the login page. After successful login, Home page of argocd looks as follows.

![alt text](/assets/images/argocd-homepage.png)


### Using kustomize

Makesure you have `kustomize` installed. if you dont have kustomize then refer the [documentation](https://kustomize.io/).
create a file named `kustomization.yaml` with the following content.

```
apiVersion: kustomize.config.k8s.io/v1beta1
kind: Kustomization

namespace: argocd
resources:
- https://raw.githubusercontent.com/argoproj/argo-cd/v2.5.10/manifests/ha/install.yaml
```

execute the following command to install argocd.

```
kubectl create ns argocd
kustomize build . | kubectl -n argocd apply -f -
```

wait until all the pods move into running state. Then follow the same steps mentioned previously to access the argocd webgui.

