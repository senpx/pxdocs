---
title: Encrypting Kubernetes PVCs with Vault
weight: 1
keywords: Portworx, Hashicorp, Vault, containers, storage, encryption
description: Instructions on using Vault with Portworx for encrypting PVCs in Kubernetes
noicon: true
series: vault-secret-uses
series2: k8s-pvc-enc
hidden: true
---

{{% content "key-management/shared/intro.md" %}}

There are two ways in which Portworx volumes can be encrypted and are dependent on how a secret passphrase is provided to PX.

### Encryption using Storage Class

In this method, PX will use the cluster wide secret key to encrypt PVCs.

#### Step 1: Set a cluster wide secret

{{% content "key-management/shared/set-cluster-wide-secret.md" %}}

{{% content "key-management/shared/storage-class-encryption.md" %}}

### Encryption using PVC annotations

In this method, each PVC can be encrypted with its own secret key.

#### Step 1: Create a Storage Class

{{% content "key-management/shared/enc-storage-class-spec.md" %}}

#### Step 2: Create a PVC with annotation

{{% content "key-management/shared/other-providers-pvc-encryption" %}}

__Important: Make sure secret `your_secret_key` exists in Vault__
