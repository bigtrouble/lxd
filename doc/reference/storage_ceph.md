(storage-ceph)=
# Ceph RBD - `ceph`

<!-- Include start Ceph intro -->

```{youtube} https://youtube.com/watch?v=kVLGbvRU98A
```

[Ceph](https://ceph.io/) is an open-source storage platform that stores its data in a storage cluster based on {abbr}`RADOS (Reliable Autonomic Distributed Object Store)`.
It is highly scalable and, as a distributed system without a single point of failure, very reliable.

Ceph provides different components for block storage and for file systems.
<!-- Include end Ceph intro -->

Ceph {abbr}`RBD (RADOS Block Device)` is Ceph's block storage component that distributes data and workload across the Ceph cluster.
It uses thin provisioning, which means that it is possible to over-commit resources.

## Terminology

<!-- Include start Ceph terminology -->
Ceph uses the term *object* for the data that it stores.
The daemon that is responsible for storing and managing data is the *Ceph {abbr}`OSD (Object Storage Daemon)`*.
Ceph's storage is divided into *pools*, which are logical partitions for storing objects.
They are also referred to as *data pools*, *storage pools* or *OSD pools*.
<!-- Include end Ceph terminology -->

Ceph block devices are also called *RBD images*, and you can create *snapshots* and *clones* of these RBD images.

## `ceph` driver in LXD

```{note}
To use the Ceph RBD driver, you must specify it as `ceph`.
This is slightly misleading, because it uses only Ceph RBD (block storage) functionality, not full Ceph functionality.
For storage volumes with content type `filesystem` (images, containers and custom filesystem volumes), the `ceph` driver uses Ceph RBD images with a file system on top (see {ref}`block.filesystem <storage-ceph-vol-config>`).

Alternatively, you can use the {ref}`storage-cephfs` driver to create storage volumes with content type `filesystem`.
```
<!-- Include start Ceph driver cluster -->
Unlike other storage drivers, this driver does not set up the storage system but assumes that you already have a Ceph cluster installed.
<!-- Include end Ceph driver cluster -->

<!-- Include start Ceph driver remote -->
This driver also behaves differently than other drivers in that it provides remote storage.
As a result and depending on the internal network, storage access might be a bit slower than for local storage.
On the other hand, using remote storage has big advantages in a cluster setup, because all cluster members have access to the same storage pools with the exact same contents, without the need to synchronize storage pools.
<!-- Include end Ceph driver remote -->

The `ceph` driver in LXD uses RBD images for images, and snapshots and clones to create instances and snapshots.

<!-- Include start Ceph driver control -->
LXD assumes that it has full control over the OSD storage pool.
Therefore, you should never maintain any file system entities that are not owned by LXD in a LXD OSD storage pool, because LXD might delete them.
<!-- Include end Ceph driver control -->

Due to the way copy-on-write works in Ceph RBD, parent RBD images can't be removed until all children are gone.
As a result, LXD automatically renames any objects that are removed but still referenced.
Such objects are kept with a  `zombie_` prefix until all references are gone and the object can safely be removed.

### Limitations

The `ceph` driver has the following limitations:

Sharing custom volumes between instances
: Custom storage volumes with {ref}`content type <storage-content-types>` `filesystem` can usually be shared between multiple instances different cluster members.
  However, because the Ceph RBD driver "simulates" volumes with content type `filesystem` by putting a file system on top of an RBD image, custom storage volumes can only be assigned to a single instance at a time.
  If you need to share a custom volume with content type `filesystem`, use the {ref}`storage-cephfs` driver instead.

Sharing the OSD storage pool between installations
: Sharing the same OSD storage pool between multiple LXD installations is not supported.
  The only scenario in which LXD allows sharing an OSD storage pool between multiple LXD installations is if you want to recover existing instances via `lxd recover`.
  In this case, you must set the {ref}`ceph.osd.force_reuse <storage-ceph-pool-config>` configuration option to `true`.
  If this option is not set, LXD refuses to reuse an OSD storage pool that it detects as being in use by another LXD installation.

Choosing an underlying file system
: When setting up the Ceph cluster that LXD will use, choose XFS as the underlying file system for the storage entities that are used to hold OSD storage pools.
  Using ext4 as the underlying file system is [not recommended by Ceph upstream](https://docs.ceph.com/en/quincy/rados/troubleshooting/troubleshooting-osd/#filesystem-issues).
  You might see unexpected and erratic failures that are unrelated to LXD itself.

Using an OSD pool of type "erasure"
: To use a Ceph OSD pool of type "erasure", you must create the OSD pool beforehand.
  You must also create a separate OSD pool of type "replicated" that will be used for storing metadata.
  This is required because Ceph RBD does not support omap.
  To specify which pool is "erasure coded", set the {ref}`ceph.osd.data_pool_name <storage-ceph-pool-config>` configuration option to the erasure coded pool name and the {ref}`source <storage-ceph-pool-config>` configuration option to the replicated pool name.

## Configuration options

The following configuration options are available for storage pools that use the `ceph` driver and for storage volumes in these pools.

(storage-ceph-pool-config)=
### Storage pool configuration
Key                           | Type                          | Default                                 | Description
:--                           | :---                          | :------                                 | :----------
ceph.cluster\_name            | string                        | ceph                                    | Name of the Ceph cluster in which to create new storage pools
ceph.osd.data\_pool\_name     | string                        | -                                       | Name of the OSD data pool
ceph.osd.force\_reuse         | bool                          | false                                   | Force using an OSD storage pool that is already in use by another LXD instance
ceph.osd.pg\_num              | string                        | 32                                      | Number of placement groups for the OSD storage pool
ceph.osd.pool\_name           | string                        | name of the pool                        | Name of the OSD storage pool
ceph.rbd.clone\_copy          | bool                          | true                                    | Whether to use RBD lightweight clones rather than full dataset copies
ceph.rbd.du                   | bool                          | true                                    | Whether to use RBD `du` to obtain disk usage data for stopped instances
ceph.rbd.features             | string                        | layering                                | Comma-separated list of RBD features to enable on the volumes
ceph.user.name                | string                        | admin                                   | The Ceph user to use when creating storage pools and volumes
source                        | string                        | -                                       | Existing OSD storage pool to use
volatile.pool.pristine        | string                        | true                                    | Whether the pool was empty on creation time

(storage-ceph-vol-config)=
### Storage volume configuration
Key                     | Type      | Condition                 | Default                               | Description
:--                     | :---      | :--------                 | :------                               | :----------
block.filesystem        | string    | block based driver        | same as volume.block.filesystem       | {{block_filesystem}}
block.mount\_options    | string    | block based driver        | same as volume.block.mount\_options   | Mount options for block devices
security.shifted        | bool      | custom volume             | false                                 | {{enable_ID_shifting}}
security.unmapped       | bool      | custom volume             | false                                 | Disable ID mapping for the volume
size                    | string    | appropriate driver        | same as volume.size                   | Size/quota of the storage volume
snapshots.expiry        | string    | custom volume             | -                                     | {{snapshot_expiry_format}}
snapshots.pattern       | string    | custom volume             | snap%d                                | {{snapshot_pattern_format}}
snapshots.schedule      | string    | custom volume             | -                                     | {{snapshot_schedule_format}}