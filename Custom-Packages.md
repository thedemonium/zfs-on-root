The following instructions assume you are building from an official [release tarball][release] (version 0.8.0 or newer) or directly from the [git repository][git]. Most users should not need to do this and should preferentially use the distribution packages. As a general rule the distribution packages will be more tightly integrated, widely tested, and better supported. However, if your distribution of choice doesn't provide packages, or you're a developer and want to roll your own, here's how to do it.

The first thing to be aware of is that the build system is capable of generating several different types of packages. Which type of package you choose depends on what's supported on your platform and exactly what your needs are.

* **DKMS** packages contain only the source code and scripts for rebuilding the kernel modules. When the DKMS package is installed kernel modules will be built for all available kernels. Additionally, when the kernel is upgraded new kernel modules will be automatically built for that kernel. This is particularly convenient for desktop systems which receive frequent kernel updates. The downside is that because the DKMS packages build the kernel modules from source a full development environment is required which may not be appropriate for large deployments.

* **kmods** packages are binary kernel modules which are compiled against a specific version of the kernel. This means that if you update the kernel you must compile and install a new kmod package. If you don't frequently update your kernel, or if you're managing a large number of systems, then kmod packages are a good choice.

* **kABI-tracking kmod** Packages are similar to standard binary kmods and may be used with Enterprise Linux distributions like Red Hat and CentOS.  These distributions provide a stable kABI (Kernel Application Binary Interface) which allows the same binary modules to be used with new versions of the distribution provided kernel.

By default the build system will generate user packages and both DKMS and kmod style kernel packages if possible. The user packages can be used with either set of kernel packages and do not need to be rebuilt when the kernel is updated. You can also streamline the build process by building only the DKMS or kmod packages as shown below.

Be aware that when building directly from a git repository you must first run the *autogen.sh* script to create the *configure* script. This will require installing the GNU autotools packages for your distribution.  To perform any of the builds, you must install all the necessary development tools and headers for your distribution.

It is important to note that if the development kernel headers for the currently running kernel aren't installed, the modules won't compile properly.

* [Red Hat, CentOS and Fedora](#red-hat-centos-and-fedora)
* [Debian and Ubuntu](#debian-and-ubuntu)

## Red Hat, CentOS and Fedora

Make sure that the required packages are installed:

```sh
$ sudo dnf install autoconf automake libtool rpm-build
$ sudo dnf install zlib-devel libuuid-devel libattr-devel libblkid-devel libselinux-devel libudev-devel
$ sudo dnf install libacl-devel libaio-devel device-mapper-devel openssl-devel libtirpc-devel elfutils-libelf-devel
$ sudo dnf install kernel-devel-$(uname -r)

# To enable the pyzfs packages additionally install the following:

# Fedora
$ sudo dnf install python3 python3-devel python3-setuptools python3-cffi 

# For Red Hat / CentOS 7
$ sudo yum install epel-release
$ sudo yum install python36 python36-devel python36-setuptools python36-cffi

# Additional for CentOS 8 
# Enable PowerTools repository - it is required for device-mapper-devel
$ sudo dnf config-manager --enable PowerTools
# For building kmod packages
$ sudo dnf install kernel-rpm-macros libffi-devel
```

[Get the source code](#get-the-source-code).

### DKMS

Building rpm-based DKMS and user packages can be done as follows:

```sh
$ cd zfs
$ ./configure --with-config=srpm
$ make -j1 pkg-utils rpm-dkms
$ sudo yum localinstall *.$(uname -p).rpm *.noarch.rpm
```

### kmod

The key thing to know when building a kmod package is that a specific Linux kernel must be specified. At configure time the build system will make an educated guess as to which kernel you want to build against. However, if configure is unable to locate your kernel development headers, or you want to build against a different kernel, you must specify the exact path with the *--with-linux* and *--with-linux-obj* options.

```sh
$ cd zfs
$ ./configure
$ make -j1 pkg-utils pkg-kmod
$ sudo yum localinstall *.$(uname -p).rpm
```

### kABI-tracking kmod

The process for building kABI-tracking kmods is almost identical to for building normal kmods.  However, it will only produce binaries which can be used by multiple kernels if the distribution supports a stable kABI.  In order to request kABI-tracking package the *--with-spec=redhat* option must be passed to configure.

**NOTE:** This type of package is not available for Fedora.

```sh
$ cd zfs
$ ./configure --with-spec=redhat
$ make -j1 pkg-utils pkg-kmod
$ sudo yum localinstall *.$(uname -p).rpm
```

## Debian and Ubuntu

Make sure that the required packages are installed:

```sh
$ sudo apt-get install build-essential autoconf automake libtool gawk alien fakeroot gdebi-core
$ sudo apt-get install zlib1g-dev uuid-dev libattr1-dev libblkid-dev libselinux-dev libudev-dev
$ sudo apt-get install libacl1-dev libaio-dev libdevmapper-dev libssl-dev libelf-dev
$ sudo apt-get install linux-headers-$(uname -r)

# To enable the pyzfs packages additionally install the following:
$ sudo apt-get install python3 python3-dev python3-setuptools python3-cffi
```

[Get the source code](#get-the-source-code).

### kmod

The key thing to know when building a kmod package is that a specific Linux kernel must be specified. At configure time the build system will make an educated guess as to which kernel you want to build against. However, if configure is unable to locate your kernel development headers, or you want to build against a different kernel, you must specify the exact path with the *--with-linux* and *--with-linux-obj* options.

```sh
$ cd zfs
$ ./configure
$ make -j1 pkg-utils deb-kmod
$ for file in *.deb; do sudo gdebi -q --non-interactive $file; done
```

### DKMS

Building deb-based DKMS and user packages can be done as follows:

```sh
$ sudo apt-get install dkms
$ cd zfs
$ ./configure --with-config=srpm
$ make -j1 pkg-utils deb-dkms
$ for file in *.deb; do sudo gdebi -q --non-interactive $file; done
```

## Get the Source Code

### Released Tarball

The released tarball contains the latest fully tested and released version of ZFS.  This is the preferred source code location for use in production systems.  If you want to use the official released tarballs, then use the following commands to fetch and prepare the source.

```sh
$ wget http://archive.zfsonlinux.org/downloads/zfsonlinux/zfs/zfs-x.y.z.tar.gz
$ tar -xzf zfs-x.y.z.tar.gz
```

### Git Master Branch

The Git *master* branch contains the latest version of the software, and will probably contain fixes that, for some reason, weren't included in the released tarball.  This is the preferred source code location for developers who intend to modify ZFS.  If you would like to use the git version, you can clone it from Github and prepare the source like this.

```sh
$ git clone https://github.com/zfsonlinux/zfs.git
$ cd zfs
$ ./autogen.sh
```

Once the source has been prepared you'll need to decide what kind of packages you're building and jump the to appropriate section above.  Note that not all package types are supported for all platforms.

[release]: https://github.com/zfsonlinux/zfs/releases/latest
[git]: https://github.com/zfsonlinux/zfs
