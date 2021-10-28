---
layout: post
title:  "AWS EKS setup using eksctl utility"
date:   2021-10-28 04:00:00 +0530
category: kubernetes
---
- [EKS](#eks)
   - [Overview](#overview)
   - [Pre-requisites](#pre-requisites)
   - [Setup](#setup)
       - [step-1: Creating iam policy](#step-1-creating-iam-policy)
       - [step-2: Creating user and secret keys](#step-2-creating-user-and-secret-keys)
       - [step-3: Configure the aws cli](#step-3-configure-the-aws-cli)
       - [step-4: Installing eks using eksctl](#step-4-installing-eks-using-eksctl)
   - [Testing](#testing)
   - [Deleting the cluster](#deleting-the-cluster)

## EKS

`EKS` stands for `Elastic kubernetes service` which is a managed kubernetes service of aws. you can use `EKS` to run Kubernetes on AWS without needing to install, operate, and maintain your own Kubernetes control plane or nodes

## Overview

`EKS` can be setup in three methods.

- Using AWS Console
- using aws cli utility
- using eksctl utility

We are going to use `eksctl` utility here which is pretty easy compared to other 2 methods.

**Note** `eksctl` utility is developed by weaveworks. Complete documentation can be found [here](https://eksctl.io/)

## Pre-requisites

Make sure you download the following utilities on your system.

- kubectl
- eksctl
- aws

**kubectl installation**

kubectl installation for windows [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-windows/)<br>
kubectl installation for Linux [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-linux/)<br>
kubectl installation for Macos [here](https://kubernetes.io/docs/tasks/tools/install-kubectl-macos/)<br>

**aws cli installation**

aws cli installation for Windows [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-windows.html)<br>
aws cli installation for Linux [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-linux.html)<br>
aws cli installation for Macos [here](https://docs.aws.amazon.com/cli/latest/userguide/install-cliv2-mac.html)<br>

**eksctl installation**

`eksctl installation for Linux`

```
# eksctl installation for Linux
curl --silent --location "https://github.com/weaveworks/eksctl/releases/latest/download/eksctl_$(uname -s)_amd64.tar.gz" | tar xz -C /tmp
sudo mv /tmp/eksctl /usr/local/bin
```

`eksctl installation for Macos`

```
# eksctl installation for Macos

brew tap weaveworks/tap
brew install weaveworks/tap/eksctl
```

`eksctl installation for Windows`

```
# eksctl installation for Windoes
chocolatey install eksctl
```

**Verification of installed binaries**

eksctl version
kubectl version --client
aws --version

You should get the version details while running the above commands.


## Setup

As mentioned earlier we will be using `eksctl` utility for the setup. eksctl should be configured with required permission to create eks cluster. So we will be first creating policy & secret keys. 

### step-1: Creating iam policy

Go to IAM -> policies -> Create policy -> Select Json -> remove the existing content and Copy & paste below(Note: replace <account-id> with your aws account number). -> provide name of the policy for ex: `eks-creation-permission` and click `creation policy`.

**Note**

you can get the `<account-id>` in aws console -> support -> support center -> Left side of the screen there is a field named `Account number` get value of it and replace the <account-id>> with it.

```
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Action": "ec2:*",
            "Effect": "Allow",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "elasticloadbalancing:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "cloudwatch:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "autoscaling:*",
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "cloudformation:*"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": "eks:*",
            "Resource": "*"
        },
        {
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters"
            ],
            "Resource": [
                "arn:aws:ssm:*:<account-id>:parameter/aws/*",
                "arn:aws:ssm:*::parameter/aws/*"
            ],
            "Effect": "Allow"
        },
        {
             "Action": [
               "kms:CreateGrant",
               "kms:DescribeKey"
             ],
             "Resource": "*",
             "Effect": "Allow"
        },
        {
            "Effect": "Allow",
            "Action": "iam:CreateServiceLinkedRole",
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "autoscaling.amazonaws.com",
                        "ec2scheduled.amazonaws.com",
                        "elasticloadbalancing.amazonaws.com",
                        "spot.amazonaws.com",
                        "spotfleet.amazonaws.com",
                        "transitgateway.amazonaws.com"
                    ]
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateInstanceProfile",
                "iam:DeleteInstanceProfile",
                "iam:GetInstanceProfile",
                "iam:RemoveRoleFromInstanceProfile",
                "iam:GetRole",
                "iam:CreateRole",
                "iam:DeleteRole",
                "iam:AttachRolePolicy",
                "iam:PutRolePolicy",
                "iam:ListInstanceProfiles",
                "iam:AddRoleToInstanceProfile",
                "iam:ListInstanceProfilesForRole",
                "iam:PassRole",
                "iam:DetachRolePolicy",
                "iam:DeleteRolePolicy",
                "iam:GetRolePolicy",
                "iam:GetOpenIDConnectProvider",
                "iam:CreateOpenIDConnectProvider",
                "iam:DeleteOpenIDConnectProvider",
                "iam:TagOpenIDConnectProvider",                
                "iam:ListAttachedRolePolicies",
                "iam:TagRole"
            ],
            "Resource": [
                "arn:aws:iam::<account-id>:instance-profile/eksctl-*",
                "arn:aws:iam::<account-id>:role/eksctl-*",
                "arn:aws:iam::<account-id>:oidc-provider/*",
                "arn:aws:iam::<account-id>:role/aws-service-role/eks-nodegroup.amazonaws.com/AWSServiceRoleForAmazonEKSNodegroup",
                "arn:aws:iam::<account-id>:role/eksctl-managed-*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:GetRole"
            ],
            "Resource": [
                "arn:aws:iam::<account-id>:role/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "iam:CreateServiceLinkedRole"
            ],
            "Resource": "*",
            "Condition": {
                "StringEquals": {
                    "iam:AWSServiceName": [
                        "eks.amazonaws.com",
                        "eks-nodegroup.amazonaws.com",
                        "eks-fargate.amazonaws.com"
                    ]
                }
            }
        }
    ]
}
```

### step-2: Creating user and secret keys

Now lets create a user and attach policy created in step-1(policy: eks-creation-permission).

Go to AWS Console

IAM -> User -> Provide username for ex: k8slearning -> select `Access key - Programmatic access` -> Click `Next permissions` -> At the top there are 3 options select `Attach existing policies directly` -> in the search bar filter with the policy created in step-1 i.e eks-creation-permission -> click the checkmark against the policy -> Click `Next tags` -> Click `Review` -> Click `Create User`.

you will get `access key` & `secret access key`. Make a note of them or you can even download them from csv by clicking `Download.csv`.

**Note** you will not get these credentials once you lost them so make a note of them prior closing the screen.

### step-3: Configure the aws cli

Now we need to configure the aws cli.

Execute the below command. It prompts for few details like `AWS Access key`, `aws secret accesskey`,`region`. So provide the details accordingly.

**Note** `Accesskey`, `Secret access key` are already generated in step-2, provide any region for ex: us-east-1

```
aws configure
```

### step-4: Installing eks using eksctl

Lets us first generate keypair which we can use to login into k8s nodes.

```
ssh-keygen -t rsa -b 4096 -f ./k8slearning
```

**dry run of cluster creation**


```
eksctl create cluster --name k8slearningupdated  --version 1.19 --nodegroup-name pool1 --node-type t2.micro --nodes 2 --node-volume-size 50 --node-volume-type gp2 --ssh-access --ssh-public-key ./k8slearning.pub --dry-run
```

**Note**: Dry run will not create any resources instead it will just show/print what resources it is going to create.

**cluster creation**

```
eksctl create cluster --name k8slearningupdated  --version 1.19 --nodegroup-name pool1 --node-type t2.micro --nodes 2 --node-volume-size 50 --node-volume-type gp2 --ssh-access --ssh-public-key ./k8slearning.pub
```

**Note** Cluster creation takes lot of time. So please wait..

By default kubeconfig file will be writtent ~/.kube/config. 

**Note :**

Incase if you want to get the already generated kubeconfig and write it to another file then you can use the below command.

```
eksctl utils write-kubeconfig --cluster <cluster name> --kubeconfig <filepath>
```

## Testing

just to verify that cluster is ready then you can use the below command.

```
kubectl get nodes
```

cluster info can be checked by using below command.

```
kubectl cluster-info
```

to get the component status then you can use the below command.

```
kubectl get cs
```

you can use the already generated key to login into the worker nodes. for ex:

kubectl get nodes -o wide

the above command provides worker node details(names,ip address). get the external of any of the worker node.

ssh -i k8slearning ec2-user@\<External address\>

## Deleting the cluster

eksctl delete cluster --region=`<region`> --name=`<cluster`>