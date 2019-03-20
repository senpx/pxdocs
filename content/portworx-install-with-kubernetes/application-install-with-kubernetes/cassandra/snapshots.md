---
title: Cassandra Snapshots
keywords: portworx, container, Kubernetes, storage, Docker, k8s, flexvol, pv, persistent disk, snapshots, stork, clones
description: Learn to take a snapshot of Cassandra volumes on Kubernetes
linkTitle: Snapshots
---

## Managing Snapshots {#managing-snapshots}

Cassandra snapshots first flush application memory, then create a hardlink to the `SSTable` files. This means that the snaps are application consistent \(mem is flushed\) but the snap data itself is still within the volume, so if something were to happen to the underlying volume, you still have a corrupted volume and can’t properly roll back. So these snaps are useful to going back to a point in time

PX snaps create a real usable volume which is distinct and separate from the volume cassandra is currently using. That means you can can standup a parallel second instance of cassandra from that volume and so on. However it is crash consistent \(cassandra’s memory is not flushed\)

### Best Practice {#best-practice}

It is recommended to use 3DSnaps for cassandra as they are application-consistent.

* [3DSnaps overview](/portworx-install-with-kubernetes/storage-operations/create-snapshots/snaps-3d)
* [Cassandra example for 3DSnaps](/portworx-install-with-kubernetes/storage-operations/create-snapshots/snaps-3d/#cassandra)

{{% content "portworx-install-with-kubernetes/application-install-with-kubernetes/shared/discussion-forum.md" %}}
