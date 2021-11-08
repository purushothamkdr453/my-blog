---
layout: post
title:  "Selfmanaged kubeadm kubernetes cluster on oracle cloud"
date:   2021-11-04 05:00:00 +0530
category: kubernetes
---
- [selfmanaged kubeadm k8s cluster on oracle cloud](#selfmanaged-kubeadm-k8s-cluster-on-oracle-cloud)
   - [Overview](#overview)
   - [Setup](#setup)
       - [step-1: Creating Virtual Cloud Network](#step-1-creating-virtual-cloud-network)
       - [step-2: Creating Route rules and security lists](#step-2-creating-route-rules-and-security-lists)
       - [step-3: Creating subnets and attaching route rules,security lists](#step-3-creating-subnets-and-attaching-route-rules,-security lists)

## selfmanaged kubeadm k8s cluster on oracle cloud

`kubernetes cluster` can be setup in multiple ways. in this tutorial we are going to use kubeadm to setup kubernetes on existing infra(virtual machines).

## Overview

## Setup

### Creating Virtual Cloud Network

sudo apt install net-tools

Verify that Product uuid is unique for every node(for both masters/workers). You can get product uuid by running the below command.

Product UUID

```
sudo cat /sys/class/dmi/id/product_uuid
```

MAC Address

```
ip addr show | grep link/ether | awk '{print $2}'
```

**Disable swap**

swapoff -a
sed -i '/swap/d' /etc/fstab

**Disable firewall**

systemctl disable --now ufw

**Enable and load kernel modules**

cat >> /etc/modules-load.d/containerd.conf <<EOF
overlay
br_netfilter
EOF

modprobe overlay
modprobe br_netfilter

**Add kernel settings**

cat >>/etc/sysctl.d/kubernetes.conf<<EOF
net.bridge.bridge-nf-call-ip6tables = 1
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
EOF

sysctl --system

**Installing container runtime**

sudo apt-get update

sudo apt-get install \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu \
  $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt-get update

sudo apt-get containerd.io

docker info | grep -i cgroup

sudo mkdir -p /etc/containerd

containerd config default | sudo tee /etc/containerd/config.toml

sudo systemctl restart containerd


**installing kubelet, kubeadm, kubectl**

sudo apt-get update

sudo apt-get install -y apt-transport-https ca-certificates curl

sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg

echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list

sudo apt-get update

sudo apt-get install -y kubelet=1.18.0-00 kubeadm=1.18.0-00 kubectl=1.18.0-00

sudo apt-mark hold kubelet kubeadm kubectl

**on master**

kubeadm init --pod-network-cidr=192.168.0.0/16 --cri-socket=/run/containerd/containerd.sock

kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml

kubeadm join 10.0.0.41:6443 --token 7l4yww.m5qjv6o1wz72stbq --discovery-token-ca-cert-hash sha256:ff0fed7469dadfe11a433998f075037ae1817eb46ede2a591dc48c94f68c3332 --cri-socket=/run/containerd/containerd.sock