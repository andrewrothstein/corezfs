# corezfs

## ZFS on CoreOS
This is a script to compile and install ZFS on CoreOS. It is meant to be installed on a fresh clean CoreOS instance. Although it can be run manually, it is envisioned that it is typically used as part of an automated provisioning process.

## For the Impatient
Copy the script to the target machine (which should be a fresh CoreOS install), compile the code, and then install it.

To build ZFS and use directly on a single machine:
```bash
mkdir corezfs
curl -L https://api.github.com/repos/varasys/corezfs/tarball | tar -zxv -C corezfs --strip-components=1
sudo ./corezfs build --no-archive
sudo ./corezfs install
sudo rm -rf corezfs #optional clean-up
```

To build ZFS and create an archive file that can be used on other machines:
```bash
#on the build machine
mkdir corezfs
curl -L https://api.github.com/repos/varasys/corezfs/tarball | tar -zxv -C corezfs --strip-components=1
sudo ./corezfs build
```

The commands above will create an archive file in the coreos directory which should
be copied into the corezfs directory on the target install machine. Note that the commands above did not install ZFS on the build machine.

To install the build from the example above on another machine, copy the archive file to the other machine and install it.

```bash
#on the install machine
#copy the archive file into the coreos folder
sudo ./corezfs install
```

If the installation is successful, the command "sudo zfs list" will return "no datasets available". If the installation is not successful, that command will probably complain about kernel drivers not being loaded.

## Introduction
[CoreOS](https://coreos.com/os/docs/latest) is a linux distribution designed specifically for running containers but does not currently come with support for ZFS.

[ZFS](http://zfsonlinux.org) is a very performant filesystem that supports error checking, snapshots, clones, native nfs and cifs support, and incremental backups. The ZFS on Linux project is a port of OpenZFS for Linux.

The motivation for this script is to be able to mount a ZFS filesystem to use as persistent storage on a Kubernetes controller node running CoreOS.

This script was written on CoreOS stable (1353.7.0), but in theory, will work on any version / channel (ie. stable, beta, alpha). It installs the latest release of ZFS on Linux which is based on OpenZFS and consists of a repository [zfs](https://github.com/zfsonlinux/zfs) which includes the upstream OpenZFS implementation and a repository [spl](https://github.com/zfsonlinux/spl) which is a shim to run OpenZFS on Linux.

Note that the script does not effect the filesystem that CoreOS is mounted on, it allows additional block devices to be mounted using the ZFS file system.

The script downloads the CoreOS build environment and then starts a temporary container running the build-environment to download and build the latest release of ZFS on Linux.

Because the /lib/modules and /usr/local directories are read-only on CoreOS, a read-write filesystem overlay is provided for both these directories.

The standard systemd unit files included from the ZFS source code are installed along with two additional systemd .mount units (lib-modules.mount and usr-local.mount) to mount the overlays and a zfs.service unit to load the ZFS kernel module.

At the completion of the installation the zfs module should be loaded and user space tools in the PATH, and zfs mounts should persist across reboots (if there is a zpool.cache file; see issues section).

The build command builds ZFS in the following directories and creates an tar.gz archive of these directories (which can be transferred to other machines).

1. **/opt/modules**: kernel modules (overlaid onto /lib/modules) 
2. **/opt/usr/local**: user space tools (overlaid onto /usr/local)

The install command extracts an archive file if available (otherwise the build command must have been already run), creates some links to the /etc/systemd and /etc/zfs directories, and install and configures the systemd unit files.

Hopefully this script will allow more people to experiment with ZFS on CoreOS to gain enough support that the CoreOS developers will bake it into CoreOS (note that this implementation does not gracefully handle updates to the CoreOS OS).

## References
This script is adapted from the instructions from:

1. https://coreos.com/os/docs/latest/kernel-modules.html  
2. https://github.com/zfsonlinux/zfs/wiki/Building-ZFS

The build process will download about 350 MB of files, and during installation, the corezfs folder will grow to over 3GB, so it must be run from a location with this much free space.

## Issues

1. This should really be baked into CoreOS (so hopefully this script is just a temporary stop-gap solution until the CoreOS developers include ZFS support natively)  
2. It is uncertain whether the kernel drivers will continue to work after a CoreOS update, or whether the script needs to be re-run to re-build them (further support for why it should be baked into CoreOS).  
3. This script would be much better and robust as a Makefile, but `make` is not included in the bare CoreOS tool set (please lobby the CoreOS developers to include it).  
4. This script tries to faithfully duplicate the stock ZFS installation with no customization from the upstream ZoL source. The upstream ZoL maintainers created two different systemd unit files to import zpools at boot. One method (zfs-import-scan.service) scans the /dev/disk/by-id/ directory for available pools and tries to import them. The second method (zfs-import-cache.service) imports a zpool defined in the /usr/local/etc/zfs/zpool.cache file. By default the systemd unit file for the first method is disabled, and the systemd unit file for the second method is enabled. There are a couple issues with this:  

    - The upstream zfs-import-cache.service unit file includes a `ConditionPathExists=/usr/local/etc/zfs/zpool.cache` clause so it won't run unless that file exists, and further, when it does run it has a hard-coded dependency on that file
    - That file isn't created as part of the default ZoL installation
    - That file isn't created when a new pool is created (ie. `zpool create pool /dev/disk/by-id/scsi0`)
    - There is actually meant to be one file per ZPool, so even if that file existed for one zpool, any additional zpools wouldn't be loaded since the systemd unit file only loads the pool defined in the cache file
    
    Ultimately this means that *By default, ZPools aren't imported  on reboot*. Use one of the work-arounds below to import ZPools in reboot:
    
     1. One workaround is to disable importing by cache file and instead scan the /dev/disk/by-id/ directory on each boot:
         
		 ```bash
		 sudo systemctl disable --now zfs-import-cache.service
		 sudo systemctl enable --now zfs-import-scan.service
		 ```
           
     2. A second workaround is to manually create the cache file when creating the ZPool (but this only works if there is only one ZPool since the zfs-import-cache-service systemd unit file is still hard coded to just the single cache file):
         
         ```bash
         sudo zpool create -o cachefile=/usr/local/etc/zfs/zpool.cache pool-01 /dev/disk/by-id/scsi0
         ```
     
     3. If you need more than one ZPool create a cache for each (see the work-around above), and then edit the /etc/systemd/system/zfs-import-cache.service file to load each (including commenting out the two lines shown below).
         
         ```bash
         [Unit]
         ...
         #ConditionPathExists=/usr/local/etc/zfs/zpool.cache
         
         [Service]
         ...
         #ExecStart=/usr/local/sbin/zpool import -c /usr/local/etc/zfs/zpool.cache -aN
         ExecStart=/usr/local/sbin/zpool import -c /usr/local/etc/zfs/pool-01.cache -aN; \
                   /usr/local/sbin/zpool import -c /usr/local/etc/zfs/pool-02.cache -aN; \
                   /usr/local/sbin/zpool import -c /usr/local/etc/zfs/pool-03.cache -aN
         ```

	4. In the stock ZFS installation the kernel drivers are loaded by the zfs-import-cache.service and zfs-import-scan.service, but by default, there is no zpool.cache file so the zfs-import-cache.service won't run because it has a condition that the zpool.cache file exists, and the zfs-import-scan.service is disabled therefore the kernel drivers won't be loaded.
	
	To ensure that the kernel drivers are loaded a new systemd service unit is included (zfs.service) to load the kerner drivers.
	
## Using ZFS
ZFS terminology uses the following two terms:

1. **ZFS Storage Pool** is an aggregation of one or more physical storage devices that are pooled together (optionally using raid). Storage pools are manipulated with the `zpool` command. The command to create a new storage pool (using the block device /dev/disk/by-id/scsi01 is `sudo zpool create -o compression=lz4 zpool01 /dev/disk/by-id/scsi01`. 
2. **ZFS Datasets** are allocations within the ZFS Storage Pool with a fixed set of properties or options. ZFS Datasets are manipulated with the `zfs` command. The command to create a new ZFS dataset in the ZFS storage pool created in the example above (zpool01) is `sudo zfs create zpool01/dataset01`.

Both the `zpool` and `zfs` commands must be run as root.

One way of thinking about ZFS is to compare it with a traditional single disk with partitions. In this comparison, think of a ZFS storage pool as being the physical disk, and ZFS datasets as the partitions on the disk.

This can be a useful comparison, but mostly by looking at where it falls apart:

1. In the traditional partitioned disk approach partitions have a fixed size, but ZFS datasets don't have a fixed size (although you can set quotas) and can grow until the storage space in the overall ZFS storage pool is exhausted.  
2. In the traditional partitioned disk approach, different partitions can be formatted with different filesystems (ie. ext4, fat, etc.), and formatting is done after the partition is created. In ZFS the entire storage pool is formatted as ZFS when it is created (using the `zpool create` command) and individual ZFS datasets do not need to be formatted like partitions do.  
3. A ZFS storage pool can't be mounted, but ZFS datasets can.

By default, when a ZFS storage pool is created a single ZFS dataset with the same name as the storage pool is created within that pool (ie. `zpool create zpool01 /dev/disk/by-id/scsi01` will create a ZFS storage pool called "zpool01" and a ZFS dataset called "zpool01" within that pool.

Additional ZFS datasets can be created, and ZFS datasets can be nested.

Each ZFS dataset has options that can be applied to it. The most common options are:

- **compression** - what type of compression to use (recommended to always set this to lz4)  
- **canmount** - whether the ZFS dataset can be mounted 
- **mountpoint** - where the ZFS dataset is mounted in the os (or empty if the ZFS dataset should not be mounted)  
- **readonly** - whether the ZFS dataset is read-only  
- **sharenfs** - settings for exporting the ZFS dataset using nfs  
- **sharesmb** - settings for exporting the ZFS dataset using samba  

Most options are inherited from the ZFS dataset's parent.

Options can be set at creation using the "-o" flag (ie. `sudo zfs create -o readonly=yes ro_dataset`), or can be altered using the `zfs set` command (ie. `sudo zfs set -o readonly=yes ro_dataset`).

By default, ZFS datasets will be mounted in the os filesystem root directory and have the same name as the ZFS dataset name (ie. the "zpool01" dataset in the example above will be mounted at "/zpool01"), but the mountpoint can be changed by changing the "mountpoint" option for the ZFS dataset. ZFS datasets don't actually have to be mounted in the os filesystem (this can be useful for storing backup data, especially when using the `zfs send` and `zfs receive` commands).

The most common ZFS storage pool operations are:

- `zpool create [-o option=value] <pool-name> <block-device>`: Create a ZFS storage pool  
- `zpool destroy <pool-name>`: Destroy a ZFS storage pool  
- `zpool list [-o <option1,option2,option3>]`: List all ZFS storage pools (show the options listed)  
- `zpool set -o <option=value>`: Change an option for an existing ZFS storage pool  
- `zpool get <option>`: See an option for an existing ZFS storage pool

The most common ZFS dataset operations are:

- `zfs create [-o option=value] <dataset>`: Create a ZFS dataset  
- `zfs destroy <dataset>`: Destroy a ZFS dataset 
- `zfs list [-o <option1,option2,option3>]`: List all ZFS datasets (show the options listed)  
- `zfs set -o <option=value>`: Change an option for an existing ZFS dataset  
- `zfs get <option>`: See an option for an existing ZFS dataset

As (a very contrived) example, consider the following partial folder structure on a server:

- /var/my_app
- /var/my_app/cache #should be on SSD
- /var/my_app/data
- /var/my_app/config #should be read-only

For this example, assume that scsi01, scsi02, and scsi03 are SSD drives and sda, sdb, and sdc are conventional drives.

The following commands will set-up ZFS storage pools and ZFS datasets to accommodate the structure shown above.

```bash
sudo zpool create -O compression=lz4 zpool-ssd raidz1 scsi01 scsi02 scsi03 #create ZFS storage pool on SSD media
sudo zpool create -O compression=lz4 zpool raidz1 sda sdb sdc #create ZFS storage pool on conventional hard drive
sudo zfs create -o mountpoint=/var/my_app zpool/my_app #create a ZFS dataset for my_app
sudo zfs create -o readonly=on zpool/my_app/config
sudo zfs create -o mountpoint=/var/my_app/cache zpool-ssd/cache
sudo mkdir /var/my_app/data #this is just a normal folder and there is no need to create a new ZFS dataset for it because it has all the same properties of it's parent
```

After running the commands above the os filesystem (from `find /var/my_app/ -type d`) looks like:

```bash
/var/my_app/
/var/my_app/cache
/var/my_app/data
/var/my_app/config
```

The ZFS datasets (from `sudo zfs list -o name,mounted,mountpoint,readonly,compression`) look like:

```bash
NAME                 MOUNTED  MOUNTPOINT          RDONLY  COMPRESS
zpool                    yes  /zpool                 off       lz4
zpool-ssd                yes  /zpool-ssd             off       lz4
zpool-ssd/cache          yes  /var/my_app/cache      off       lz4
zpool/my_app             yes  /var/my_app            off       lz4
zpool/my_app/config      yes  /var/my_app/config      on       lz4
```

This is a very contrived example to demonstrate how to have different ZFS datasets with different properties, and how ZFS datasets relate to os filesystem mount points.

## Snapshots and Clones
One of the great features of ZFS is the ability to take snapshots and make clones.

A snapshot is a read-only exact copy of a filesystem from a specific point in time (when the snapshot was taken).

A clone is an exact copy of a snapshot, but unlike a snapshot is both read/write.

A simplistic way to think about it is that a snapshot is a backup, and if you want to restore a backup you "clone" it back into existence.

### NFS and CIFS
ZFS has built-in support for serving filesystems over nfs and cifs (by setting different options for the filesystem). Refer to the zfs man page for details.

### Send and Receive
Another great feature of ZFS is the send and receive commands which allow incremental changes between two different snapshots be dumped/restored to/from a file, or more likely transferred over a network in an incredibly efficient way.

### Why make filesystems [instead of just nesting directories]?
When a filesystem is mounted, it behaves like a normal filesystem and can contain files and directories.

But note also that ZFS filesystems can be nested.

So in some installations, it is reasonable that there is only one ZFS filesystem, but in this case, everything will either have to be compressed or uncompressed (same with any other options), and it is only possible to take snapshots of the entire complete filesystem.

Making multiple filesystems allows fine grained control over different options for different things, so for example, if you had a server which included directories for both persistent data and also ephemeral cache (ie. requires space, but can be rebuilt if lost and doesn't need to be backed-up), you would probably create the following two filesystems:

```bash
sudo zfs create -o compression=lz4 zfs-pool/data
sudo zfs create -o compression=lz4 zfs-pool/cache
```

This will allow you to take snapshots of the data filesystem (and not include the cache filesystem).

If the cache folder were inside the data folder then I think the best solution is something like the following.

```bash
sudo zfs create -o compression=lz4 zfs-pool/data
sudo zfs create -o compression=lz4 -o mountpoint=/zfs-pool/data/cache zfs-pool/cache
```

This keeps the cache out of the data hierarchy in the ZPool, but mounts it in the correct location under the data hierarchy in the OS Filesystem.

Snapshots don't take up a lot of space (almost nothing), but there are also lots of various options that you may want to set which would cause you to set up a ZFS filesystem for it. It is also pretty easy to end up with several thousands of snapshots (especially when testing a backup rotation system) and the more there are the slower the ZFS user space tools work (ie. the `zfs list -t snapshot` command will take longer to list all of the snapshots).

Also, alot of the options are inherited and actions such as snapshotting can work recursively (ie. include all sub-filesystems), so when there are special cases like the cache case above, it usually works out better to not actually nest the ZFS filesystems (instead change the mount-point to make them nested in the os filesystem).

### Next Steps
Like most things, ZFS is fairly straightforward once you understand what the designers had in mind, but the devil is in the details, and it is recommended to read the complete documentation.

This introduction is not exhaustive, but hopefully it provides enough of a high level overview that the more detailed information at https://pthree.org/2012/04/17/install-zfs-on-debian-gnulinux/ is easier understood.
