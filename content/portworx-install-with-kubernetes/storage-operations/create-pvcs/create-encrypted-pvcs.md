---
title: Create encrypted PVCs
weight: 4
keywords: portworx, container, kubernetes, storage, k8s, pv, persistent disk, encryption, pvc
description: This guide is a step-by-step tutorial on how to provision encrypted PVCs with Portworx.
series: k8s-vol
---

## Volume encryption

{{% content "shared/encryption/intro.md" %}}

Use the following steps to get started with encrypted PVCs

### Step 1: Select a secrets provider

Select one of the following secret providers to store your passphrases. This passphrase will then be used for encrypting the PVCs. If you have already setup a secrets provider, goto [Step 2](#step-2-select-pvc-encryption-method)

{{<homelist series="key-management">}}

### Step 2: Select PVC encryption method

{{<homelist series2="k8s-pvc-enc">}}
