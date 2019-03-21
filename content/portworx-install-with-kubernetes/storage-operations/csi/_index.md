---
title: Portworx with CSI
keywords: csi, portworx, container, Kubernetes, storage, Docker, k8s, pv, persistent disk
description: This page describes how to deploy Portworx with CSI
---

[CSI](https://kubernetes-csi.github.io/), or _Container Storage Interface_, is the new model for integrating storage system service with Kubernetes and other orchestration systems. Kubernetes has had support for CSI since 1.10 as beta.

With CSI, Kubernetes gives storage drivers the opportunity to release on their schedule. This allows storage vendors to upgrade, update, and enhance their drivers without the need to update Kubernetes, maintaining a consistent, dependable, orchestration system.

## Install

### Install using the Portworx spec generator

When [Generating the Portworx specs](https://install.portworx.com/2.0) select CSI under Customize->Advanced Settings. This will add the CSI components to the Portworx DaemonSet.

If you are using [curl to fetch the Portworx spec](/portworx-install-with-kubernetes/px-k8s-spec-curl), you can add `csi=true` to the parameter list to include CSI specs in the generated file.

{{<info>}}**Openshift users**: 
You will need to add the px-csi-account service account to the privileged security context.

```text
oc adm policy add-scc-to-user privileged system:serviceaccount:kube-system:px-csi-account
```
{{</info>}}


## Impact on applications

The only affected object is the StorageClass. For any StorageClasses created, you will need to setup the value of `provisioner` to `com.openstorage.pxd`. Here is an example:

```text
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: portworx-csi-sc
provisioner: com.openstorage.pxd
parameters:
  repl: "3"
```

To create a PersistentVolumeClaims, simply reference the above StorageClass.

```text
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
   name: px-mysql-pvc
spec:
   storageClassName: portworx-csi-sc
   accessModes:
     - ReadWriteOnce
   resources:
     requests:
       storage: 2Gi
```

## Upgrade

Currently upgrades are _not_ supported. You will need to deploy using CSI onto a new Kubernetes cluster. The Kubernetes community is working very hard to make this possible in the near future.

## Contribute

Portworx welcomes contributions to our CSI implemenation, which is open-source and repository is at [OpenStorage](https://github.com/libopenstorage/openstorage).
