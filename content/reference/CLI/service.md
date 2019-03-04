---
title: Service operations using pxctl
linkTitle: Service
keywords: portworx, pxctl, command-line tool, cli, reference
description: How to use PX CLI service.
weight: 9
---

```text
sudo /opt/pwx/bin/pxctl service --help
NAME:
   pxctl service - Service mode utilities

USAGE:
   pxctl service command [command options] [arguments...]

COMMANDS:
     audit, a        Audit the PX node
     call-home       Enable or disable the call home feature
     diags, d        creates a new tgz package with minimal essential diagnostic information.
     drive           Storage drive maintenance
     email           Email setting commands
     exit, e         Stop the PX daemon
     info, i         Show PX module version information
     kvdb, k         PX Kvdb operations
     maintenance, m  Maintenance mode operations
     node-wipe, nw   Wipes PX configuration data on this node
     pool            Storage pool maintenance

OPTIONS:
   --help, -h  show help
```

### pxctl service audit

Audit the PX node

```text
sudo /opt/pwx/bin/pxctl service audit
AuditID		Error	Message
kvdb-limits	none	KV limits audit not yet available

kvdb-response	none	KV response audit not yet available
```

### pxctl service call-home

You can use this command to enable and disable the call home feature

```text
sudo /opt/pwx/bin/pxctl service call-home --help
NAME:
   pxctl service call-home - Enable or disable the call home feature

USAGE:
   pxctl service call-home [arguments...]
```

```text
sudo /opt/pwx/bin/pxctl service call-home enable
Call home feature successfully enabled
```

### pxctl service diags

When there is an operational failure, you can use pxctl service diags &lt;name-of-px-container&gt; to generate a complete diagnostics package. This package will be automatically uploaded to Portworx. Additionally, the service package can be mailed to Portworx at support@portworx.com. The package will be available at /tmp/diags.tgz inside the PX container. You can use docker cp to extract the diagnostics package.

```text
sudo /opt/pwx/bin/pxctl service diags --help
NAME:
   pxctl service diags - creates a new tgz package with minimal essential diagnostic information.

USAGE:
   pxctl service diags [command options] [arguments...]

OPTIONS:
   -l, --live                gets diags from running px
   -u, --upload              upload diags to cloud
   -p, --profile             only dump profile
   -a, --all                 creates a new tgz package with all the available diagnostic information.
   -c, --cluster             generate diags for all the nodes in the cluster.
   -f, --force               force overwrite existing diags.
   -o, --output string       output file name (default "/var/cores/diags.tar.gz")
       --dockerhost string   docker host daemon (default "unix:///var/run/docker.sock")
       --container string    PX container ID
   -n, --node string         generate diags for a specific remote node with the provided NodeIp or NodeID.
```

```text
sudo /opt/pwx/bin/pxctl service diags --container px-enterprise
PX container name provided:  px-enterprise
INFO[0000] Connected to Docker daemon.  unix:///var/run/docker.sock
Getting diags files...
Generated diags: /tmp/diags.tar.gz
```

### pxctl service info

Displays all Version info

```text
sudo /opt/pwx/bin/pxctl service info
PX (OCI) Version:  2.0.2.1-1d83ac2
PX (OCI) Build Version:  1d83ac2baeb27451222edcd543249dd2c2f941e4
PX Kernel Module Version:  72D3C244593F45167A6B49D
```

### pxctl service logs

Displays the pxctl logs on the system

```text
sudo /opt/pwx/bin/pxctl service logs --help
NAME:
   pxctl service logs - Display PX logs

USAGE:
   pxctl service logs [arguments...]
```

### pxctl service kvdb

kvdb command is used for confguring kvdb

```text
sudo /opt/pwx/bin/pxctl service kvdb --help
NAME:
   pxctl service kvdb - PX Kvdb operations

USAGE:
   pxctl service kvdb [command options] [arguments...]

OPTIONS:
   endpoints    List the kvdb client endpoints
   members      List the kvdb cluster members
   restore      Restore keys and values into kvdb from a kvdb.dump file
```

### pxctl service maintenance

Service maintenance command lets the cluster know that it is going down for maintenance. Once the server is offline you can add/remove drives add memory etc…

```text
sudo /opt/pwx/bin/pxctl service maintenance --help
NAME:
   pxctl service maintenance - Maintenance mode operations

USAGE:
   pxctl service maintenance [command options] [arguments...]

OPTIONS:
   -x, --exit   exit maintenance mode
   -e, --enter  enter maintenance mode
   -c, --cycle  cycle maintenance mode
```

```text
sudo /opt/pwx/bin/pxctl service maintenance --enter
This is a disruptive operation, PX will restart in maintenance mode.
Are you sure you want to proceed ? (Y/N): y
```

### pxctl service drive

You can manage the physical storage drives on a node using the pxctl service drive sub menu.

```text
sudo /opt/pwx/bin/pxctl service drive
NAME:
   pxctl service drive - Storage drive maintenance

USAGE:
   pxctl service drive command [command options] [arguments...]

COMMANDS:
     add            Add storage
     check          Check drives
     rebalance      Rebalance storage
     replace        Replace source drive with target drive
     show           Show drives

OPTIONS:
   --help, -h  show help
```

You can add drives to a server using the /opt/pwx/bin/pxctl service drive add command. To do so the server must be in maintenance mode.

```text
sudo /opt/pwx/bin/pxctl service drive add --help
NAME:
   pxctl service drive add - Add storage

USAGE:
   pxctl service drive add [arguments...]
```

```text
sudo /opt/pwx/bin/pxctl service drive add /dev/mapper/volume-3bfa72dd
Adding device  /dev/mapper/volume-3bfa72dd ...
Drive add  successful. Requires restart (Exit maintenance mode).
```

To rebalance the storage across the drives, use pxctl service drive rebalance. This is useful after prolonged operation of a node.

### pxctl service drive show

You can use pxctl service drive show to display drive information on the server

```text
sudo /opt/pwx/bin/pxctl service drive show
PX drive configuration:
Pool ID: 0
	IO_Priority: LOW
	Size: 100 GiB
	Status: Online
	Has meta data: Yes
	Drives:
	1: /dev/mapper/volume-e85a42ca, 1.0 GiB allocated of 100 GiB, Online
```

### pxctl service email

Email setting commands

```text
sudo /opt/pwx/bin/pxctl service email
NAME:
   pxctl service email

USAGE:
   pxctl service email command [command options] [arguments...]

COMMANDS:
     clear     Clear email settings for alerts.
     set       Configure email settings for alerts.
```

### pxctl service scan

You can use pxctl service scan to scan for bad blocks on a drive

```text
sudo /opt/pwx/bin/pxctl service scan
NAME:
   pxctl service scan - scan for bad blocks

USAGE:
   pxctl service scan command [command options] [arguments...]

COMMANDS:
     cancel    cancel running scan
     pause     pause running scan
     resume    resume paused scan
     schedule  examine or set schedule
     start     start scan
     status    scan status

```

### pxctl service node-wipe

pxctl service node-wipe deletes all data related to Portworx from the node. It will also wipe the storage device that was provided to Portworx. This command can be run only when Portworx is stopped on the node. Run this command if a node needs to be re-initialized.

{{<info>}}
**Note:** This is a disruptive command and could lead to data loss. Please use caution.
{{</info>}}

```text
sudo /opt/pwx/bin/pxctl service node-wipe --help
NAME:
   pxctl service node-wipe - Wipes PX configuration data on this node

USAGE:
   pxctl service node-wipe [command options] [arguments...]

OPTIONS:
   --storage_devices value, -s value  comma-separated list of storage devices to be wiped.

```

Here is an example:

```text
sudo /opt/pwx/bin/pxctl service node-wipe
This is a distruptive operation.
It will delete all PX configuration files from this node. Data on the storage disks attached on this node will be irrevocably deleted.
Are you sure you want to proceed ? (Y/N): y
This is a distruptive operation.
It will delete all PX configuration files from this node. Data on the storage disks attached on this node will be irrevocably deleted.
Failed to set pxd timeout. Wipe command might take more time to finish.
Are you sure you want to wipe data from the disk: ' /dev/sdb ' (Y/N): y
/dev/sdb: 8 bytes were erased at offset 0x00010040 (btrfs): 5f 42 48 52 66 53 5f 4d
Removed PX footprint from device /dev/sdb.
Wiped node successfully.
```

### pxctl service pool

Pool maintenance

```text
sudo /opt/pwx/bin/pxctl service pool
NAME:
   pxctl service pool - Storage pool maintenance

USAGE:
   pxctl service pool command [command options] [arguments...]

COMMANDS:
   show      Show pools
   update    Update pool properties
```

### pxctl service pool update

Updates the pool properties

```text
sudo pxctl service pool update --help
```

```
Update pool properties

Usage:
  pxctl service pool update [flags]

Flags:
      --resize               extend pool to maximum available physical storage
      --io_priority string   IO Priority (Valid Values: [high medium low]) (default "low")
      --labels string        comma separated name=value pairs (default "NoLabel")
  -h, --help                 help for update
```