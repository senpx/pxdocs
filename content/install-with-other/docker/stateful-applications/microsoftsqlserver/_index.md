---
title: Microsoft SQL Server
keywords: portworx, px-developer, microsoft sql server, database, cluster, storage
description: Follow this guide to learn how use Microsoft SQL Server with Portworx.
weight: 2
noicon: true
---


This page provides instructions for deploying Microsoft SQL Server \(MS-SQL\) with Portworx on Kubernetes.

Use these [Kubernetes specs](https://github.com/portworx/px-docs/tree/gh-pages/k8s-samples/mssql) to deploy MS-SQL.

### Create a HA volume for MS-SQL

To create a highly available storage volume for MS-SQL, without having to provision storage in advance, run this command:

```text
docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=P@ssw0rd' \
      -p 1433:1433 --volume-driver=pxd \
      -v name=mssqlvol,size=10,repl=3:/var/opt/mssql \
      -d microsoft/mssql-server-linux
```

This command runs `mssql-server-linux` with a 10 GB volume created dynamically with three-way replication, guaranteeing that persistent data will be fully replicated on three separate nodes.

### Connect to MS-SQL server

You can now connect to MS-SQL in its container on port 1433. For example, to access the MS-SQL command prompt via `docker`, run:

```text
# docker exec -it <Container ID> /opt/mssql-tools/bin/sqlcmd \
 -S localhost -U SA -P "P@ssw0rd" '
```

{{<info>}}You can run multiple instances of MS-SQL on the same host, each with its own unique persistent volume mapped, and each with its own unique IP address published.{{</info>}}

### Database recoverability using Snapshots

To take a recoverable snapshot of the `mssql-server` instance for a point in time, use `pxctl` as follows:

```text
# pxctl snap create mssqlvol --name mssqlvol_snap_0628
Volume successfully snapped: 342580301989879504
pxctl snap list
ID			NAME			SIZE	HA	SHARED	ENCRYPTED	IO_PRIORITY	SCALE	STATUS
342580301989879504	mssqlvol_snap_0628	10 GiB	3	no	no		LOW		0	up - detached
```

### Let's sum things up

By default, a Portworx volume snapshot is read-writable. The snapshot taken is visible globally throughout the cluster, and can be used to start another instance of MS-SQL on a different node.

First, list our cloud snapshots:

```text
# pxctl snap list
ID			NAME			SIZE	HA	SHARED	ENCRYPTED	IO_PRIORITY	SCALE	STATUS
342580301989879504	mssqlvol_snap_0628	10 GiB	3	no	no		LOW		0	up - detached
```

Next, we would want to run the MS-SQL server:

```
# docker run -e 'ACCEPT_EULA=Y' -e 'SA_PASSWORD=P@ssw0rd' \
>       -p 1433:1433 --volume-driver=pxd \
>       -v mssqlvol_snap_0628:/var/opt/mssql \
>       -d microsoft/mssql-server-linux
```

Finally, let's make sure our server is up and running:

```
# docker ps
CONTAINER ID        IMAGE                          COMMAND                  CREATED             STATUS              PORTS                    NAMES
46eff5a9cbd6        microsoft/mssql-server-linux   "/bin/sh -c /opt/mssq"   4 minutes ago       Up 4 minutes        0.0.0.0:1433->1433/tcp   compassionate_perlman
0636d98250c4        portworx/px-dev                "/docker-entry-point."   2 hours ago         Up 2 hours                                   portworx.service
jeff-coreos-2 core # docker inspect --format '{{ .Mounts }}' 46eff5a9cbd6
[{mssqlvol_snap_0628 /var/lib/osd/mounts/mssqlvol_snap_0628 /var/opt/mssql pxd  true rprivate}]
```

For futher reading on running an MS-SQL container image with Docker, refer to [this](https://docs.microsoft.com/en-us/sql/linux/quickstart-install-connect-docker?view=sql-server-linux-2017#a-idpersista-persist-your-data) page.