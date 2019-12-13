# Introduction

## raidz vs draid

ZFS users are most likely very familiar with raidz already, so a comparison with draid would help. The illustrations below are simplified, but sufficient for the purpose of a comparison. For example, 31 drives can be configured as a zpool of 6 raidz1 vdevs and a hot spare:
![raidz1](https://cloud.githubusercontent.com/assets/6722662/23642396/9790e432-02b7-11e7-8198-ae9f17c61d85.png)

As shown above, if drive 0 fails and is replaced by the hot spare, only 5 out of the 30 surviving drives will work to resilver: drives 1-4 read, and drive 30 writes.

The same 30 drives can be configured as 1 draid1 vdev of the same level of redundancy (i.e. single parity, 1/4 parity ratio) and single spare capacity:
![draid1](https://cloud.githubusercontent.com/assets/6722662/23642395/9783ef8e-02b7-11e7-8d7e-31d1053ee4ff.png)

The drives are shuffled in a way that, after drive 0 fails, all 30 surviving drives will work together to restore the lost data/parity:
* All 30 drives read, because unlike the raidz1 configuration shown above, in the draid1 configuration the neighbor drives of the failed drive 0 (i.e. drives in a same data+parity group) are not fixed.
* All 30 drives write, because now there is no dedicated spare drive. Instead, spare blocks come from all drives.

To summarize:
* Normal application IO: draid and raidz are very similar. There's a slight advantage in draid, since there's no dedicated spare drive which is idle when not in use.
* Restore lost data/parity: for raidz, not all surviving drives will work to rebuild, and in addition it's bounded by the write throughput of a single replacement drive. For draid, the rebuild speed will scale with the total number of drives because all surviving drives will work to rebuild.

The dRAID vdev must shuffle its child drives in a way that regardless of which drive has failed, the rebuild IO (both read and write) will distribute evenly among all surviving drives, so the rebuild speed will scale. The exact mechanism used by the dRAID vdev driver is beyond the scope of this simple introduction here. If interested, please refer to the recommended readings in the next section.

## Recommended Reading

Parity declustering (the fancy term for shuffling drives) has been an active research topic, and many papers have been published in this area. The [Permutation Development Data Layout](http://www.cse.scu.edu/~tschwarz/TechReports/hpca.pdf) is a good paper to begin. The dRAID vdev driver uses a shuffling algorithm loosely based on the mechanism described in this paper.

# Use dRAID

First get the code [here](https://github.com/zfsonlinux/zfs/pull/9558), build zfs with _configure --enable-debug_, and install. Then load the zfs kernel module with the following options:
* zfs_vdev_scrub_max_active=10 zfs_vdev_async_write_min_active=4: These options help dRAID rebuild performance.
* draid_debug_lvl=5: This option controls the verbosity level of dRAID debug traces, which is very useful for troubleshooting.

Again, very important to _configure_ both spl and zfs with _--enable-debug_.

## Create a dRAID vdev

Unlike a raidz vdev, before a dRAID vdev can be created, a configuration file must be created with the _draidcfg_ command:
```
# draidcfg -p 1 -d 4 -s 2 -n 17 17.nvl
Not enough entropy at /dev/random: read -1, wanted 8.
Using /dev/urandom instead.
 Worst ( 3 x  5 +  2) x   544: 0.882
Seed chosen: f0cbfeccac3071b0
```

The command in the example above creates a configuration for a 17-drive dRAID1 vdev with 4 data blocks per strip and 2 distributed spares, and saves it to file _17.nvl_. Options:
* p: parity level, can be 1, 2, or 3.
* d: # data blocks per stripe.
* s: # distributed spare
* n: total # of drives
* It's required that: (n - s) % (p + d) == 0

Note that:
* Errors like "Not enough entropy at /dev/random" are harmless
* In the future, the _draidcfg_ may get integrated into _zpool create_ so there'd be no separate step for configuration generation.

The configuration file is binary, to examine the contents:
```
# draidcfg -r 17.nvl 
dRAID1 vdev of 17 child drives: 3 x (4 data + 1 parity) and 2 distributed spare
Using 32 base permutations
   1,12,13, 5,15,11, 2, 6, 4,16, 9, 7,14,10, 3, 0, 8,
   0, 1, 5,10, 8, 6,15, 4, 7,14, 2,13,12, 3,11,16, 9,
   1, 7,11,13,14,16, 4,12, 0,15, 9, 2,10, 3, 6, 5, 8,
   5,16, 3,15,10, 0,13,11,12, 8, 2, 9, 6, 4, 7, 1,14,
   9,15, 6, 8,12,11, 7, 1, 3, 0,13, 5,16,14, 4,10, 2,
  10, 1, 5,11, 3, 6,15, 2,12,13, 9, 4,16,14, 0, 7, 8,
  10,16,12, 7, 1, 3, 9,14, 5,15, 4,11, 2, 0,13, 8, 6,
   7,12, 4,13, 6,11, 9,15,14, 2,16, 3, 0, 1,10, 5, 8,
  10, 5, 8, 2, 1,11,16,15,12, 3,13, 4, 0, 7, 9, 6,14,
   1, 6,15, 0,14, 5, 9,11, 8,16,10, 2,13,12, 3, 4, 7,
  14, 4, 2, 0,12, 7, 3, 6, 8,13,10, 1,11,16,15, 9, 5,
   6,14, 8,10, 1, 0,15, 4, 5, 3,16,13, 9,12, 2, 7,11,
  13, 5, 8,14, 1,10,16,11,15, 7, 0,12, 2, 9, 4, 6, 3,
   9, 6, 3, 7,15, 1, 4, 8,14, 5, 0, 2,16,10,12,11,13,
  12, 0, 6, 7, 1, 9,14, 8,11,16, 4, 2,13,15, 3, 5,10,
  14, 6,12,10,15,13, 7, 0, 3,16, 5, 9, 2, 8, 4,11, 1,
  15,16, 8,13, 6, 4, 7,11, 1, 2,14,12,10, 5, 9, 3, 0,
   0,11,10,14,12, 1,16, 3,13, 9, 5, 7, 2, 4, 6,15, 8,
   2,10,12, 4, 3, 5,15, 1,11, 0, 7,13, 6, 9,14, 8,16,
  11, 8,16,12, 6,13,10, 9, 2, 7, 3, 4, 5, 0,14,15, 1,
   4,16,12,15,14, 3, 7, 1, 9,10, 6, 8,11, 0,13, 2, 5,
   5,16,13,11, 4, 6, 7,12, 0, 9,15, 1,14, 3, 8,10, 2,
  12, 6, 7, 0,10,15, 8, 2,16,14,11, 1, 4, 5, 9,13, 3,
   8, 4, 1,13, 6, 5, 0,15, 7, 3,11,14,16, 9,10,12, 2,
  16,14,15, 2,10,11, 6,13, 4, 9, 8, 0, 5,12, 3, 1, 7,
   9, 6, 8, 3,12,14,16,13,11,10, 4, 5, 7,15, 2, 0, 1,
   3, 9,15, 0, 7, 1, 8,11,12, 2,10, 6,13,16, 5,14, 4,
   0,14, 6,16, 1,10, 9,15,12, 8,11, 3, 2, 7,13, 5, 4,
  12,13, 9, 5,11, 6, 3, 4,14,10, 1, 7, 8, 2, 0,16,15,
  16, 9, 0, 2, 3,10, 1,11, 6, 4,13,12,14, 7, 5,15, 8,
  16, 9, 6, 0, 1, 4,11,14,12, 3, 2,15,13,10, 5, 8, 7,
   7, 8,11,14,10, 6,15,13, 1, 4,16, 9, 2, 3, 0,12, 5,
```

Now a dRAID vdev can be created using the configuration. The only difference from a normal _zpool create_ is the addition of a configuration file in the vdev specification:
```
# zpool create -f tank draid1 cfg=17.nvl sdd sde sdf sdg sdh sdi sdj sdk sdl sdm sdn sdo sdp sdq sdr sds sdt
```
Note that:
* The total number of drives must equal the _-n_ option of _draidcfg_.
* The parity level must match the _-p_ option, e.g. use draid3 for _draidcfg -p 3_

When the numbers don't match, _zpool create_ will fail but with a generic error message, which can be confusing.

Now the dRAID vdev is online and ready for IO:
```
# zpool status
  pool: tank
 state: ONLINE
config:

        NAME            STATE     READ WRITE CKSUM
        tank            ONLINE       0     0     0
          draid1-0      ONLINE       0     0     0
            sdd         ONLINE       0     0     0
            sde         ONLINE       0     0     0
            sdf         ONLINE       0     0     0
            sdg         ONLINE       0     0     0
            sdh         ONLINE       0     0     0
            sdu         ONLINE       0     0     0
            sdj         ONLINE       0     0     0
            sdv         ONLINE       0     0     0
            sdl         ONLINE       0     0     0
            sdm         ONLINE       0     0     0
            sdn         ONLINE       0     0     0
            sdo         ONLINE       0     0     0
            sdp         ONLINE       0     0     0
            sdq         ONLINE       0     0     0
            sdr         ONLINE       0     0     0
            sds         ONLINE       0     0     0
            sdt         ONLINE       0     0     0
        spares
          %draid1-0-s0  AVAIL   
          %draid1-0-s1  AVAIL
```

There are two logical spare vdevs shown above at the bottom:
* The names begin with a '%' followed by the name of the parent dRAID vdev.
* These spare are logical, made from reserved blocks on all the 17 child drives of the dRAID vdev.
* Unlike traditional hot spares, the distributed spare can only replace a drive in its parent dRAID vdev.

The dRAID vdev behaves just like a raidz vdev of the same parity level. You can do IO to/from it, scrub it, fail a child drive and it'd operate in degraded mode.

## Rebuild to distributed spare

When there's a bad/offlined/failed child drive, the dRAID vdev supports a completely new mechanism to reconstruct lost data/parity, in addition to the resilver. First of all, resilver is still supported - if a failed drive is replaced by another physical drive, the resilver process is used to reconstruct lost data/parity to the new replacement drive, which is the same as a resilver in a raidz vdev.

But if a child drive is replaced with a distributed spare, a new process called rebuild is used instead of resilver:
```
# zpool offline tank sdo
# zpool replace tank sdo '%draid1-0-s0'
# zpool status
  pool: tank
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: rebuilt 2.00G in 0h0m5s with 0 errors on Fri Feb 24 20:37:06 2017
config: 

        NAME                STATE     READ WRITE CKSUM
        tank                DEGRADED     0     0     0
          draid1-0          DEGRADED     0     0     0
            sdd             ONLINE       0     0     0
            sde             ONLINE       0     0     0
            sdf             ONLINE       0     0     0
            sdg             ONLINE       0     0     0
            sdh             ONLINE       0     0     0
            sdu             ONLINE       0     0     0
            sdj             ONLINE       0     0     0
            sdv             ONLINE       0     0     0
            sdl             ONLINE       0     0     0
            sdm             ONLINE       0     0     0
            sdn             ONLINE       0     0     0
            spare-11        DEGRADED     0     0     0
              sdo           OFFLINE      0     0     0
              %draid1-0-s0  ONLINE       0     0     0
            sdp             ONLINE       0     0     0
            sdq             ONLINE       0     0     0
            sdr             ONLINE       0     0     0
            sds             ONLINE       0     0     0
            sdt             ONLINE       0     0     0
        spares
          %draid1-0-s0      INUSE     currently in use
          %draid1-0-s1      AVAIL
```

The scan status line of the _zpool status_ output now says _"rebuilt"_ instead of _"resilvered"_, because the lost data/parity was rebuilt to the distributed spare by a brand new process called _"rebuild"_. The main differences from _resilver_ are:
* The rebuild process does not scan the whole block pointer tree. Instead, it only scans the spacemap objects.
* The IO from rebuild is sequential, because it rebuilds metaslabs one by one in sequential order.
* The rebuild process is not limited to block boundaries. For example, if 10 64K blocks are allocated contiguously, then rebuild will fix 640K at one time. So rebuild process will generate larger IOs than resilver.
* For all the benefits above, there is one price to pay. The rebuild process cannot verify block checksums, since it doesn't have block pointers.
* Moreover, the rebuild process requires support from on-disk format, and **only** works on draid and mirror vdevs. Resilver, on the other hand, works with any vdev (including draid).

Although rebuild process creates larger IOs, the drives will not necessarily see large IO requests. The block device queue parameter _/sys/block/*/queue/max_sectors_kb_ must be tuned accordingly. However, since the rebuild IO is already sequential, the benefits of enabling larger IO requests might be marginal.

At this point, redundancy has been fully restored without adding any new drive to the pool. If another drive is offlined, the pool is still able to do IO:
```
# zpool offline tank sdj
# zpool status 
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: rebuilt 2.00G in 0h0m5s with 0 errors on Fri Feb 24 20:37:06 2017
config:

        NAME                STATE     READ WRITE CKSUM
        tank                DEGRADED     0     0     0
          draid1-0          DEGRADED     0     0     0
            sdd             ONLINE       0     0     0
            sde             ONLINE       0     0     0
            sdf             ONLINE       0     0     0
            sdg             ONLINE       0     0     0
            sdh             ONLINE       0     0     0
            sdu             ONLINE       0     0     0
            sdj             OFFLINE      0     0     0
            sdv             ONLINE       0     0     0
            sdl             ONLINE       0     0     0
            sdm             ONLINE       0     0     0
            sdn             ONLINE       0     0     0
            spare-11        DEGRADED     0     0     0
              sdo           OFFLINE      0     0     0
              %draid1-0-s0  ONLINE       0     0     0
            sdp             ONLINE       0     0     0
            sdq             ONLINE       0     0     0
            sdr             ONLINE       0     0     0
            sds             ONLINE       0     0     0
            sdt             ONLINE       0     0     0
        spares
          %draid1-0-s0      INUSE     currently in use
          %draid1-0-s1      AVAIL
```

As shown above, the _draid1-0_ vdev is still in _DEGRADED_ mode although two child drives have failed and it's only single-parity. Since the _%draid1-0-s1_ is still _AVAIL_, full redundancy can be restored by replacing _sdj_ with it, without adding new drive to the pool:
```
# zpool replace tank sdj '%draid1-0-s1'
# zpool status
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: rebuilt 2.13G in 0h0m5s with 0 errors on Fri Feb 24 23:20:59 2017
config:

        NAME                STATE     READ WRITE CKSUM
        tank                DEGRADED     0     0     0
          draid1-0          DEGRADED     0     0     0
            sdd             ONLINE       0     0     0
            sde             ONLINE       0     0     0
            sdf             ONLINE       0     0     0
            sdg             ONLINE       0     0     0
            sdh             ONLINE       0     0     0
            sdu             ONLINE       0     0     0
            spare-6         DEGRADED     0     0     0
              sdj           OFFLINE      0     0     0
              %draid1-0-s1  ONLINE       0     0     0
            sdv             ONLINE       0     0     0
            sdl             ONLINE       0     0     0
            sdm             ONLINE       0     0     0
            sdn             ONLINE       0     0     0
            spare-11        DEGRADED     0     0     0
              sdo           OFFLINE      0     0     0
              %draid1-0-s0  ONLINE       0     0     0
            sdp             ONLINE       0     0     0
            sdq             ONLINE       0     0     0
            sdr             ONLINE       0     0     0
            sds             ONLINE       0     0     0
            sdt             ONLINE       0     0     0
        spares
          %draid1-0-s0      INUSE     currently in use
          %draid1-0-s1      INUSE     currently in use
```

Again, full redundancy has been restored without adding any new drive. If another drive fails, the pool will still be able to handle IO, but there'd be no more distributed spare to rebuild (both are in _INUSE_ state now). At this point, there's no urgency to add a new replacement drive because the pool can survive yet another drive failure.

### Rebuild for mirror vdev

The sequential rebuild process also works for the mirror vdev, when a drive is attached to a mirror or a mirror child vdev is replaced.

By default, rebuild for mirror vdev is turned off. It can be turned on using the zfs module option _spa_rebuild_mirror=1_.

### Rebuild throttling

The rebuild process may delay _zio_ by _spa_vdev_scan_delay_ if the draid vdev has seen any important IO in the recent _spa_vdev_scan_idle_ period. But when a dRAID vdev has lost all redundancy, e.g. a draid2 with 2 faulted child drives, the rebuild process will go full speed by ignoring _spa_vdev_scan_delay_ and _spa_vdev_scan_idle_ altogether because the vdev is now in critical state.

After delaying, the rebuild zio is issued using priority _ZIO_PRIORITY_SCRUB_ for reads and _ZIO_PRIORITY_ASYNC_WRITE_ for writes. Therefore the options that control the queuing of these two IO priorities will affect rebuild _zio_ as well, for example _zfs_vdev_scrub_min_active_, _zfs_vdev_scrub_max_active_, _zfs_vdev_async_write_min_active_, and _zfs_vdev_async_write_max_active_.

## Rebalance

Distributed spare space can be made available again by simply replacing any failed drive with a new drive. This process is called _rebalance_ which is essentially a _resilver_:
```
# zpool replace -f tank sdo sdw
# zpool status
 state: DEGRADED
status: One or more devices has been taken offline by the administrator.
        Sufficient replicas exist for the pool to continue functioning in a
        degraded state.
action: Online the device using 'zpool online' or replace the device with
        'zpool replace'.
  scan: resilvered 2.21G in 0h0m58s with 0 errors on Fri Feb 24 23:31:45 2017
config:

        NAME                STATE     READ WRITE CKSUM
        tank                DEGRADED     0     0     0
          draid1-0          DEGRADED     0     0     0
            sdd             ONLINE       0     0     0
            sde             ONLINE       0     0     0
            sdf             ONLINE       0     0     0
            sdg             ONLINE       0     0     0
            sdh             ONLINE       0     0     0
            sdu             ONLINE       0     0     0
            spare-6         DEGRADED     0     0     0
              sdj           OFFLINE      0     0     0
              %draid1-0-s1  ONLINE       0     0     0
            sdv             ONLINE       0     0     0
            sdl             ONLINE       0     0     0
            sdm             ONLINE       0     0     0
            sdn             ONLINE       0     0     0
            sdw             ONLINE       0     0     0
            sdp             ONLINE       0     0     0
            sdq             ONLINE       0     0     0
            sdr             ONLINE       0     0     0
            sds             ONLINE       0     0     0
            sdt             ONLINE       0     0     0
        spares
          %draid1-0-s0      AVAIL   
          %draid1-0-s1      INUSE     currently in use
```

Note that the scan status now says _"resilvered"_. Also, the state of _%draid1-0-s0_ has become _AVAIL_ again. Since the resilver process checks block checksums, it makes up for the lack of checksum verification during previous rebuild.

The dRAID1 vdev in this example shuffles three (4 data + 1 parity) redundancy groups to the 17 drives. For any single drive failure, only about 1/3 of the blocks are affected (and should be resilvered/rebuilt). The rebuild process is able to avoid unnecessary work, but the resilver process by default will not. The rebalance (which is essentially resilver) can speed up a lot by setting module option _zfs_no_resilver_skip_ to 0. This feature is turned off by default because of issue https://github.com/zfsonlinux/zfs/issues/5806. 

# Troubleshooting

Please report bugs to [the dRAID PR](https://github.com/zfsonlinux/zfs/pull/9558), as long as the code is not merged upstream. The following information would be useful:
* dRAID configuration, i.e. the *.nvl file created by _draidcfg_ command.
* Output of _zpool events -v_
* dRAID debug traces, which by default goes to _dmesg_ via _printk()_. The dRAID debugging traces can also use _trace_printk()_, which is more preferable but unfortunately GPL only.