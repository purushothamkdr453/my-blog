---
layout: post
title:  "Selfmanaged kubeadm kubernetes cluster on oracle centos nodes"
date:   2021-11-05 10:30:00 +0530
category: kubernetes
---
- [selfmanaged kubeadm k8s cluster on oracle centos nodes](#selfmanaged-kubeadm-k8s-cluster-on-oracle-centos-nodes)
   - [Overview](#overview)
   - [Setup](#setup)
       - [step-1: Creating Virtual Cloud Network](#step-1-creating-virtual-cloud-network)
       - [step-2: Creating internet gateway](#step-2:-creating-internet-gateway)
       - [step-3: Creating natgateway](#step-3:-creating-natgateway)
       - [step-4: Creating Route rules and security lists](#step-4-creating-route-rules-and-security-lists)
       - [step-5: Creating public and private subnets](#step-5-creating-public-and-private-subnets)
       - [step-6: Create oracle virtual machine nodes](#step-6:-create-oracle-virtual-machine-nodes)
       - [step-7: Disable swap and set some other pre requisities](#step-7:-disable-swap-and-set-some-other-pre-requisities)
       - [step-8: installing container run time](#step-8:-installing-container-run-time)
       - [step-9: installing k8s components](#step-9:-installing-k8s-components)
       - [step-10: kubernetes initialization](#step-10:-kubernetes-initialization)
       - [step-11: kubernetes worker node join](#step-11:-kubernetes-worker-node-join)
       - [step-12: installing overlay network](#step-12:-installing-overlay-network)
       - [step-13: taint the master node](#step-13:-taint-the-master-node)
       - [step-14: testing](#step-14:-testing)
       - [step-15: Deploying oci cloud controller manager](#step-15:-deploying-oci-cloud-controller-manager)

## selfmanaged kubeadm k8s cluster on oracle centos nodes

`kubernetes cluster` can be setup in multiple ways. in this tutorial we are going to use kubeadm to setup kubernetes on existing infra(virtual machines).

## Overview

## Setup

`networking setup`

vcn (name: k8s learning) -> 10.0.0.0/26.<br/>
subnet1 (name: public ) -> 10.0.0.0/27. This is a public subnet.<br/>
subnet2 (name: private) -> 10.0.0.32/27. This is a private subnet.<br/>

`Virtual machine details`

5 nodes - centos 7 operating system

bastion ( 1 CPU & 1GB RAM) -> this will be launched in public subnet which is 10.0.0.0/27.<br/>
master1 ( 2 CPU & 4GB RAM) -> this will be treated as k8s master1 node launched in private subnet(10.0.0.32/27).<br/>
master2 ( 2 CPU & 4GB RAM) -> this will be treated as k8s master2 node launched in private subnet(10.0.0.32/27).<br/>
worker1 ( 2 CPU & 4GB RAM) -> this will be treated as k8s worker1 node launched in private subnet(10.0.0.32/27).<br/>
worker2 ( 2 CPU & 4GB RAM) -> this will be treated as k8s worker2 node launched in private subnet(10.0.0.32/27).<br/>

so it should look something like this.

![alt text](/assets/images/k8s-multimaster-multiworker-loadbalancer.png)

Loadbalancer -> which routes the requests to master nodes(master1, master2). Loadbalancer will be launched in public subnet.

`k8s setup`

2 masters(master1,master2) & 2 workers(worker1,worker2) &  1 Loadbalancer

### step-1: Creating Virtual Cloud Network

create virtual cloud network. steps to create vcn as follows.

OCI Console -> Networking -> Virtual cloud networks -> Create VCN -> provide any vcn name(for ex: k8slearning) -> provide ipv4 cidr(for ex: 10.0.0.0/26) -> click `create VCN`.

### step-2: Creating internet gateway

create internet gateway. steps to create internet gateway as follows.

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning) -> Select Internetgateways -> select `Create Internet Gateway` -> provide name for internet gateway(for ex: myigw)

### step-3: Creating natgateway

create nat gateway. steps to create nat gateway as follows.

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning) -> select Natgateway -> select `Create NAT Gateway` -> provide name for nat gateway(for ex: myngw) -> select `ephemeral public ip address` -> click `create NAT Gateway`.

### step-4: Creating Route rules and security lists

**Route table**

Create 2 route tables `public` & `private`.

steps to create route tables as follows.

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning) -> Route tables -> select `create route table` -> provide name for route table for (ex: public or ex: private) -> click `create`.

**Note** make sure you create 2 route tables 1 for private and another one for public.

now lets add route rules.

`public route table rules`

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning)  -> Select `Route tables` -> select `public` route table -> Click `Add Route rules` -> select `target type` as internet gateway -> provide desitnation cidr as `0.0.0.0/0` -> select `target internet gateway` as myigw(which is created in step-2).

`Private route table rules`

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning)  -> Select `Route tables` -> select `public` route table -> Click `Add Route rules` -> select `target type` as `NAT Gateway` -> provide desitnation cidr as `0.0.0.0/0` -> select `target nat gateway` as myngw(which is created in step-3).

**security lists**

create 2 security lists `public` & `private`

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning) -> Security lists -> select `create security list` -> provide name for security list for (ex: public or ex: private) -> click `create`.

**Note** make sure you create 2 security lists 1 for private and another one for public.

now lets add ingress/egress rules to both private and public security lists.

`public security list`

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning)  -> Select `Security Lists` -> select `public` the add ingress/egrees rules as mentioned below.

**ingress**

![alt text](/assets/images/public-security-list-ingress.png)

**egress**

![alt text](/assets/images/public-security-list-egress.png)

`private security list`

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning)  -> Select `Security Lists` -> select `private` the add ingress/egrees rules as mentioned below.

**ingress**

![alt text](/assets/images/private-security-list-ingress.png)

**egress**

![alt text](/assets/images/private-security-list-egress.png)

### step-5: Creating public and private subnets

**public subnet**

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning) -> select `subnets` -> create `create subnet` -> provide name for subnet(for ex: public) -> choose subnet type `regional` -> provide cidr as `10.0.0.0/27` -> select `route table` dropdown choose `public` -> select `subnet access` as public subnet -> select `security lists` dropdown choose `public` -> click `create`.

**private subnet**

OCI Console -> Networking -> Virtual cloud networks -> select the created VCN(for ex: k8slearning) -> select `subnets` -> create `create subnet` -> provide name for subnet(for ex: private) -> choose subnet type `regional` -> provide cidr as `10.0.0.32/27` -> select `route table` dropdown choose `private` -> select `subnet access` as private subnet -> select `security lists` dropdown choose `private` -> click `create`.

### step-6: Create oracle virtual machine nodes

ssh-keygen -t rsa -f k8slearning

create 5 virtual machines(bastion, master1, master2, worker1, worker2).

**bastion**

Console -> select `Compute` -> Select `instances` -> click `create instance` -> provide name as `bastion` -> select image as `centos` & shape as `VM.Standard.E3.Flex` with 1 cpu & 1 gb ram -> select `networking` as k8slearning -> select `subnet` as public -> select `add ssh keys` then select `upload public key` -> upload `k8slearning.pub` public key which is created above -> click `create`.

![alt text](/assets/images/oracle-bastion-centos-image-shape-spec.png)

![alt text](/assets/images/oracle-bastion-centos-networking-spec.png)

![alt text](/assets/images/oracle-bastion-centos-sshkeys.png)

**master1**

Console -> select `Compute` -> Select `instances` -> click `create instance` -> provide name as `k8smaster1` -> select image as `centos` & shape as `VM.Standard.E3.Flex` with 2 cpu & 4 gb ram -> select `networking` as k8slearning -> select `subnet` as private -> select `add ssh keys` then select `upload public key` -> upload `k8slearning.pub` public key which is created above -> click `create`.

![alt text](/assets/images/oracle-k8smaster-centos-hostdetails.png)

![alt text](/assets/images/oracle-k8smaster-centos-image-shape-spec.png)

![alt text](/assets/images/oracle-k8smaster-centos-networking-spec.png)

![alt text](/assets/images/oracle-k8smaster-centos-sshkeys.png)

**master2**

Console -> select `Compute` -> Select `instances` -> click `create instance` -> provide name as `k8smaster2` -> select image as `centos` & shape as `VM.Standard.E3.Flex` with 2 cpu & 4 gb ram -> select `networking` as k8slearning -> select `subnet` as private -> select `add ssh keys` then select `upload public key` -> upload `k8slearning.pub` public key which is created above -> click `create`.

![alt text](/assets/images/oracle-k8smaster-centos-hostdetails.png)

![alt text](/assets/images/oracle-k8smaster-centos-image-shape-spec.png)

![alt text](/assets/images/oracle-k8smaster-centos-networking-spec.png)

![alt text](/assets/images/oracle-k8smaster-centos-sshkeys.png)


**worker1**


Console -> select `Compute` -> Select `instances` -> click `create instance` -> provide name as `k8sworker1` -> select image as `centos` & shape as `VM.Standard.E3.Flex` with 2 cpu & 4 gb ram -> select `networking` as k8slearning -> select `subnet` as private -> select `add ssh keys` then select `upload public key` -> upload `k8slearning.pub` public key which is created above -> click `create`.

![alt text](/assets/images/oracle-k8sworker1-centos-hostdetails.png)

![alt text](/assets/images/oracle-k8smaster-centos-image-shape-spec.png)

![alt text](/assets/images/oracle-k8smaster-centos-networking-spec.png)

![alt text](/assets/images/oracle-k8smaster-centos-sshkeys.png)

**worker2**

Console -> select `Compute` -> Select `instances` -> click `create instance` -> provide name as `k8sworker1` -> select image as `centos` & shape as `VM.Standard.E3.Flex` with 2 cpu & 4 gb ram -> select `networking` as k8slearning -> select `subnet` as private -> select `add ssh keys` then select `upload public key` -> upload `k8slearning.pub` public key which is created above -> click `create`.

![alt text](/assets/images/oracle-k8sworker1-centos-hostdetails.png)

![alt text](/assets/images/oracle-k8smaster-centos-image-shape-spec.png)

![alt text](/assets/images/oracle-k8smaster-centos-networking-spec.png)

![alt text](/assets/images/oracle-k8smaster-centos-sshkeys.png)

### step-6: Create Reserverd IP

Let us create a `reserved ip` which will be used later for loadbalancer.

OCI -> Networking -> IP Management -> Reserverd PublicIP -> Create `Reserved PublicIP Address` -> provide the name for reserved public ip for ex: k8sloadbalancer -> From the Dropdown of `IP Address Source` select Oracle -> Click `submit`.

### step-7: Creating loadbalancer

OCI -> Networking -> Loadbalancer -> Provide loadbalancer name `apiserverlb` -> select visibility `public` -> select reserved ip address(select the reservered ip which was created in previous step from dropdown) -> Select the VCN(created earlier) -> select public subnet(which is created earlier) -> select next -> select `Weighted Round Robin` -> Select Add backends and choose master1, master2 from the list -> click `Add backends` -> under `health check policy` -> Choose protocol as `tcp` and `port` as 6443 -> select next -> under listener choose protocol as `https` and port as `6443` 


### step-7: Disable swap and set some other pre requisities


Verify that `Product uuid` & `mac address` is unique for every node(for both masters/workers). You can get product uuid by running the below command.

```
# Product UUID
sudo cat /sys/class/dmi/id/product_uuid

# MAC Address
ip addr show | grep link/ether | awk '{print $2}'
```

**Note** The below settings should be applied on both `k8smaster` & `k8sworker` nodes.

**Disable swap**

```
sudo sed -i '/ swap / s/^\(.*\)$/#\1/g' /etc/fstab
sudo swapoff -a
```

**Disable SELinux**

```
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config
```

**enable and load kernel modules**

```
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
br_netfilter
EOF

cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
EOF

sudo sysctl --system
```

**Disable firewall**

`on both master and worker nodes`

```
systemctl stop firewalld
systemctl disable firewalld
```

### step-8: installing container run time

you can install any container run time `containerd` or `crio` or `docker`. here we will be installing `containerd runtime`.

**Note** The below commands should be executed on both `k8smaster` & `k8sworker` nodes.

```
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables  = 1
net.ipv4.ip_forward                 = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sudo sysctl --system

sudo yum install -y yum-utils

sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

sudo yum install -y containerd.io

sudo mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml

sudo systemctl restart containerd
```

### step-9: installing k8s components

**Note** The below commands should be executed on both `k8smaster` & `k8sworker` nodes.

`installing kubelet, kubeadm, kubectl`

```
cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
repo_gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/yum-key.gpg https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

sudo yum install -y kubelet-1.21.0 kubeadm-1.21.0 kubectl-1.21.0 --disableexcludes=kubernetes

sudo systemctl restart kubelet
sudo systemctl enable --now kubelet
```

### step-10: kubernetes initialization

login into the master node i.e `k8smaster` node via `bastion` host. become a root user by executing `sudo -i` then create a file named `kube-init.yaml` with the below content.

```
---
apiVersion: kubeadm.k8s.io/v1beta2
kind: ClusterConfiguration
networking:
   podSubnet: 192.168.0.0/16
kubernetesVersion: "v1.21.0"
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

then execute the below command.

```
kubeadm init --config kube-init.yaml
```

output of the above command provides you the `kubeconfig` and `kubeadm join` command.

### step-11: kubernetes worker node join

**Note** This step should be executed on worker nodes.

Execute the `kubeadm join` command. you will get the command from the output of `kubeadm init --config kube-init.yaml`.

Incase if you lost the the join command from the output of `kubeadm init --config kube-init.yaml` command then execute the below command on master which will provide you the join command.

```
#This should be executed on master node
kubeadm token create --print-join-command
```

### step-12: installing overlay network

**Note** This step should be executed on master node.

deploy any overlay network. in this blog we are going to install calico network.

```
kubectl apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

### step-13: taint the master node

taint then master node with the below label so that no workloads will be scheduled on master nodes.

```
kubectl taint node k8smaster node-role.kubernetes.io/master=true:NoSchedule
```

### step-14: testing

login into `k8smaster` node via `bastion`. Become root user by executing `sudo -i`.

export KUBECONFIG=/etc/kubernetes/admin.conf
kubectl get nodes

kubectl cluster-info

Makesure all the nodes are in ready status which indicates that cluster is setup successfully.

Now you can deploy a sample pod

for ex:

```
# create a pod
kubectl run busybox --image busybox --command sleep 1d
# verify whether pod is running or not
kubectl get pod busybox
```

Awesome so we are able to deploy the pods successfully. Lets deploy nginx pod and expose it as `Loadbalancer` service

```
kubectl expose pod nginx --type=LoadBalancer --port=80

kubectl get svc nginx
```

if you notice the output of `kubectl get svc nginx` external_ip is in pending status. it is because k8s nodes are not able to talk or have permissions to `oci cloud api interface` to create loadbalancer. 

So to get rid of this we need to deploy `oci cloud controller manager`. steps are mentioned below.

### step-15: Deploying oci cloud controller manager

Execute the below command on `both master node & worker nodes`.

```
kubeadm reset
```

**Note:** The above command removes everything that was created.

Create a file named `kube-init.yaml` on master node with the below content.

```
apiVersion: kubeadm.k8s.io/v1beta2
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: abcdef.0123456789abcdef
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
nodeRegistration:
  criSocket: /run/containerd/containerd.sock
  name: k8smaster
  taints:
  - effect: NoSchedule
    key: node-role.kubernetes.io/master
  kubeletExtraArgs:
    cloud-provider: external
    provider-id: <ocid of master node>
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta2
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager:
  extraArgs:
    cloud-provider: external
dns:
  type: CoreDNS
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: k8s.gcr.io
kind: ClusterConfiguration
kubernetesVersion: v1.20.0
networking:
  dnsDomain: cluster.local
  serviceSubnet: 10.96.0.0/12
  podSubnet: 192.168.0.0/16
scheduler: {}
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
cgroupDriver: systemd
```

**Note** \<ocid of master node\> replace this with ocid of master node. ocid are unique for every node.

create a file named `kube-join.yaml` on worker nodes with the below content.

```
apiVersion: kubeadm.k8s.io/v1beta2
caCertPath: /etc/kubernetes/pki/ca.crt
discovery:
  bootstrapToken:
    apiServerEndpoint: k8smaster:6443
    token: abcdef.0123456789abcdef
    unsafeSkipCAVerification: true
  timeout: 5m0s
  tlsBootstrapToken: abcdef.0123456789abcdef
kind: JoinConfiguration
nodeRegistration:
  name: k8sworker1
  taints: null
  kubeletExtraArgs:
    cloud-provider: external
    provider-id: <ocid of worker node>
```

**Note** \<ocid id of worker node\> replace this with ocid of worker node. ocid are unique for every node. so if you are joining multiple worker nodes then makesure you update the placeholder with respective node's ocid.

**on master node**

```
kubeadm init --config kube-init.yaml
kubectl --kubeconfig=/etc/kubernetes/admin.conf apply -f https://docs.projectcalico.org/manifests/calico.yaml
```

**on worker nodes**

```
kubeadm join --config kube-join.yaml
```

now create a file with named `cloud-provider-example.yaml` with the below content.

```
auth:
  region: us-phoenix-1
  tenancy: ocid1.tenancy.oc1..aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
  user: ocid1.user.oc1..aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
  key: |
    -----BEGIN RSA PRIVATE KEY-----
    <snip>
    -----END RSA PRIVATE KEY-----
  # Omit if there is not a password for the key
  passphrase: supersecretpassword
  fingerprint: 8c:bf:17:7b:5f:e0:7d:13:75:11:d6:39:0d:e2:84:74

  # Omit all of the above options then set useInstancePrincipals to true if you
  # want to use Instance Principals API access
  # (https://docs.us-phoenix-1.oraclecloud.com/Content/Identity/Tasks/callingservicesfrominstances.htm).
  # Ensure you have setup the following OCI policies and your kubernetes nodes are running within them
  # allow dynamic-group [your dynamic group name] to read instance-family in compartment [your compartment name]
  # allow dynamic-group [your dynamic group name] to use virtual-network-family in compartment [your compartment name]
  # allow dynamic-group [your dynamic group name] to manage load-balancers in compartment [your compartment name]
  useInstancePrincipals: false

# compartment configures Compartment within which the cluster resides.
compartment: ocid1.compartment.oc1..aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

# vcn configures the Virtual Cloud Network (VCN) within which the cluster resides.
vcn: ocid1.vcn.oc1..aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

loadBalancer:
  # subnet1 configures one of two subnets to which load balancers will be added.
  # OCI load balancers require two subnets to ensure high availability.
  subnet1: ocid1.subnet.oc1.phx.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

  # subnet2 configures the second of two subnets to which load balancers will be
  # added. OCI load balancers require two subnets to ensure high availability.
  subnet2: ocid1.subnet.oc1.phx.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

  # SecurityListManagementMode configures how security lists are managed by the CCM.
  # If you choose to have security lists managed by the CCM, ensure you have setup the following additional OCI policy:
  # Allow dynamic-group [your dynamic group name] to manage security-lists in compartment [your compartment name]
  #
  #   "All" (default): Manage all required security list rules for load balancer services.
  #   "Frontend":      Manage only security list rules for ingress to the load
  #                    balancer. Requires that the user has setup a rule that
  #                    allows inbound traffic to the appropriate ports for kube
  #                    proxy health port, node port ranges, and health check port ranges.
  #                    E.g. 10.82.0.0/16 30000-32000.
  #   "None":          Disables all security list management. Requires that the
  #                    user has setup a rule that allows inbound traffic to the
  #                    appropriate ports for kube proxy health port, node port
  #                    ranges, and health check port ranges. E.g. 10.82.0.0/16 30000-32000.
  #                    Additionally requires the user to mange rules to allow
  #                    inbound traffic to load balancers.
  securityListManagementMode: All

  # Optional specification of which security lists to modify per subnet. This does not apply if security list management is off.
  securityLists:
    ocid1.subnet.oc1.phx.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa: ocid1.securitylist.oc1.iad.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
    ocid1.subnet.oc1.phx.bbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbbb: ocid1.securitylist.oc1.iad.aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa

# Optional rate limit controls for accessing OCI API
rateLimiter:
  rateLimitQPSRead: 20.0
  rateLimitBucketRead: 5
  rateLimitQPSWrite: 20.0
  rateLimitBucketWrite: 5
```

**Note**: Make sure you replace the the above details with your details.

After once the details are updated then execute the below command.

```
kubectl  create secret generic oci-cloud-controller-manager \
     -n kube-system                                           \
     --from-file=cloud-provider.yaml=cloud-provider-example.yaml
```

Now lets deploy the `oci cloud controller manager` using the below commands.

```
kubectl apply -f https://raw.githubusercontent.com/oracle/oci-cloud-controller-manager/master/manifests/cloud-controller-manager/oci-cloud-controller-manager.yaml

kubectl apply -f https://raw.githubusercontent.com/oracle/oci-cloud-controller-manager/master/manifests/cloud-controller-manager/oci-cloud-controller-manager-rbac.yaml
```

Now makesure the `cloud controller` pod is running in `kube-system` namespace.

```
kubectl -n kube-system get po | grep oci
```

Now try to create service of loadbalancer.

```
kubectl run nginx --image nginx
kubectl expose pod nginx --type=LoadBalancer --port=80
```





