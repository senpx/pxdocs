---
title: Portworx on Rancher 2.x
linkTitle: Install Portworx on Rancher 2.x
keywords: portworx, PX-Developer, container, Rancher, storage
description: Instructions on installing Portworx using public catalog (Helm Chart) on Rancher 2.x 
weight: 6
series: px-rancher
noicon: true
---

This section covers information on installing Portworx using public catalog (Helm Chart) on Rancher 2.x.

## Step 1: Install Rancher

Follow the instructions for installing [Rancher 2.x](https://rancher.com/docs/rancher/v2.x/en/installation/).

## Step 2: Prerequisites - Choosing the right worker node flavor for your Rancher 2.x Kubernetes cluster for Portworx

Portworx is a highly available software-defined storage solution that you can use to manage persistent storage for your containerized databases or other stateful apps in your Rancher 2.x Kubernetes cluster. To make sure that your cluster is set up with the compute resources that are required for Portworx, review the FAQs in this step.

Portworx pre-requisites [here](/start-here-installation/#installation-prerequisites)

**NOTE:** </br>
 
{{<info>}}

* Currently `RancherOS` distro is not supported for Portworx. 

* Portworx requires that Rancher hosts have at least one non-root disk or partition to contribute.{{</info>}}

**How can I make sure that my data is stored highly available?** </br>
You need at least 3 worker nodes in your Portworx cluster so that Portworx can replicate your data across nodes. By replicating your data across worker nodes, Portworx can ensure that your stateful app can be rescheduled to a different worker node in case of a failure without losing data. 

## Step 3: Creating or preparing your cluster for Portworx

To install Portworx, you must have an Rancher Kubernetes cluster that runs Kubernetes version 1.10 or higher. To make sure that your cluster is set up with worker nodes that offer best performance for you Portworx cluster, review Step 1: Choosing the right worker node flavor for your Rancher cluster for Portworx.

Every Portworx cluster must be connected to a key-value store to store Portworx metadata. The Portworx key-value store serves as the single source of truth for your Portworx storage layer. If the key-value store is not available, then you cannot work with your Portworx cluster to access or store your data. Existing data is not changed or removed when the Portworx database is unavailable.

## Step 4: Installing Portworx on Rancher 2.x

Portworx provides a helm chart for Rancher 2.x that is available in the public catalog. The Helm chart deploys a trial version of the Portworx enterprise edition `px-enterprise` that you can use for 30 days. After the trial version expires, you must [purchase a Portworx license](/reference/knowledge-base/px-licensing/) to continue to use your Portworx cluster. In addition, [Stork](https://docs.portworx.com/portworx-install-with-kubernetes/) is installed on your Kubernetes cluster. Stork is the Portworx storage scheduler and allows you to co-locate pods with their data, and create and restore snapshots of Portworx volumes.

* To install the Portworx Helm chart, navigate to your cluster, select System namespace.  Search for Portworx catalog in the load application section and select View Details to start the Helm chart form.  The contents of the answer file are located in the appendix called answer.yml.

![Get K8S Version](/img/px-rancher-1.png)

**KEY VALUE STORE PARAMETERS** 
Under configuration enter your Etcd information.  This is a list separated by semicolons ie: For example 
`etcd:http://70.0.99.40:2379;etcd:http://70.0.99.51:2379;etcd:http://70.0.99.41:2379`  Or select the internal KVDB option.

![Get K8S Version](/img/px-rancher-2.png)

**STORAGE PARAMETERS**
In your environment set the drives field to the LVM that you will be using for Portworx storage.   A recommended practice is to add a separate SSD block device as a Journal drive.  If you have one available enter auto in the Journal device section.

**NETWORK PARAMETERS**
In your environment, you will put the interface dedicated to Portworx traffic in the Data Network Interface field, and enter the Kubernetes host interface in the Management Network Interface.

**ADVANCED PARAMETERS**

Set the Install Stork and Lighthouse fields to true.  Define a Portworx Cluster Name that is relevant to your environment.  Set the following version information:

* Stork version: 2.1.0
* Lighthouse version:	2.0.2 
* Portworx version:	2.0.3.1

Once completed with the form select Launch.  Depending on your Internet speed and the performance of your systems it will take 5-20 minutes to install.  Once completed all process for Portworx will be green.

![Get K8S Version](/img/px-rancher-3.png)

## Step 5: Adding Portworx storage to your apps

Now that your Portworx cluster is all set, you can start creating Portworx volumes by using [Kubernetes dynamic provisioning](https://kubernetes.io/docs/concepts/storage/dynamic-provisioning/). The Portworx Helm chart already set up a few default storage classes in your cluster that you can see by running the `kubectl get storageclasses | grep portworx` command. You can also create your own storage class to define settings, such as:

- Encryption for a volume
- IO priority of the disk where you want to store the data
- Number of data copies that you want to store across worker nodes
- Sharing of volumes across pods

For more information about how to create your own storage class and add Portworx storage to your app. For an overview of supported configurations in a PVC, see [Using Dynamic Provisioning](https://docs.portworx.com/portworx-install-with-kubernetes/#using-dynamic-provisioning).

## What's next?
Now that you set up Portworx on your Kubernetes cluster, you can explore the following features:

- **Use existing Portworx volumes:** If you have an existing Portworx volume that you created manually or that was not automatically deleted when you deleted the PVC, you can statically provision the corresponding PV and PVC and use this volume with your app. For more information, see [Using existing volumes](https://docs.portworx.com/portworx-install-with-kubernetes/#using-the-portworx-volume).
- **Running stateful sets on Portworx:** If you have a stateful app that you want to deploy as a stateful set into your cluster, you can set up your stateful set to use storage from your Portworx cluster. For more information, see [Create a mysql StatefulSet](https://docs.portworx.com/portworx-install-with-kubernetes/#create-a-mysql-statefulset).
- **Running your pods hyperconverged:** You can configure your Portworx cluster to schedule pods on the same worker node where the pod's volume resides. This setup is also referred to as hyperconverged and can improve the data storage performance. For more information, see [Run pods on same host as a volume](https://docs.portworx.com/portworx-install-with-kubernetes/).
- **Creating snapshots of your Portworx volumes:** You can save the current state of a volume and its data by creating a Portworx snapshot. Snapshots can be stored on your local Portworx cluster or in the Cloud. For more information, see [Create and use local snapshots](https://docs.portworx.com/portworx-install-with-kubernetes/).
