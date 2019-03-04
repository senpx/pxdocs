---
title: Install on Kubernetes with Docker EE
keywords: portworx, container, storage, Docker, kubernetes, k8s
description: Learn how to use Portworx to provide storage for your stateful services running on Kubernetes with Docker EE.
weight: 2
noicon: true
series: px-docker-install
---

This document explains how to install Portworx with Kubernetes on Docker EE 2.x.

## Prerequisites

You must have Docker EE 2.x running with configured Kubernetes cluster.

{{<info>}}**Non-Kubernetes users:** To install stand-alone Portworx on Docker EE 2.x follow this [doc](/install-with-other/docker/standalone/).{{</info>}}

## Install Docker EE 2.x

Follow Docker documentation to install Docker EE 2.x https://docs.docker.com/install/linux/docker-ee/centos.

## Deploy UCP

#### Select UCP version

Follow Docker documentation to install UCP https://docs.docker.com/ee/ucp/admin/install.

{{<info>}}Note that UCP version must be 3.1.x or higher for RBAC compitability https://docs.docker.com/ee/ucp/release-notes.{{</info>}}

#### Install UCP

Here is an example command to install UCP 3.1.2.

```text
docker image pull docker/ucp:3.1.2
```

```text
docker container run --rm -it --name ucp -v /var/run/docker.sock:/var/run/docker.sock docker/ucp:3.1.2 install --host-address <node-ip> --interactive
```

{{<info>}}NOTE: Do not init swarm, UCP with do that for you.{{</info>}}

#### Login to UCP

Use the credentials to login to UCP Dashboard, example: admin/password.
```text
https://<node-ip>:443
```

## Configure kubernetes environment

#### Check your Kubernetes version

Navigate to Admin -> About -> Kubernetes and look for GoVersion.
```text
https://<node-ip>/manage/about/kubernetes
```
![Get K8S Version](/img/docker-ee-k8s1.png)

#### Install kubectl

Follow Kubernetes documentation to install kubectl package https://kubernetes.io/docs/tasks/tools/install-kubectl/#install-kubectl.

#### Generate new client bundle

Navigate to Admin -> My Profile -> Client Bundles and select Generate New Client Bundle from dropdown menu.

```text
https://<node-ip>/manage/profile/clientbundle
```
![Generate New Client Bundle](/img/docker-ee-k8s2.png)

#### Download Client Bundle and set env

Install unzip and use it to unpackage bundle.

```text
yum install -y unzip
unzip ucp-bundle-admin.zip
eval "$(<env.sh)"
```

Now use kubectl to get nodes.

```text
kubectl get nodes -o wide
```

## Install PX

{{% content "portworx-install-with-kubernetes/shared/1-generate-the-spec-footer.md" %}}

{{% content "portworx-install-with-kubernetes/shared/4-apply-the-spec.md" %}}
