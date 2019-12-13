# Supported boot parameters
* rollback=\<on|yes|1\> Do a rollback of specified snapshot.
* zfs_debug=\<on|yes|1\> Debug the initrd script
* zfs_force=\<on|yes|1\> Force importing the pool. Should not be necessary.
* zfs=\<off|no|0\> Don't try to import ANY pool, mount ANY filesystem or even load the module.
* rpool=\<pool\> Use this pool for root pool.
* bootfs=\<pool\>/\<dataset\> Use this dataset for root filesystem.
* root=\<pool\>/\<dataset\> Use this dataset for root filesystem.
* root=ZFS=\<pool\>/\<dataset\> Use this dataset for root filesystem.
* root=zfs:\<pool\>/\<dataset\> Use this dataset for root filesystem.
* root=zfs:AUTO Try to detect both pool and rootfs

In all these cases, \<dataset\> could also be \<dataset\>@\<snapshot\>.

The reason there are so many supported boot options to get the root filesystem, is that there are a lot of different ways too boot ZFS out there, and I wanted to make sure I supported them all.

# Pool imports
## Import using /dev/disk/by-*
The initrd will, if the variable <code>USE_DISK_BY_ID</code> is set in the file <code>/etc/default/zfs</code>, to import using the /dev/disk/by-* links. It will try to import in this order:

1. /dev/disk/by-vdev
2. /dev/disk/by-\*
3. /dev

## Import using cache file
If all of these imports fail (or if <code>USE_DISK_BY_ID</code> is unset), it will then try to import using the cache file.

## Last ditch attempt at importing
If that ALSO fails, it will try one more time, without any <code>-d</code> or <code>-c</code> options.

# Booting
## Booting from snapshot:
Enter the snapshot for the <code>root=</code> parameter like in this example:

```
linux   /ROOT/debian-1@/boot/vmlinuz-3.2.0-4-amd64 root=ZFS=rpool/ROOT/debian-1@some_snapshot ro boot=zfs $bootfs quiet
```

This will clone the snapshot <code>rpool/ROOT/debian-1@some_snapshot</code> into the filesystem <code>rpool/ROOT/debian-1_some_snapshot</code> and use that as root filesystem. The original filesystem and snapshot is left alone in this case.

**BEWARE** that it will first destroy, blindingly, the <code>rpool/ROOT/debian-1_some_snapshot</code> filesystem before trying to clone the snapshot into it again. So if you've booted from the same snapshot previously and done some changes in that root filesystem, they will be undone by the destruction of the filesystem.

## Snapshot rollback
From version <code>0.6.4-1-3</code> it is now also possible to specify <code>rollback=1</code> to do a rollback of the snapshot instead of cloning it. **BEWARE** that this will destroy _all_ snapshots done after the specified snapshot!

## Select snapshot dynamically
From version <code>0.6.4-1-3</code> it is now also possible to specify a NULL snapshot name (such as <code>root=rpool/ROOT/debian-1@</code>) and if so, the initrd script will discover all snapshots below that filesystem (sans the at), and output a list of snapshot for the user to choose from.

## Booting from native encrypted filesystem
Although there is currently no support for native encryption in ZFS On Linux, there is a patch floating around 'out there' and the initrd supports loading key and unlock such encrypted filesystem.

## Separated filesystems
### Descended filesystems
If there are separate filesystems (for example a separate dataset for <code>/usr</code>), the snapshot boot code will try to find the snapshot under each filesystems and clone (or rollback) them.

Example:

```
rpool/ROOT/debian-1@some_snapshot
rpool/ROOT/debian-1/usr@some_snapshot
```

These will create the following filesystems respectively (if not doing a rollback):

```
rpool/ROOT/debian-1_some_snapshot
rpool/ROOT/debian-1/usr_some_snapshot
```

The initrd code will use the <code>mountpoint</code> option (if any) in the original (without the snapshot part) dataset to find _where_ it should mount the dataset. Or it will use the name of the dataset below the root filesystem (<code>rpool/ROOT/debian-1</code> in this example) for the mount point.