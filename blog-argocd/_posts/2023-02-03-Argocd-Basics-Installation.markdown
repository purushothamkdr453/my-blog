---
layout: post
title:  "What is Argocd? Argocd Basics & Architecture?"
date:   2023-02-03 04:00:00 +0530
category: argocd
---

- [argocd](#argocd)
   - [Overview](#overview)
   - [Architecture](#architecture)
   - [Features](#features)
   
## argocd

## Overview

`argocd` is a declarative & continuous delivery tool for kubernetes. Argocd follows `Gitops principles`. Following are the Gitops principles.

- Declarative Configuration
- Version Controlled, Immutable Storage
- Automatic Pull Operations
- Continuous Reconciliation

Argocd also follows all the above mentioned principles. Lets understand more in details how argocd follows each one of the above principle.

**Declarative configuration** - All the configuration related to argocd(argocd apps, argocd projects etc etc) will be defined declaratively.
<br/>
**Version Controllerd** - Everything related to Argocd will be stored in Source code repository(for ex: Github).
<br/>
**Automatic Pull operations** - argocd pulls latest configuration changes from sourcecode tool whenever there is a change made to it(source code configuration). 
<br/>
**Continuous Reconciliation** - Argocd continuously checks the k8s cluster and if there is any change introduced manually then that change will be reverted by argocd.


## Architecture

![alt text](/assets/images/argocd-architecture.png)

Argo CD is implemented as a kubernetes controller which continuously monitors running applications and compares the current, live state against the desired target state (as specified in the Git repo). A deployed application whose live state deviates from the target state is considered OutOfSync. Argo CD reports & visualizes the differences, while providing facilities to automatically or manually sync the live state back to the desired target state. Any modifications made to the desired target state in the Git repo can be automatically applied and reflected in the specified target environments.

**components**

**API Server**

The API server is a gRPC/REST server which exposes the API consumed by the Web UI, CLI, and CI/CD systems. It has the following responsibilities:

- application management and status reporting
- invoking of application operations (e.g. sync, rollback, user-defined actions)
- repository and cluster credential management (stored as K8s secrets)
- authentication and auth delegation to external identity providers
- RBAC enforcement
- listener/forwarder for Git webhook events

**Repository Server**

The repository server is an internal service which maintains a local cache of the Git repository holding the application manifests. It is responsible for generating and returning the Kubernetes manifests when provided the following inputs:

repository URL
revision (commit, tag, branch)
application path
template specific settings: parameters, helm values.yaml


**Application controller**

The application controller is a Kubernetes controller which continuously monitors running applications and compares the current, live state against the desired target state (as specified in the repo). It detects OutOfSync application state and optionally takes corrective action. It is responsible for invoking any user-defined hooks for lifecycle events (PreSync, Sync, PostSync).

## Features

- Automated deployment of applications to specified target environments
- Support for multiple config management/templating tools (Kustomize, Helm, Jsonnet, plain-YAML)
- Ability to manage and deploy to multiple clusters
- SSO Integration (OIDC, OAuth2, LDAP, SAML 2.0, GitHub, GitLab, Microsoft, LinkedIn)
- Multi-tenancy and RBAC policies for authorization
- Rollback/Roll-anywhere to any application configuration committed in Git repository
- Health status analysis of application resources
- Automated configuration drift detection and visualization
- Automated or manual syncing of applications to its desired state
- Web UI which provides real-time view of application activity
- CLI for automation and CI integration
- Webhook integration (GitHub, BitBucket, GitLab)
- Access tokens for automation
- PreSync, Sync, PostSync hooks to support complex application rollouts (e.g.blue/green & canary upgrades)
- Audit trails for application events and API calls
- Prometheus metrics
- Parameter overrides for overriding helm parameters in Git


We will be taking more about all the above features in the subsequent blogs.