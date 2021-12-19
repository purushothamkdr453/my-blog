---
layout: post
title:  "istio trafficmanagement - Part 2"
date:   2021-12-19 05:10:00 +0530
category: servicemesh
---

- [trafficmanagement](#trafficmanagement)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Application](#application)
   - [Weighted Traffic Deployment](#weighted-traffic-deployment)
     

## trafficmanagement

## Overview

As mentioned in the previous blog, lets try to apply weighted traffic to ratings service.

## Pre-requisites

Makesure you have followed all the instructions outlined in [previous blog](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/12/18/Istio-traffic-management-part-1.html).

## Application

![alt text](/assets/images/bookinfo-application.png)

## Weighted Traffic Deployment

Dont touch the port-forward command which was executed( mentioned in previous blog). Open new terminal and navigate to the path where istioctl package was downloaed. Execute the below commads.

```
cd istio-1.11.3
export PATH=$PWD/bin:$PATH
```

Create 2 files `reviews-virtualservice.yaml` & `reviews-destinationrules.yaml`
 
copy the below content to `reviews-virtualservice.yaml`

```
apiVersion: networking.istio.io/v1beta1
kind: VirtualService
metadata:
  name: reviews-v2-v3
spec:
  hosts:
  - reviews
  http:
  - route:
    - destination:
        host: reviews
        subset: v2
      weight: 99
    - destination:
        host: reviews
        subset: v3
      weight: 1
```

copy the below content to `reviews-destinationrules.yaml`

```
apiVersion: networking.istio.io/v1alpha3
kind: DestinationRule
metadata:
  name: dest-v2-v3
spec:
  host: reviews
  subsets:
  - name: v2
    labels:
      app: reviews
      version: v2
  - name: v3
    labels:
      app: reviews
      version: v3     
```

```
kubectl -n learning apply -f reviews-virtualservice.yaml
kubectl -n learning apply -f reviews-destinationrules.yaml
```

Now try to access the page at http://localhost:8080/productpage. If you refresh the page now couple of times now you should notice that most of traffic is going to black star ratings and less traffic goes to red star ratings.