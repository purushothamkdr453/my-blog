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
       - kubernetes cluster
       - helm
   - [Setup](#setup)
       - [step-1: Installing istio service mesh](#step-1-installing-istio-service-mesh)
    

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

![alt text](../assets/images/istio-architecture.png)

## Pre-requisites

`helm` & `kubernetes cluster` are pre-requisites so install them prior proceeding with other steps.

to install `helm` refer to [this](https://helm.sh/docs/intro/install/)

you can install any kubernetes cluster i.e `managed kubernetes` or `kubeadm` or `minikube`. 

## Setup

checking the status of minkube. Execute the below command.

minikube status

![alt text](../assets/images/minikube-status.png)

kubectl get nodes

![alt text](../assets/images/kubectl-getnodes.png)

### step-1: Installing istio service mesh


