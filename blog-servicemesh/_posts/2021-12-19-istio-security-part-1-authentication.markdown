---
layout: post
title:  "Istio Security Part 1 - Authentication"
date:   2021-12-19 07:00:00 +0530
category: servicemesh
---

- [Security](#Security)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Installing istio](#installing-istio)
   - [Creating namespaces](#creating-namespaces)
   - [Deploying bookinfo application](#deploying-bookinfo-application)
   - [Deploying Sample application](#deploying-sample-application)
   - [Deploying Sample application to test namespace with out sidecar](#deploying-sample-application-to-test-namespace-with-out-sidecar)
   - [Testing the bookinfo app from test and frontend apps](#testing-the-bookinfo-app-from-test-and-frontend-apps)
   - [Restricting strict mtls mode to bookinfo app](#restricting-strict-mtls-mode-to-bookinfo-app)
   - [Checking whether traffic is encrypted or not](#checking-whether-traffic-is-encrypted-or-not)


## Security

## Overview

In this blog we will look into authentication part of istio security. By default istio sidecars are configured to accept both plain text & encrypted traffic. However the best practice is to restrict to accept only encrypted traffic. We can acheive this by using peer authenticaion policy. Details instructions are below.


## Pre-requisites

Makesure you have k8s cluster. if you want to setup k8s cluster using minikube refer to [this](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/12/18/Installing-istio-servicemesh-using-istioctl.html#setup).

if you have followed my other blogs then probably you would be already having istio components installed. Makesure you delete them. we need to delete it because demo profile which we used is not helpful for this scenario. We need to tweak it a little bit. Execute the below commands(only if you have already installed istio components & other namespaces).

```
kubectl delete ns istio-system
kubectl delete ns learning
kubectl delete ns test
```

## Installing istio

create a file named `demo-profile.yaml` with the below content.

```
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  components:
    base:
      enabled: true
    cni:
      enabled: false
    egressGateways:
    - enabled: true
      k8s:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
      name: istio-egressgateway
    ingressGateways:
    - enabled: true
      k8s:
        resources:
          requests:
            cpu: 10m
            memory: 40Mi
        service:
          ports:
          - name: status-port
            port: 15021
            targetPort: 15021
          - name: http2
            port: 80
            targetPort: 8080
          - name: https
            port: 443
            targetPort: 8443
          - name: tcp
            port: 31400
            targetPort: 31400
          - name: tls
            port: 15443
            targetPort: 15443
      name: istio-ingressgateway
    istiodRemote:
      enabled: false
    pilot:
      enabled: true
      k8s:
        env:
        - name: PILOT_TRACE_SAMPLING
          value: "100"
        resources:
          requests:
            cpu: 10m
            memory: 100Mi
  hub: docker.io/istio
  meshConfig:
    accessLogFile: /dev/stdout
    defaultConfig:
      proxyMetadata: {}
    enablePrometheusMerge: true
    outboundTrafficPolicy:
      mode: REGISTRY_ONLY
  profile: demo
  tag: 1.11.0
  values:
    base:
      enableCRDTemplates: false
      validationURL: ""
    gateways:
      istio-egressgateway:
        autoscaleEnabled: false
        env: {}
        name: istio-egressgateway
        secretVolumes:
        - mountPath: /etc/istio/egressgateway-certs
          name: egressgateway-certs
          secretName: istio-egressgateway-certs
        - mountPath: /etc/istio/egressgateway-ca-certs
          name: egressgateway-ca-certs
          secretName: istio-egressgateway-ca-certs
        type: ClusterIP
        zvpn: {}
      istio-ingressgateway:
        autoscaleEnabled: false
        env: {}
        name: istio-ingressgateway
        secretVolumes:
        - mountPath: /etc/istio/ingressgateway-certs
          name: ingressgateway-certs
          secretName: istio-ingressgateway-certs
        - mountPath: /etc/istio/ingressgateway-ca-certs
          name: ingressgateway-ca-certs
          secretName: istio-ingressgateway-ca-certs
        type: LoadBalancer
        zvpn: {}
    global:
      configValidation: true
      defaultNodeSelector: {}
      defaultPodDisruptionBudget:
        enabled: true
      defaultResources:
        requests:
          cpu: 10m
      imagePullPolicy: ""
      imagePullSecrets: []
      istioNamespace: istio-system
      istiod:
        enableAnalysis: false
      jwtPolicy: third-party-jwt
      logAsJson: false
      logging:
        level: default:info
      meshNetworks: {}
      mountMtlsCerts: false
      multiCluster:
        clusterName: ""
        enabled: false
      network: ""
      omitSidecarInjectorConfigMap: false
      oneNamespace: false
      operatorManageWebhooks: false
      pilotCertProvider: istiod
      priorityClassName: ""
      proxy:
        autoInject: enabled
        clusterDomain: cluster.local
        componentLogLevel: misc:error
        enableCoreDump: false
        excludeIPRanges: ""
        excludeInboundPorts: ""
        excludeOutboundPorts: ""
        image: proxyv2
        includeIPRanges: '*'
        logLevel: warning
        privileged: true
        readinessFailureThreshold: 30
        readinessInitialDelaySeconds: 1
        readinessPeriodSeconds: 2
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 10m
            memory: 40Mi
        statusPort: 15020
        tracer: zipkin
      proxy_init:
        image: proxyv2
        resources:
          limits:
            cpu: 2000m
            memory: 1024Mi
          requests:
            cpu: 10m
            memory: 10Mi
      sds:
        token:
          aud: istio-ca
      sts:
        servicePort: 0
      tracer:
        datadog: {}
        lightstep: {}
        stackdriver: {}
        zipkin: {}
      useMCP: false
    istiodRemote:
      injectionURL: ""
    pilot:
      autoscaleEnabled: false
      autoscaleMax: 5
      autoscaleMin: 1
      configMap: true
      cpu:
        targetAverageUtilization: 80
      enableProtocolSniffingForInbound: true
      enableProtocolSniffingForOutbound: true
      env: {}
      image: pilot
      keepaliveMaxServerConnectionAge: 30m
      nodeSelector: {}
      replicaCount: 1
      traceSampling: 1
    telemetry:
      enabled: true
      v2:
        enabled: true
        metadataExchange:
          wasmEnabled: false
        prometheus:
          enabled: true
          wasmEnabled: false
        stackdriver:
          configOverride: {}
          enabled: false
          logging: false
          monitoring: false
          topology: false
```

lets install the istio using the above configuration.

```
istioctl install -f demo-profile.yaml -y
```

you can verify the installed istio components using below commands.

```
kubectl get pods -n istio-system
kubectl get svc -n istio-system
```

## Creating namespaces

Lets create 3 namespaces i.e `learning` & `frontend` & `test`.

```
kubectl create ns learning
kubectl create ns frontend
kubectl create ns test
```

label the `learning` & `frontend` with istio injection. Execute the below commands.

```
kubectl label ns learning istio-injection=enabled
kubectl label ns frontend istio-injection=enabled
```

you can verify the applied lables by executing the below commands.

```
kubectl get ns {learning,frontend} --show-labels
```

## Deploying bookinfo application

lets deploy the sample `bookinfo` application inside `learning` namespace. Execute the below command.

```
kubectl apply -f samples/bookinfo/platform/kube/bookinfo.yaml -n learning
kubectl -n learning apply -f samples/bookinfo/networking/bookinfo-gateway.yaml
```

Since we can not get a loadbalancer in minikube let us execute the port-forward on istiod pod which resides in istio-system namespace. execute the below command.

```
kubectl -n istio-system port-forward $(kubectl get pods -n istio-system | grep -v 'NAME' | grep 'istio-ingressgateway' | awk '{print $1}') 8080:8080
```

Now let us go to the browser and try to access the endpoint i.e http://localhost:8080/productpage.


## Deploying Sample application

lets deploy a sample application into `frontend` namespace.

Create a file named `frontend.yaml` with the below content.

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

```
kubectl apply -f frontend.yaml -n frontend
```

you can verify that sample application is created successfully by using the below commands.

```
kubectl get pods -n frontend
kubectl get svc -n frontend
kubectl get sa frontend -n frontend
```

## Deploying Sample application to test namespace with out sidecar

lets deploy the sample application to `test` namespace. Execute the below command.

```
kubectl apply -f frontend.yaml -n test
```

you can verify the pods created using below command.

```
kubectl get pods -n test
```

if you notice the above command closely you only have 1 container i.e main container there is no sidecar container because `test` namespace is not labled with `istio-injection`.

## Testing the bookinfo app from test and frontend apps

Lets try to access the `bookinfo` productpage app from sample apps deployed in `test` & `frontend` namespace.

**From Frontend namespace app**

kubectl -n frontend exec $(kubectl get pods -n frontend | grep -v 'NAME' | awk '{print $1}') -c frontend -- curl -sS http://productpage.learning:9080/productpage

**From test namespace app**

kubectl -n test exec $(kubectl get pods -n test | grep -v 'NAME' | awk '{print $1}') -- curl -sS http://productpage.learning:9080/productpage

Both the commands will provide you a successful response. This is good but how are we getting successful response from the test namespace app although it doesnt have any sidecar istio container. Well it is because by default istio configures the destination workloads to accept both plain text & encrypted traffic so which is why plain traffic request from `test namespace sample app` to `productpage` is working fine.

## Restricting strict mtls mode to bookinfo app

create a file named `strict-mtls-learning.yaml` with the below content.

```
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls
  namespace: learning
spec:
  mtls:
    mode: STRICT
```

**Note** The above peerauthentication policy restricts strict mtls to only the workloads in `learning` namespace. if you want the same behavior to be applied across all the workloads across all namespaces then use the below content.

```
apiVersion: security.istio.io/v1beta1
kind: PeerAuthentication
metadata:
  name: strict-mtls
  namespace: istio-system
spec:
  mtls:
    mode: STRICT
```

lets apply the above file.

```
kubectl apply -f strict-mtls-learning.yaml
```

you can verify the peer authentication policy created using the below command.

```
kubectl get peerauthentication -n learning
```

Now lets try to access the productpage app from both namespaces apps.

**From Frontend namespace app**

kubectl -n frontend exec $(kubectl get pods -n frontend | grep -v 'NAME' | awk '{print $1}') -c frontend -- curl -sS http://productpage.learning:9080/productpage

You will get successful response from `frontend` namespace app.

**From test namespace app**

kubectl -n test exec $(kubectl get pods -n test | grep -v 'NAME' | awk '{print $1}') -- curl -sS http://productpage.learning:9080/productpage

you will get the following error while try to access from `test` namespace app.

**Error**

curl: (56) Recv failure: Connection reset by peer
command terminated with exit code 56

This is expected because the destination workload now can access only encrypted traffic(as specificed in the policy above). However `test namespace app` can only send plain traffic because it doesnt contain istio sidecar so it can not initiate secure traffic.

## Checking whether traffic is encrypted or not

requests sent from app or pod(without istio sidecar) are always plain text traffic i.e not encrypted. To Verify this open 2 terminals

**Terminal-1**

```
kubectl -n learning exec -it $(kubectl get pods -n learning | grep -v 'NAME' | grep productpage | awk '{print $1}') -c istio-proxy -- sh
```

The above command opens a shell session inside `istio-proxy` container of `productpage` pod. then execute the below command.

```
sudo tcpdump dst port 9080 -A
```

The above command listens for traffic on destination port 9080

![alt text](/assets/images/tcp-dump-initiation.png)

**Terminal-2**

initiate the requests from `test namespace app` & `frontend namespace app`

**From Frontend namespace app**

kubectl -n frontend exec $(kubectl get pods -n frontend | grep -v 'NAME' | awk '{print $1}') -c frontend -- curl -sS http://productpage.learning:9080/productpage

when you execute the above command output from terminal-1 looks like below.

![alt text](/assets/images/encrypted-traffic.png)

The above output doesnt displays any clear text or curl headers which indicates that it is encrypted traffic.

**From test namespace app**

kubectl -n test exec $(kubectl get pods -n test | grep -v 'NAME' | awk '{print $1}') -- curl -sS http://productpage.learning:9080/productpage

when you execute the above command output from terminal-1 looks like below.

![alt text](/assets/images/plain-traffic.png)

the above output displays curl headers i.e Endpoint, Host etc etc which indicates that this is plain traffic.

Congratulations. Lets look into Authorization part of istio in next blog.