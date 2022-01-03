---
layout: post
title:  "Istio observavility"
date:   2021-12-19 06:00:00 +0530
category: servicemesh
---

- [Observability](#Observability)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Deploying kiali Prometheus Grafana jaeger](#deploying-kiali-prometheus-grafana-jaeger)
   - [Accessing the kiali dashboard](#accessing-the-kiali-dashboard)


## Observability

## Overview

Istio can be integrated with several telemetry applications such as `Kiali`, `prometheus`, `zipkin`, `jaeger`, `Grafana` etc. These can help you gain an understanding of the structure of your service mesh, display the topology of the mesh, and analyze the health of your mesh.

## Pre-requisites

Makesure you have followed all the instructions outlined in [Part-1](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/10/27/Installing-istio-servicemesh-using-istioctl.html).

## Deploying kiali Prometheus Grafana jaeger

Dont touch the port-forward command which was executed( mentioned in part-1). Open new terminal and navigate to the path where istioctl package was downloaed. Execute the below commads.

```
cd istio-1.11.3
export PATH=$PWD/bin:$PATH
#-- installing addons
kubectl apply -f ./samples/addons/
```

Last command install `prometheus`, `Grafana`, `kiali`, `jaeger`. These components are installed in `istio-system` namespace.

```
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

**Note** it may take a while for these pods to come into running state. so please wait.

Sample screenshots are shown below.

**observability pods**

output of `kubectl get pods -n istio-system`

![alt text](/assets/images/observability-pods.png)

**observability services**

output of `kubectl get svc -n istio-system`

![alt text](/assets/images/observability-services.png)

## Accessing the kiali dashboard

istioctl dashboard kiali

**Note** as soon as you execute the above command, kiali dashboard gets opened in the browser. again the above command will be run in foreground so dont kill that command. If you want to execute something please open a new termainl and follow the same instructions(cd istio-1.11.3 & export PATH=$PWD/bin:$PATH).


Go to Kiali dashbord -> Select `Graph` -> Under Namespace dropdown select `istio-system` & `learning`. Right now you will not see any graph because there is no traffic. Sample screenshot shown below.

![alt text](/assets/images/kiali-dashboard-graph.png)

Lets put someload on the app. Go the browser and refresh the page i.e http://localhost:8080/productpage several times. After a while you should see graph in the kiali dashboard. Sample screenshot below.

![alt text](/assets/images/kiali-dashboard-graph-with-traffic.png)

So in this way `Kiali` helps us in understanding the topoloy of the mesh i.e which service is interacting with which service and also helps us in pinpoint the issue where exactly it is failing.

Instead of hitting the browser several time to put/apply load. you can also do the same using command. for example you can use the below command.

```
for i in $(seq 1 100); do curl -o /dev/null -sS http://localhost:8080/productpage; done
```
Just like kiali you can also access other components such sas

**Grafana**

istioctl dashboard grafana

Istio components related dashboards are accessible at http://localhost:3000/dashboards

**Prometheus**

istioctl dashboard prometheus

Promethues dashboard available at http://localhost:9090

**Jaeger**

istioctl dashboard jaeger