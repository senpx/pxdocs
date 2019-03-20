---
title: "Install Portworx on kubernetes via Helm"
linkTitle: Helm
keywords: portworx, container, Kubernetes, storage, Docker, k8s, flexvol, pv, persistent disk
description: "Find out how to install PX on Kubernetes via the Portworx Helm chart"
weight: 2
hidden: true
---

The helm chart (portworx) deploys Portworx and [Stork](https://docs.portworx.com/scheduler/kubernetes/stork.html) in your Kubernetes cluster.

## Pre-requisites

* Helm has been installed on the client machine from where you would install the chart. (https://docs.helm.sh/using_helm/#installing-helm)
* Tiller version 2.9.0 and above
* Portworx pre-requisites [here](/start-here-installation/#installation-prerequisites)

## Install

To install Portworx via the chart with the release name `my-release` run the following commands.

First clone the Portworx helm chart repo.

```text
git clone https://github.com/portworx/helm.git
```

Now install the chart and substitute relevant values for your setup.

{{<info>}}`clusterName` should be a unique name identifying your Portworx cluster. The default value is `mycluster`, but it is suggested to update it with your naming scheme.{{</info>}}

For eg:

```text
helm install --debug --name my-release --set etcdEndPoint=etcd:http://192.168.70.90:2379,clusterName=$(uuidgen) ./helm/charts/portworx/
```

Refer to all the configuration options while deploying Portworx via the Helm chart:
[Configurable Options](https://github.com/portworx/helm/tree/master/charts/portworx#configuration)

## Uninstall

Below are the steps to wipe your entire Portworx installation.

1. Run cluster-scoped wipe command below. This has to be run from the client machine which has kubectl access.

    ```text
    curl -fsL https://install.portworx.com/px-wipe | bash
    ```
2. Delete the helm release

    ```text
    helm delete <release name> --purge
    ```

## Post-Install

Once you have a running Portworx installation, below sections are useful.

{{<homelist series2="k8s-postinstall">}}

## Troubleshooting helm installation failures

Refer to the common troubleshooting instructions for Portworx deployments via Helm [Troubleshooting portworx installation](https://github.com/portworx/helm/tree/master/charts/portworx#basic-troubleshooting)
