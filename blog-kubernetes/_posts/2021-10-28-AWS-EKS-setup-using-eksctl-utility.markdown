---
layout: post
title:  "Kubernetes Introduction"
date:   2021-10-28 11:10:00 +0530
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
      "Sid": "EC2IG",
      "Effect": "Allow",
      "Action": "ec2:DeleteInternetGateway",
      "Resource": "arn:aws:ec2:*:*:internet-gateway/*"
    },
    {
      "Sid": "ELBIAM",
      "Effect": "Allow",
      "Action": "iam:CreateServiceLinkedRole",
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "iam:AWSServiceName": "elasticloadbalancing.amazonaws.com"
        }
      }
    },
    {
      "Sid": "IAM",
      "Effect": "Allow",
      "Action": [
        "iam:CreateInstanceProfile",
        "iam:DeleteInstanceProfile",
        "iam:GetRole",
        "iam:GetInstanceProfile",
        "iam:RemoveRoleFromInstanceProfile",
        "iam:CreateRole",
        "iam:DeleteRole",
        "iam:AttachRolePolicy",
        "iam:PutRolePolicy",
        "iam:ListInstanceProfiles",
        "iam:AddRoleToInstanceProfile",
        "iam:ListInstanceProfilesForRole",
        "iam:PassRole",
        "iam:CreateServiceLinkedRole",
        "iam:DetachRolePolicy",
        "iam:DeleteRolePolicy",
        "iam:DeleteServiceLinkedRole",
        "iam:GetRolePolicy",
        "iam:ListAttachedRolePolicies"
      ],
      "Resource": [
        "arn:aws:iam::*:instance-profile/eksctl-*",
        "arn:aws:iam::*:role/eksctl-*",
        "arn:aws:iam::*:role/aws-service-role/eks.amazonaws.com/*",
        "arn:aws:iam::*:role/aws-service-role/eks-nodegroup.amazonaws.com/*"
      ]
    },
    {
      "Sid": "IAMOIDC",
      "Effect": "Allow",
      "Action": "iam:GetOpenIDConnectProvider",
      "Resource": "arn:aws:iam::<account-id>:oidc-provider/oidc.eks.eu-west-1.amazonaws.com/*"
    },
    {
      "Sid": "EC2",
      "Effect": "Allow",
      "Action": [
        "ec2:AuthorizeSecurityGroupIngress",
        "ec2:DescribeInstances",
        "ec2:AttachInternetGateway",
        "ec2:DeleteRouteTable",
        "ec2:RevokeSecurityGroupEgress",
        "ec2:CreateRoute",
        "ec2:CreateInternetGateway",
        "ec2:DescribeVolumes",
        "ec2:DeleteInternetGateway",
        "ec2:DescribeKeyPairs",
        "ec2:ImportKeyPair",
        "ec2:CreateTags",
        "ec2:RunInstances",
        "ec2:DisassociateRouteTable",
        "ec2:CreateVolume",
        "ec2:RevokeSecurityGroupIngress",
        "ec2:DescribeImageAttribute",
        "ec2:DeleteNatGateway",
        "ec2:CreateSubnet",
        "ec2:DescribeSubnets",
        "ec2:AttachVolume",
        "ec2:CreateNatGateway",
        "ec2:CreateVpc",
        "ec2:DescribeVpcAttribute",
        "ec2:ModifySubnetAttribute",
        "ec2:DescribeAvailabilityZones",
        "ec2:ReleaseAddress",
        "ec2:DeleteLaunchTemplate",
        "ec2:DescribeSecurityGroups",
        "ec2:CreateLaunchTemplate",
        "ec2:DescribeVpcs",
        "ec2:DeleteSubnet",
        "ec2:DescribeVolumesModifications",
        "ec2:AssociateRouteTable",
        "ec2:DescribeInternetGateways",
        "ec2:DeleteVolume",
        "ec2:DescribeAccountAttributes",
        "ec2:DescribeRouteTables",
        "ec2:DetachVolume",
        "ec2:ModifyVolume",
        "ec2:DescribeLaunchTemplates",
        "ec2:CreateRouteTable",
        "ec2:DetachInternetGateway",
        "ec2:DeleteVpc",
        "ec2:DescribeAddresses",
        "ec2:DeleteTags",
        "ec2:DescribeDhcpOptions",
        "ec2:DescribeNetworkInterfaces",
        "ec2:CreateSecurityGroup",
        "ec2:ModifyVpcAttribute",
        "ec2:ModifyInstanceAttribute",
        "ec2:AuthorizeSecurityGroupEgress",
        "ec2:DescribeTags",
        "ec2:DeleteRoute",
        "ec2:DescribeLaunchTemplateVersions",
        "ec2:DescribeNatGateways",
        "ec2:AllocateAddress",
        "ec2:DescribeImages",
        "ec2:DeleteSecurityGroup"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ELB",
      "Effect": "Allow",
      "Action": [
        "elasticloadbalancing:ModifyListener",
        "elasticloadbalancing:SetLoadBalancerPoliciesForBackendServer",
        "elasticloadbalancing:CreateTargetGroup",
        "elasticloadbalancing:AddTags",
        "elasticloadbalancing:DeleteLoadBalancerListeners",
        "elasticloadbalancing:ModifyLoadBalancerAttributes",
        "elasticloadbalancing:CreateLoadBalancerPolicy",
        "elasticloadbalancing:CreateLoadBalancer",
        "elasticloadbalancing:DeleteTargetGroup",
        "elasticloadbalancing:SetLoadBalancerPoliciesOfListener",
        "elasticloadbalancing:DescribeTargetGroups",
        "elasticloadbalancing:DeleteListener",
        "elasticloadbalancing:DetachLoadBalancerFromSubnets",
        "elasticloadbalancing:RegisterTargets",
        "elasticloadbalancing:DeleteLoadBalancer",
        "elasticloadbalancing:DescribeLoadBalancers",
        "elasticloadbalancing:DescribeLoadBalancerPolicies",
        "elasticloadbalancing:ModifyTargetGroupAttributes",
        "elasticloadbalancing:DeregisterInstancesFromLoadBalancer",
        "elasticloadbalancing:RegisterInstancesWithLoadBalancer",
        "elasticloadbalancing:DeregisterTargets",
        "elasticloadbalancing:DescribeLoadBalancerAttributes",
        "elasticloadbalancing:DescribeTargetGroupAttributes",
        "elasticloadbalancing:ConfigureHealthCheck",
        "elasticloadbalancing:CreateListener",
        "elasticloadbalancing:DescribeListeners",
        "elasticloadbalancing:ApplySecurityGroupsToLoadBalancer",
        "elasticloadbalancing:AttachLoadBalancerToSubnets",
        "elasticloadbalancing:CreateLoadBalancerListeners",
        "elasticloadbalancing:DescribeTargetHealth",
        "elasticloadbalancing:ModifyTargetGroup"
      ],
      "Resource": "*"
    },
    {
      "Sid": "ECR",
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:InitiateLayerUpload",
        "ecr:ListImages",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:PutImage",
        "ecr:BatchGetImage",
        "ecr:DescribeImages",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:DescribeRepositories"
      ],
      "Resource": "*"
    },
    {
      "Sid": "AUTOSCALING",
      "Effect": "Allow",
      "Action": [
        "autoscaling:DeleteAutoScalingGroup",
        "autoscaling:DescribeScalingActivities",
        "autoscaling:CreateLaunchConfiguration",
        "autoscaling:DescribeAutoScalingGroups",
        "autoscaling:UpdateAutoScalingGroup",
        "autoscaling:CreateAutoScalingGroup",
        "autoscaling:DescribeLaunchConfigurations",
        "autoscaling:DeleteLaunchConfiguration"
      ],
      "Resource": "*"
    },
    {
      "Sid": "SSM",
      "Effect": "Allow",
      "Action": [
        "ssm:GetParametersByPath",
        "ssm:GetParameter",
        "ssm:DeleteParameter",
        "ssm:DescribeParameters",
        "ssm:GetParameters",
        "ssm:DeleteParameters",
        "ssm:PutParameter",
        "ssm:GetParameterHistory"
      ],
      "Resource": "*"
    },
    {
      "Sid": "Full",
      "Effect": "Allow",
      "Action": ["cloudformation:*", "eks:*"],
      "Resource": "*"
    },
    {
      "Sid": "KMS",
      "Effect": "Allow",
      "Action": ["kms:DescribeKey"],
      "Resource": "*"
    },
    {
      "Sid": "IAMGetRole",
      "Effect": "Allow",
      "Action": ["iam:GetRole"],
      "Resource": "*"
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

**dry run of cluster creation**


```
eksctl create cluster --name k8slearning  --version 1.19 --nodegroup-name pool1 --node-type t2.micro --region us-east-1 --dry-run
```

**Note**: Dry run will not create any resources instead it will just show/print what resources it is going to create.

**cluster creation**

```
eksctl create cluster --name k8slearning  --version 1.19 --nodegroup-name pool1 --node-type t2.micro --region us-east-1
```

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

## Deleting the cluster

eksctl delete cluster --region=`<region`> --name=`<cluster`>