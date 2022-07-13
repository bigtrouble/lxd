(storage-lvm)=
# LVM - `lvm`

{abbr}`LVM (Logical Volume Manager)` is a storage management framework rather than a file system.
It is used to manage physical storage devices, allowing you to create a number of logical storage volumes that use and virtualize the underlying physical storage devices.

Note that it is possible to over-commit the physical storage in the process, to allow flexibility for scenarios where not all available storage is in use at the same time.

## Terminology

LVM can combine several physical storage devices into a *volume group*.
You can then allocate *logical volumes* of different types from this volume group.

One supported volume type is a *thin pool*, which allows over-committing the resources by creating  thinly provisioned volumes whose total allowed maximum size is larger than the available physical storage.
Another type is a *volume snapshot*, which captures a specific state of a logical volume.

## `lvm` driver in LXD

The `lvm` driver in LXD uses logical volumes for images, and volume snapshots for instances and snapshots.

LXD assumes that it has full control over the volume group.
Therefore, you should never maintain any file system entities that are not owned by LXD in a LXD volume group, because LXD might delete them.

By default, LVM storage pools use an LVM thin pool and create logical volumes for all LXD storage entities (images, instances and custom volumes) in there.
This behavior can be changed by setting {ref}`lvm.use_thinpool <storage-lvm-pool-config>` to `false` when you create the pool.
In this case, LXD uses "normal" logical volumes for all storage entities that are not snapshots.
Note that this entails serious performance and space reductions for the `lvm` driver (close to the `dir` driver both in speed and storage usage).
The reason for this is that most storage operations must fall back to rsyncing, because logical volumes that are not thin pools do not support snapshots of snapshots.
In addition, non-thin snapshots take up much more storage space than thin snapshots, because they must reserve space for their maximum size at creation time.
Therefore, this option should only be chosen if the use case requires it.

For environments with a high instance turnover (for example, continuous integration) you should tweak the backup `retain_min` and `retain_days` settings in `/etc/lvm/lvm.conf` to avoid slowdowns when interacting with LXD.

## Configuration options

The following configuration options are available for storage pools that use the `lvm` driver and for storage volumes in these pools.

(storage-lvm-pool-config)=
### Storage pool configuration
Key                           | Type                          | Default                                 | Description
:--                           | :---                          | :------                                 | :----------
lvm.thinpool\_name            | string                        | LXDThinPool                             | Thin pool where volumes are created
lvm.thinpool\_metadata\_size  | string                        | 0 (auto)                                | The size of the thin pool metadata volume (the default is to let LVM calculate an appropriate size)
lvm.use\_thinpool             | bool                          | true                                    | Whether the storage pool uses a thin pool for logical volumes
lvm.vg.force\_reuse           | bool                          | false                                   | Force using an existing non-empty volume group
lvm.vg\_name                  | string                        | name of the pool                        | Name of the volume group to create
rsync.bwlimit                 | string                        | 0 (no limit)                            | The upper limit to be placed on the socket I/O when rsync must be used to transfer storage entities
rsync.compression             | bool                          | true                                    | Whether to use compression while migrating storage pools
source                        | string                        | -                                       | Path to block device or loop file or file system entry

(storage-lvm-vol-config)=
### Storage volume configuration
Key                     | Type      | Condition                 | Default                               | Description
:--                     | :---      | :--------                 | :------                               | :----------
block.filesystem        | string    | block based driver        | same as volume.block.filesystem       | {{block_filesystem}}
block.mount\_options    | string    | block based driver        | same as volume.block.mount\_options   | Mount options for block devices
lvm.stripes             | string    | LVM driver                | -                                     | Number of stripes to use for new volumes (or thin pool volume)
lvm.stripes.size        | string    | LVM driver                | -                                     | Size of stripes to use (at least 4096 bytes and multiple of 512 bytes)
security.shifted        | bool      | custom volume             | false                                 | {{enable_ID_shifting}}
security.unmapped       | bool      | custom volume             | false                                 | Disable ID mapping for the volume
size                    | string    | appropriate driver        | same as volume.size                   | Size/quota of the storage volume
snapshots.expiry        | string    | custom volume             | -                                     | {{snapshot_expiry_format}}
snapshots.pattern       | string    | custom volume             | snap%d                                | {{snapshot_pattern_format}}
snapshots.schedule      | string    | custom volume             | -                                     | {{snapshot_schedule_format}}