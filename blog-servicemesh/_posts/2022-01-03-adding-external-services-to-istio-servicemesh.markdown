---
layout: post
title:  "Adding external services to istio servicmesh"
date:   2022-01-03 11:40:00 +0530
category: servicemesh
---

- [Servicemesh](#servicemesh)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Accessing external services](#accessing-external-services)
   - [restricting access to external services using REGISTRY_ONLY mode](#restricting-access-to-external-services-using-REGISTRY_ONLY-mode)
   - [Creating Service Entry](#creating-service-entry)
   

## Servicemesh

## Overview

By default istio sidecars proxies are configured in such a way that all requests to unknown services will pass through. for ex: lets say if you try to send a request from any pod(for ex: busybox) to google.com then request will be sent from pod to it's sidecar proxy, sidecar proxy accepts & pass through this request although google.com is not part of servicmesh.


## Pre-requisites

- k8s cluster
- istio installation

for k8s cluster setup refer to [this](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/12/18/Installing-istio-servicemesh-using-istioctl.html#setup)

for istio installation refer to [this](https://devopsbypr.in/blog-servicemesh/servicemesh/2021/12/18/Installing-istio-servicemesh-using-istioctl.html#step-1-installing-istio-service-mesh)

## Accessing external services

Lets create a new namespace(test) with istio injection enabled and then deploying a sample busybox pod from which we will be making requests to external services. Execute the below commands.

```
kubectl create ns test
kubectl label ns test istio-injection=enabled
kubectl run busyboxcurl --image=radial/busyboxplus:curl --command sleep 6000 -n test
```

exec into the busybox pod and try to access `https://imdpune.gov.in` 

```
kubectl -n test exec -it busyboxcurl -c busyboxcurl -- sh
curl https://imdpune.gov.in or curl -I https://imdpune.gov.in
```

when you execute the above command you will get a successful response. This works because outbound traffic policy mode set/configured for proxy is "ALLOW_ONLY". This is the default behavior.

open new terminal and navigate to the path where istioctl package was downloaded. Execute the below commads.

```
cd istio-1.11.3
export PATH=$PWD/bin:$PATH
#-- installing addons
kubectl apply -f ./samples/addons/
```

lets access the kiali dashboard

```
istioctl dashboard kiali
```

execute some more curl requests from inside the busybox pod. Now observe the graph from kiali dashboard.

![alt text](/assets/images/kiali-externalservice-allow-only.png)

## restricting access to external services using REGISTRY_ONLY mode

as shown in the previous step, istio sidecar proxies are configured with "ALLOW_ONLY" mode by default which is why all external services are accessible.

lets delete all the installed istio components & other namespaces created.

```
kubectl delete ns istio-system
kubectl delete ns learning
kubectl delete ns test
```

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
        privileged: false
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

lets redeploy the istio components. execute the below command.

```
istioctl install -f demo-profile.yaml -y
kubectl create ns test
kubectl label ns test istio-injection=enabled
kubectl run busyboxcurl --image=radial/busyboxplus:curl --command sleep 6000 -n test
# after exec into the busyboxcurl pod exec the below command
curl https://imdpune.gov.in
```

when you run the above command you will get an error.

![alt text](/assets/images/kiali-externalservice-register-only.png)

To get rid of this error we need to create a service entry. This is explained in the next step.

## Creating Service Entry

create a filenamed `service-entry-imd.yaml` with the below content.

```
apiVersion: networking.istio.io/v1alpha3
kind: ServiceEntry
metadata:
  name: external-svc-imd-weather
  namespace: test
spec:
  hosts:
  - imdpune.gov.in
  exportTo:
  - "."
  location: MESH_EXTERNAL
  ports:
  - number: 443
    name: https
    protocol: https
  resolution: DNS
```

lets install the service entry.

```
kubectl apply -f service-entry-imd.yaml
```

you can verify the service entry creation is using the below command.

```
kubectl get serviceentry -n test
```

open new terminal and navigate to the path where istioctl package was downloaded. Execute the below commads to install addons(kiali, prometheus etc etc)

```
cd istio-1.11.3
export PATH=$PWD/bin:$PATH
#-- installing addons
kubectl apply -f ./samples/addons/
istioctl dashboard kiali
```

lets test now

```
kubectl -n test exec -it busyboxcurl -c busyboxcurl -- sh
curl https://imdpune.gov.in
```

you will get a successful response after executing the above command. however you will get an error if you try to access any other external service other than `https://imdpune.gov.in` this is expected because we have enabled/allowed only `https://imdpune.gov.in` in the service entry created above.

Navigate to kiali dashboard after making couple of curl requests. Screenshot below for reference.

![alt text](/assets/images/kiali-external-service-entry-imd.png)
