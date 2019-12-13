### GitHub Repositories

The official source for ZFS on Linux is maintained at GitHub by the [zfsonlinux][zol-org] organization.  The project consists of two primary git repositories named [spl][spl-repo] and [zfs][zfs-repo], both are required to build ZFS on Linux.  

**NOTE:** The SPL was merged in to the [zfs][zfs-repo] repository, the last major release with a separate SPL is `0.7`.

* **SPL**: The SPL is thin shim layer which is responsible for implementing the fundamental interfaces required by OpenZFS.  It's this layer which allows OpenZFS to be used across multiple platforms.

* **ZFS**: The ZFS repository contains a copy of the upstream OpenZFS code which has been adapted and extended for Linux.  The vast majority of the core OpenZFS code is self-contained and can be used without modification.

### Installing Dependencies

The first thing you'll need to do is prepare your environment by installing a full development tool chain.  In addition, development headers for both the kernel and the following libraries must be available.  Finally, if you wish to run the ZFS Test Suite `ksh` must be installed.

It is important to note that if the development kernel headers for the currently running kernel aren't installed, the modules won't compile properly.

For RHEL and CentOS:

```sh
$ sudo yum install autoconf automake libtool make rpm-build ksh
$ sudo yum install zlib-devel libuuid-devel libattr-devel libblkid-devel libselinux-devel libudev-devel
$ sudo yum install libacl-devel libaio-devel device-mapper-devel openssl-devel libtirpc-devel elfutils-libelf-devel
$ sudo yum install kernel-devel-$(uname -r)
$ sudo yum install epel-release
$ sudo yum install python36 python36-devel python36-setuptools python36-cffi
```

For Fedora:

```sh
$ sudo dnf install autoconf automake libtool make rpm-build ksh
$ sudo dnf install zlib-devel libuuid-devel libattr-devel libblkid-devel libselinux-devel libudev-devel
$ sudo dnf install libacl-devel libaio-devel device-mapper-devel openssl-devel libtirpc-devel elfutils-libelf-devel
$ sudo dnf install kernel-devel-$(uname -r)
$ sudo dnf install python3 python3-devel python3-setuptools python3-cffi 
```

For Debian and Ubuntu:

```sh
$ sudo apt-get install build-essential autoconf automake libtool gawk alien fakeroot ksh
$ sudo apt-get install zlib1g-dev uuid-dev libattr1-dev libblkid-dev libselinux-dev libudev-dev
$ sudo apt-get install libacl1-dev libaio-dev libdevmapper-dev libssl-dev libelf-dev
$ sudo apt-get install linux-headers-$(uname -r)
$ sudo apt-get install python3 python3-dev python3-setuptools python3-cffi
```

### Build Options

There are two options for building ZFS on Linux, the correct one largely depends on your requirements.

* **Packages**: Often it can be useful to build custom packages from git which can be installed on a system.  This is the best way to perform integration testing with systemd, dracut, and udev.  The downside to using packages it is greatly increases the time required to build, install, and test a change.

* **In-tree**: Development can be done entirely in the SPL and ZFS source trees.  This speeds up development by allowing developers to rapidly iterate on a patch.  When working in-tree developers can leverage incremental builds, load/unload kernel modules, execute utilities, and verify all their changes with the ZFS Test Suite.

The remainder of this page focuses on the **in-tree** option which is the recommended method of development for the majority of changes.  See the [[custom-packages]] page for additional information on building custom packages.

### Developing In-Tree

#### Clone from GitHub

Start by cloning the SPL and ZFS repositories from GitHub.  The repositories have a **master** branch for development and a series of **\*-release** branches for tagged releases.  After checking out the repository your clone will default to the master branch.  Tagged releases may be built by checking out spl/zfs-x.y.z tags with matching version numbers or matching release branches.  Avoid using mismatched versions, this can result build failures due to interface changes.

**NOTE:** SPL was merged in to the [zfs][zfs-repo] repository, last release with separate SPL is `0.7`.
```
git clone https://github.com/zfsonlinux/zfs
```

If you need 0.7 release or older:
```
git clone https://github.com/zfsonlinux/spl
```

#### Configure and Build

For developers working on a change always create a new topic branch based off of master.  This will make it easy to open a pull request with your change latter.  The master branch is kept stable with extensive [regression testing][buildbot] of every pull request before and after it's merged.  Every effort is made to catch defects as early as possible and to keep them out of the tree.  Developers should be comfortable frequently rebasing their work against the latest master branch.

If you want to build 0.7 release or older, you should compile SPL first:

```
cd ./spl
git checkout master
sh autogen.sh
./configure
make -s -j$(nproc)
```

In this example we'll use the master branch and walk through a stock **in-tree** build, so we don't need to build SPL separately. Start by checking out the desired branch then build the ZFS and SPL source in the tradition autotools fashion.

```
cd ./zfs
git checkout master
sh autogen.sh
./configure
make -s -j$(nproc)
```

**tip:** `--with-linux=PATH` and `--with-linux-obj=PATH` can be passed to configure to specify a kernel installed in a non-default location.  This option is also supported when building ZFS.  
**tip:** `--enable-debug` can be set to enable all ASSERTs and additional correctness tests.  This option is also supported when building ZFS.  
**tip:** for version `<=0.7` `--with-spl=PATH` and `--with-spl-obj=PATH`, where `PATH` is a full path, can be passed to configure if it is unable to locate the SPL. 

**Optional**  Build packages

```
make deb #example for Debian/Ubuntu 
```

#### Running zloop.sh and zfs-tests.sh

There are a few helper scripts provided in the top-level scripts directory designed to aid developers working with in-tree builds.

* **zfs-helper.sh:** Certain functionality (i.e. /dev/zvol/) depends on the ZFS provided udev helper scripts being installed on the system.  This script can be used to create symlinks on the system from the installation location to the in-tree helper.  These links must be in place to successfully run the ZFS Test Suite.  The **-i** and **-r** options can be used to install and remove the symlinks.

```
sudo ./scripts/zfs-helpers.sh -i
```

* **zfs.sh:** The freshly built kernel modules can be loaded using `zfs.sh`.  This script can latter be used to unload the kernel modules with the **-u** option.

```
sudo ./scripts/zfs.sh
```

* **zloop.sh:** A wrapper to run ztest repeatedly with randomized arguments.  The ztest command is a user space stress test designed to detect correctness issues by concurrently running a random set of test cases.  If a crash is encountered, the ztest logs, any associated vdev files, and core file (if one exists) are collected and moved to the output directory for analysis.

```
sudo ./scripts/zloop.sh
```

* **zfs-tests.sh:** A wrapper which can be used to launch the ZFS Test Suite.  Three loopback devices are created on top of sparse files located in `/var/tmp/` and used for the regression test.  Detailed directions for the ZFS Test Suite can be found in the [README][zts-readme] located in the top-level tests directory.

```
 ./scripts/zfs-tests.sh -vx
```

**tip:** The **delegate** tests will be skipped unless group read permission is set on the zfs directory and its parents.

[zol-org]: https://github.com/zfsonlinux/
[spl-repo]: https://github.com/zfsonlinux/spl
[zfs-repo]: https://github.com/zfsonlinux/zfs
[buildbot]: http://build.zfsonlinux.org/
[zts-readme]: https://github.com/zfsonlinux/zfs/tree/master/tests