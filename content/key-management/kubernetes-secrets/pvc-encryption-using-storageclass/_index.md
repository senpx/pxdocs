---
title: Encrypting PVCs using StorageClass with Kubernetes Secrets
weight: 1
keywords: Portworx, Kubernetes, Kubernetes Secrets, containers, storage, encryption
description: Instructions on using Kubernetes Secrets with Portworx for encrypting PVCs using StorageClass
noicon: true
series: kubernetes-secret-uses
series2: k8s-pvc-enc
hidden: true
---

{{% content "key-management/shared/intro.md" %}}

Using a Storage Class parameter, you can tell Portworx to encrypt all PVCs created using that Storage Class. Portworx uses a cluster wide secret to encrypt all the volumes created using the secure Storage Class.

#### Step 1: Create cluster wide secret key
A cluster wide secret key is a common key that points to a secret value/passphrase which can be used to encrypt all your volumes.

Create a cluster wide secret in Kubernetes, if not already created:
```text
$ kubectl -n portworx create secret generic px-vol-encryption \
  --from-literal=cluster-wide-secret-key=<value>
```
Note that the cluster wide secret has to reside in the `px-vol-encryption` secret under the `portworx` namespace.

Now you have to give Portworx the cluster wide secret key, that acts as the default encryption key for all volumes.
```text
$ PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
$ kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl secrets set-cluster-key \
  --secret cluster-wide-secret-key
```

{{% content "key-management/shared/storage-class-encryption.md" %}}