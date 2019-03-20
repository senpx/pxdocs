---
title: Shared
hidden: true
description: Learn how to install Portworx with Kubenetes
keywords: portworx, kubernetes
---

### Apply the specs

Apply the generated specs in your cluster.
```text
kubectl apply -f px-spec.yaml
```

#####  Monitor the portworx pods

Wait till all portworx pods show as ready in the below output.
```text
kubectl get pods -o wide -n kube-system -l name=portworx
```

#####  Monitor Portworx cluster status

```text
PX_POD=$(kubectl get pods -l name=portworx -n kube-system -o jsonpath='{.items[0].metadata.name}')
kubectl exec $PX_POD -n kube-system -- /opt/pwx/bin/pxctl status
```

## Post-Install

Once you have a running Portworx installation, below sections are useful.

{{<homelist series2="k8s-postinstall">}}
