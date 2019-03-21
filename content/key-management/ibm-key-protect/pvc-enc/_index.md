---
title: Encrypting Kubernetes PVCs with IBM Key Protect
weight: 1
keywords: Portworx, IBM, IBM Key Protect, containers, storage, encryption
description: Instructions on using IBM Key Protect with Portworx for encrypting PVCs in Kubernetes
noicon: true
disableprevnext: true
series: ibm-key-protect-uses
series2: k8s-pvc-enc
hidden: true
---

{{% content "key-management/shared/intro.md" %}}


### Encryption using per volume secrets

In this method each volume will use its own unique passphrase to encrypt the volume. Portworx uses IBM Key Protect APIs to generate a unique 256 bit passphrase. This passphrase will be used during encryption and decryption.

#### Step 1: Create a Storage Class

{{% content "key-management/shared/enc-storage-class-spec.md" %}}

#### Step 2: Create a Persistent Volume Claim

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-data
  annotations:
    volume.beta.kubernetes.io/storage-class: px-secure-sc
spec:
  storageClassName: px-mysql-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

```

If you do not want to specify the `secure` flag in the storage class, but you want to encrypt the PVC using that Storage Class, then create the PVC as below

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: secure-pvc
  annotations:
    px/secure: "true"
spec:
  storageClassName: portworx-sc
  accessModes:
  - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi
```
Note the `px/secure: "true"` annotation on the PVC object.

### Encryption using cluster wide secret

In this method a default cluster wide secret will be set for the Portworx cluster. Such a secret will be referenced by the user and Portworx as **default** secret. Any PVC request referencing the secret name as `default` will use this cluster wide secret as a passphrase to encrypt the volume.

#### Step 1: Set the cluster wide secret key

{{% content "key-management/shared/set-cluster-wide-passphrase.md" %}}

#### Step 2: Create a Storage Class

{{% content "key-management/shared/enc-storage-class-spec.md" %}}

#### Step 3: Create a Persistent Volume Claim

```yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: mysql-data
  annotations:
    px/secret-name: default
    volume.beta.kubernetes.io/storage-class: px-secure-sc
spec:
  storageClassName: px-mysql-sc
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 2Gi

```

Take a note of the annotation `px/secret-name: default`. This specific annotation indicates Portworx to use the default secret to encrypt the volume. In this case it will **NOT**  create a new passphrase for this volume and NOT use per volume encryption. If the annotation is not provided then Portworx will use the per volume encryption workflow as described in the previous section.

Again, if your Storage Class does not have the `secure` flag set, but you want to encrypt the PVC using the same Storage Class, then add the annotation `px/secure: "true"` to the above PVC.

