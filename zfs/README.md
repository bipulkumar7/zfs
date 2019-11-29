![img](http://zfsonlinux.org/images/zfs-linux.png)

ZFS on Linux is an advanced file system and volume manager which was originally
developed for Solaris and is now maintained by the OpenZFS community.

[![codecov](https://codecov.io/gh/zfsonlinux/zfs/branch/master/graph/badge.svg)](https://codecov.io/gh/zfsonlinux/zfs)
[![coverity](https://scan.coverity.com/projects/1973/badge.svg)](https://scan.coverity.com/projects/zfsonlinux-zfs)

# Official Resources

  * [Site](http://zfsonlinux.org)
  * [Wiki](https://github.com/zfsonlinux/zfs/wiki)
  * [Mailing lists](https://github.com/zfsonlinux/zfs/wiki/Mailing-Lists)
  * [OpenZFS site](http://open-zfs.org/)

# Installation

Full documentation for installing ZoL on your favorite Linux distribution can
be found at [our site](http://zfsonlinux.org/).

# Contribute & Develop

We have a separate document with [contribution guidelines](./.github/CONTRIBUTING.md).

# Release

ZFS on Linux is released under a CDDL license.  
For more details see the NOTICE, LICENSE and COPYRIGHT files; `UCRL-CODE-235197`

# Supported Kernels
  * The `META` file contains the officially recognized supported kernel versions.
# Getting started & prerequisite

  *     VirtualBox 6.0
  *     ZFS Installed

# Disk setup on Virtualbox 6.0

Add 4(four) additional  disk sdd, sde, sdf, sdg inside virtual machine.
        i.e /dev/sdd, /dev/sde, /dev/sdf, /dev/sdg

# Maths on disks
        /dev/sdd =  8 GB
        /dev/sde =  8 GB
    +   /dev/sdf =  8 GB
        /dev/sdg =  8 GB

         TOTAL   = 32 GB

>  blkid  | grep -o .....sd['d|e|f|g'].
>  lsblk | grep   sd['d|e|f|g']


# Common  notes

 * VDEV = Virtual device

A VDEV is a meta-device that can represent one or more devices. ZFS supports 7 different types of VDEV.


# Creating ZFS pool

There are two types of simple storage pools we can create.
1. A striped pool, where a copy of data is stored across all drives
2. mirrored pool, where a single complete copy of data is stored on all drives.


* Creating a simple pool or striped pool with name *tank*

>  zpool create tank /dev/sdd /dev/sde /dev/sdf  /dev/sdg

In this case, I'm using four disk VDEVs. Notice that I'm  using full device paths.
VDEVs are always dynamically striped, this is effectively a RAID-0 between four drives (no redundancy).
We should also check the status of the zpool:

> zpool status tank
  pool: tank
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          sdd       ONLINE       0     0     0
          sde       ONLINE       0     0     0
          sdf       ONLINE       0     0     0


Once it get executed above command. It creates a partition table with Solaris file type.
Apart /tank directory will created *that holds the data upto 30 GB not verified* .



# Destroying zfs pool *tank* 

>  zpool destroy tank

# A simple mirrored zpool

In this next example, I wish to mirror all four drives (/dev/sdd, /dev/sde, /dev/sdf and /dev/sdg). So, rather than using the disk VDEV, I'll be using "mirror".

> zpool create tank mirror sdd sde sdf sdg
> zpool status tank
  pool: tank
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0


Size is 7.3 G of /tank
Notice that "mirror-0" is now the VDEV, with each physical device managed by it. As mentioned earlier, this would be analogous to a Linux software RAID "/dev/md0" device representing the four physical devices. Let's now clean up our pool, and create another.

> zpool destroy tank

# Nested VDEVs

VDEVs can be nested. A perfect example is a standard RAID-1+0 (commonly referred to as "RAID-10"). This is a stripe of mirrors. In order to specify the nested VDEVs, I just put them on the command line in order (emphasis mine):


> zpool create tank mirror  sdd sde  mirror sdf sdg
> zpool status tank
  pool: tank
 state: ONLINE
  scan: none requested
config:

        NAME        STATE     READ WRITE CKSUM
        tank        ONLINE       0     0     0
          mirror-0  ONLINE       0     0     0
            sdd     ONLINE       0     0     0
            sde     ONLINE       0     0     0
          mirror-1  ONLINE       0     0     0
            sdf     ONLINE       0     0     0
            sdg     ONLINE       0     0     0

errors: No known data errors

The first VDEV is "mirror-0" which is managing /dev/sdd and /dev/sde. This was done by calling "mirror sdd sde". The second VDEV is "mirror-1" which is managing /dev/sdf and /dev/sdg. This was done by calling "mirror sdf sdg". Because VDEVs are always dynamically striped, "mirror-0" and "mirror-1" are striped, thus creating the RAID-1+0 setup

> zpool destroy tank

# File VDEVs

As mentioned, pre-allocated files can be used fer setting up zpools on your existing ext4 filesystem (or whatever). It should be noted that this is meant entirely for testing purposes, and not for storing production data. Using files is a great way to have a sandbox, where you can test compression ratio, the size of the deduplication table, or other things without actually committing production data to it. When creating file VDEVs, you cannot use relative paths, but must use absolute paths. Further, the image files must be preallocated, and not sparse files or thin provisioned. Let's see how this works:


> for i in {1..4}; do dd if=/dev/zero of=/tmp/file$i bs=1G count=2 &> /dev/null; done
> zpool create tank /tmp/file1 /tmp/file2 /tmp/file3 /tmp/file4
> zpool status tank
  pool: tank
 state: ONLINE
  scan: none requested
config:

        NAME          STATE     READ WRITE CKSUM
        tank          ONLINE       0     0     0
          /tmp/file1  ONLINE       0     0     0
          /tmp/file2  ONLINE       0     0     0
          /tmp/file3  ONLINE       0     0     0
          /tmp/file4  ONLINE       0     0     0




In this case, we created a RAID-0. We used preallocated files using /dev/zero that are each 2GB in size. Thus, the size of our zpool is 8 GB in usable space. Each file, as with our first example using disks, is a VDEV. Of course, you can treat the files as disks, and put them into a mirror configuration, RAID-1+0, RAIDZ-1 (coming in the next post), etc.

Total size is 7.3 GB

> zpool destroy tank

# Hybrid pools

let's create a hybrid pool with cache and log drives. Again, I emphasized the nested VDEVs for clarity:

>  zpool create tank mirror /tmp/file1 /tmp/file2 mirror /tmp/file3  /tmp/file4 log mirror sdd sde cache sdf sdg
>  zpool status tank
  pool: tank
 state: ONLINE
  scan: none requested
config:

        NAME            STATE     READ WRITE CKSUM
        tank            ONLINE       0     0     0
          mirror-0      ONLINE       0     0     0
            /tmp/file1  ONLINE       0     0     0
            /tmp/file2  ONLINE       0     0     0
          mirror-1      ONLINE       0     0     0
            /tmp/file3  ONLINE       0     0     0
            /tmp/file4  ONLINE       0     0     0
        logs
          mirror-2      ONLINE       0     0     0
            sdd         ONLINE       0     0     0
            sde         ONLINE       0     0     0
        cache
          sdf           ONLINE       0     0     0
          sdg           ONLINE       0     0     0

errors: No known data errors

There's a lot going on here, so let's disect it. First, we created a RAID-1+0 using our four preallocated image files. Notice the VDEVs "mirror-0" and "mirror-1", and what they are managing. Second, we created a third VDEV called "mirror-2" that actually is not used for storing data in the pool, but is used as a ZFS intent log, or ZIL. We'll cover the ZIL in more detail in another post. Then we created two VDEVs for caching data called "sdg" and "sdh". The are standard disk VDEVs that we've already learned about. However, they are also managed by the "cache" VDEV. So, in this case, we've used 6 of the 7 VDEVs listed above, the only one missing is "spare".

Noticing the indentation will help you see what VDEV is managing what. The "tank" pool is comprised of the "mirror-0" and "mirror-1" VDEVs for long-term persistent storage. The ZIL is magaged by "mirror-2", which is comprised of /dev/sde and /dev/sdf. The read-only cache VDEV is managed by two disks, /dev/sdg and /dev/sdh. Neither the "logs" nor the "cache" are long-term storage for the pool, thus creating a "hybrid pool" setup.




= Real life example





Also do notice the simplicity in the implementation. Consider doing something similar with LVM, RAID and ext4. You would need to do the following:

> mdadm -C /dev/md0 -l 0 -n 4 /dev/sde /dev/sdf /dev/sdg /dev/sdh
> pvcreate /dev/md0
> vgcreate /dev/md0 tank
> lvcreate -l 100%FREE -n videos tank
> mkfs.ext4 /dev/tank/videos
> mkdir -p /tank/videos
> mount -t ext4 /dev/tank/videos /tank/videos
