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
