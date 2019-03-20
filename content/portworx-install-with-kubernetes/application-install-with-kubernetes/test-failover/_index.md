---
title: Test Failover of a MySQL pod on Portworx
linkTitle: Test Failover of a MySQL pod
keywords: portworx, failover, container, Kubernetes, storage, Docker, k8s, flexvol, pv, persistent disk
description: Learn how to failover a stateful application using Kubernetes and Portworx.  Try it for yourself today.
noicon: true
---


Failover MySQL Pod to a different node

## Show Database {#show-database}

```text
export MYSQLPOD=$(kubectl get pods -l app=mysql --no-headers | awk '{print $1}')
kubectl logs $MYSQLPOD
kubectl exec -ti $MYSQLPOD -- bash
mysql --user=root --password=password
show databases;
exit
exit
```

## Create a database {#create-a-database}

```text
export MYSQLPOD=$(kubectl get pods -l app=mysql --no-headers | awk '{print $1}')
kubectl logs $MYSQLPOD
kubectl exec -ti $MYSQLPOD -- bash
mysql --user=root --password=password
create database TEST_1234;
show databases;
exit
exit
```

## Let’s find what node the mysql pod is running {#lets-find-what-node-the-mysql-pod-is-running}

```text
export MYSQL_NODE=$(kubectl describe pod -l app=mysql | grep Node: | awk -F'[ \t//]+' '{print $2}')
echo $MYSQL_NODE
```

## Mark node as unschedulable. {#mark-node-as-unschedulable}

```text
kubectl cordon $MYSQL_NODE
```

## Delete the pod. {#delete-the-pod}

```text
kubectl delete pod -l app=mysql
```

## Verify the pod has moved to a different node {#verify-the-pod-has-moved-to-a-different-node}

```text
kubectl describe pods -l app=mysql
```

## Verify we can see the database we created {#verify-we-can-see-the-database-we-created}

```text
export MYSQLPOD=$(kubectl get pods -l app=mysql --no-headers | awk '{print $1}')
kubectl logs $MYSQLPOD
kubectl exec -ti $MYSQLPOD -- bash
mysql --user=root --password=password
show databases;
exit
exit
```

## Clean up {#clean-up}

### Bring the node back online {#bring-the-node-back-online}

```text
kubectl uncordon $MYSQL_NODE
```

### Delete database {#delete-database}

```text
export MYSQLPOD=$(kubectl get pods -l app=mysql --no-headers | awk '{print $1}')
kubectl logs $MYSQLPOD
kubectl exec -ti $MYSQLPOD -- bash
mysql --user=root --password=password
drop database TEST_1234;
show databases;
exit
```

{{% content "portworx-install-with-kubernetes/application-install-with-kubernetes/shared/discussion-forum.md" %}}
