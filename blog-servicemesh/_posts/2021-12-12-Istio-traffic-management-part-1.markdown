---
layout: post
title:  "istio trafficmanagement - Part 1"
date:   2021-12-19 05:00:00 +0530
category: servicemesh
---

- [trafficmanagement](#trafficmanagement)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Application](#application)
   - [Deploying Gateway and Virtual service](#deploying-gateway-and-virtual-service)
     

## trafficmanagement

## Overview

Istio’s traffic routing rules let you easily control the flow of traffic and API calls between services. Istio simplifies configuration of service-level properties like circuit breakers, timeouts, and retries, and makes it easy to set up important tasks like A/B testing, canary rollouts, and staged rollouts with percentage-based traffic splits. It also provides out-of-box reliability features that help make your application more resilient against failures of dependent services or the network.

Istio’s traffic management model relies on the Envoy proxies that are deployed along with your services. All traffic that your mesh services send and receive (data plane traffic) is proxied through Envoy, making it easy to direct and control traffic around your mesh without making any changes to your services.

## Pre-requisites

Makesure you have followed all the instructions outlined in [Part-1](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/10/27/Installing-istio-servicemesh-using-istioctl.html).

## Application

![alt text](/assets/images/bookinfo-application.png)

## Deploying Gateway and Virtual service

In previous blog we tried to access the app at http://localhost:8080/productpage however it was failing. lets fix it.

Dont touch the port-forward command which was executed( mentioned in previous blog). Open new terminal and navigate to the path where istioctl package was downloaed. Execute the below commads.

```
cd istio-1.11.3
export PATH=$PWD/bin:$PATH
```

Now lets deploy Gateway & Virtual Service. Why do we need to deploy these custom crd components. Well the reason is below.

Gateway -> Entry point to the mesh
Virtualservice -> Once the request enters into the mesh, virtual service determines to which other service the request should be forwarded to.

if you dont deploy these 2 components then your request will not be entered into the mesh itself. This is why we were not able to access the app at "http://localhost:8080/productpage" in previous blog. 

```
kubectl -n learning apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

```
# to view the created gateway
kubectl get gateway -n learning
# to view the created virtual service
kubectl get virtualservice -n learning
```

Now try to access the app at http://localhost:8080/productpage. Boom its working now. screenshot below for the reference.

![alt text](/assets/images/product-page-output.png)

If you notice for every refresh of the page, you see a different rating i.e ratings in black color, ratings in red color & no ratings and it is round robin balanced. This is default behavior in kubernetes. 

Is it possible to apply weighted traffic i.e 99% percent traffic(black ratings) & 1% percent traffic(red ratings)? Yes it is possible. To checkout this scenario refer to [part-2 section of traffic management]().