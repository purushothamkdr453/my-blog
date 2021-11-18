---
layout: post
title:  "CKS - System Hardening using Apparmor"
date:   2021-11-18 09:10:00 +0530
category: cks
---

- [Apparmor](#apparmor)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Setup](#setup)
       - [step-1: Creating Apparmor profile](#step-1-creating-apparmor-profile)
       - [step-2: Loading profile](#step-2-loading-profile)
       - [step-3: Applying apparmor profile to a pod](#step-3-applying-apparmor-profile-to-a-pod)
       - [step-4: checking whether pod is running with applied profile](#step-4-checking-whether-pod-is-running-with-applied-profile)
       - [step-5: pod violating apparmor profile](#step-5-pod-violating-apparmor-profile)
       - [step-6: when profile can not be loaded to a pod](#step-6-when-profile-can-not-be-loaded-to-a-pod)
       - [step-7: disabling apparmor profile](#step-7-disabling-apparmor-profile)
       - [step-8: Logging the voilated events](#step-8-logging-the-voilated-events)
    - [Reference](#reference)
       

## Apparmor

## Overview

AppArmor is a Linux kernel security module that supplements the standard Linux user and group based permissions to confine programs to a limited set of resources. AppArmor can be configured for any application to reduce its potential attack surface and provide greater in-depth defense. It is configured through profiles tuned to allow the access needed by a specific program or container, such as Linux capabilities, network access, file permissions, etc. Each profile can be run in either enforcing mode, which blocks access to disallowed resources, or complain mode, which only reports violations.

AppArmor can help you to run a more secure deployment by restricting what containers are allowed to do, and/or provide better auditing through system logs. However, it is important to keep in mind that AppArmor is not a silver bullet and can only do so much to protect against exploits in your application code. It is important to provide good, restrictive profiles, and harden your applications and cluster from other angles as well.


## Pre-requisites

- k8s cluster

Before we apply the apparmor profile to pod or container there are pre-requisities which should be met.

**Underlying k8s nodes support for apparmor**

makesure underlying nodes of k8s support apparmor. you can verify whether underlying nodes support apparmor by executing the below command.

So login into k8s nodes and execute the below command.

```
# Apparmor kernel module is loaded or not
cat /sys/module/apparmor/parameters/enabled
# Output of the above command should be 'Y" 
```

**Note**: Ubuntu OS by default ships with apparmor.

**check whether kubelet support apparmor**

kubelet should also support apparmor. you can check by executing the below command.

```
kubectl get nodes -o=jsonpath=$'{range .items[*]}{@.metadata.name}: {.status.conditions[?(@.reason=="KubeletReady")].message}\n{end}'
```

Output of the above command should be `kubelet is posting ready status. AppArmor enabled`

**Container runtime should support Apparmor**

All the k8s supported runtimes such as 'containerd`, `crio`, `docker` support apparmor.

**Verify that apparmor service is running or not**

```
systemctl status apparmor
```

**Verify that apparmor has loaded some default profiles already**

```
aa-status
```

or

```
cat /sys/kernel/security/apparmor/profiles 
```

The above commands display info about `loaded apparmor profiles`.

Profiles are nothing but a rules/conditions which restrict the programs(pods or containers) to certains resources(files, linux capabiltiies, network access etc.)

There are 3 modes for profiles

- Enforce
- Complain
- unconfined

 In enforce mode –  AppArmor prevents applications from taking restricted actions. 
 In complain mode - AppArmor allows applications to take restricted actions and creates a log entry complaining about this.
 
## Setup

### step-1: Creating Apparmor profile

create a file named `deny-file-writes` under `/etc/apparmor.d` directory with the below content.

```
#include <tunables/global>
 
# profile <profilename> <flags> 
profile deny-file-writes flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  deny /** w,
}
```

The above profile denies all file writes. our profile name is `deny-file-writes`(same can be seen above).

### step-2: Loading profile 

Now we have profile created. Lets load it.

```
apparmor_parser /etc/apparmor.d/deny-file-writes
```

**Note** the above command by default loads the profile in `enforce` mode. if you
want to load the profile in `complain mode` the execute the below command.

```
apparmor_parser -C /etc/apparmor.d/deny-file-writes
```

Lets verify whether the profile is loaded or not by executing the below command.

```
aa-status
```
### step-3: Applying apparmor profile to a pod

Create a pod named `hello-apparmor.yaml` with below content.

```
apiVersion: v1
kind: Pod
metadata:
  name: hello-apparmor
  annotations:
    # Tell Kubernetes to apply the AppArmor profile "deny-file-writes".
    container.apparmor.security.beta.kubernetes.io/hello: localhost/deny-file-writes
spec:
  containers:
  - name: hello
    image: busybox
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
```

```
kubectl create -f hello-apparmor.yaml
```

you can verify whether the profile is applied to pod or not by executing the below command.

```
kubectl get events | grep hello-apparmor
```

![alt text](/assets/images/apparmor-status-from-get-events.png)


### step-4: checking whether pod is running with applied profile

in the previous step we have already seen whether profile is applied to a pod or not. There is another way we can check from inside pod. Lets exec into the pod.

```
# exec into the pod
kubectl exec -it hello-apparmor -- sh
# checking the applied profile
cat /proc/1/attr/current
```

### step-5: pod violating apparmor profile

Lets test a scenario where it voilates the apparmor profile.

```
# create a file using touch
touch message.txt
```

the above command fails because it is voilating the applied profile(which denies file writes).

### step-6: when profile can not be loaded to a pod

Lets try to apply a profile which doesnt exist. Create a file named `test-apparmor.yaml`

```
apiVersion: v1
kind: Pod
metadata:
  name: test-apparmor
  annotations:
    container.apparmor.security.beta.kubernetes.io/hello: localhost/test-apparmor
spec:
  containers:
  - name: hello
    image: busybox
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
```

```
kubectl apply -f test-apparmor.yaml
```

pod creation gets failed. Please refer to the screenshot below.

![alt text](/assets/images/apparmor-applying-profile-which-doesnt-exist.png)


### step-7: disabling apparmor profile

if you want to disable the profile the execute the below commands.

```
# ln -s /etc/apparmor.d/<profile-name> /etc/apparmor.d/disable/
ln -s /etc/apparmor.d/deny-file-writes /etc/apparmor.d/disable/
# apparmor_parser -R /etc/apparmor.d/<profile-name>
apparmor_parser -R /etc/apparmor.d/deny-file-writes
```

### step-8: Logging the voilated events

create a file named `deny-file-writes-audit` under `/etc/apparmor.d/` directory with below content.

```
#include <tunables/global>
 
# profile <profilename> <flags> 
profile deny-file-writes-audit flags=(attach_disconnected) {
  #include <abstractions/base>

  file,

  # Deny all file writes.
  audit deny /** w,
}
```

```
apparmor_parser /etc/apparmor.d/deny-file-writes-audit
```

makesure that new profile is loaded by executing `aa-status` command.

create a file named `test-apparmor-audit.yaml` with below content.

```
apiVersion: v1
kind: Pod
metadata:
  name: test-apparmor-audit
  annotations:
    container.apparmor.security.beta.kubernetes.io/hello: localhost/deny-file-writes-audit
spec:
  containers:
  - name: hello
    image: busybox
    command: [ "sh", "-c", "echo 'Hello AppArmor!' && sleep 1h" ]
```

```
kubectl apply -f test-apparmor-audit.yaml
```

Now lets exec into the pod and try to create file

```
kubectl exec -it test-apparmor-audit -- sh
touch /tmp/purushotham.txt
grep -i "/tmp/purushotham.txt" /var/log/syslog 
```

**Note** denied actions can be logged any of the following locations.

- /var/log/messages
- /var/log/syslog
- /var/log/audit/audit.log
- dmesg

Please refer to the below screenshot for reference.

![alt text](/assets/images/apparmor-audit-logs.png)

## Reference

For more information about `apparmor` please refer to [apparmor](https://gitlab.com/apparmor/apparmor/-/wikis/Documentation) documentation.