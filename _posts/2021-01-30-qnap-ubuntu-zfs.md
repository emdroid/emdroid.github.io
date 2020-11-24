---
title: "QNAP NAS native Ubuntu Server installation (ZFS)"
header:
  teaser: /assets/images/posts/qnap-ubuntu.jpg
toc: true
categories:
  - Operating Systems
  - Linux
tags:
  - nas
  - server
  - os
  - linux
  - ubuntu
  - zfs
  - raid
---

![QNAP Ubuntu](/assets/images/posts/qnap-ubuntu.jpg){: .align-center .img-large}

Installing the Ubuntu Server on QNAP / RAID-5 ZFS encrypted root.

<!-- more -->

## 1. Introduction

This article describes installing the Ubuntu Linux Server on my [QNAP TS-453 Pro](https://www.qnap.com/en-us/product/ts-453%20pro) NAS machine natively, completely replacing the original [QNAP QTS](https://www.qnap.com/qts/) operating system.

{% capture notice_contents %}
**<a name="disclaimer">Disclaimer</a>**

This article describes an **advanced technique** to install the Ubuntu Linux Server directly onto a QNAP NAS box.
This is **not officially supported by QNAP**, so there is eventual chance of screwing things up (although the first section of the article provides the guidance on how to save and restore the original system, should it be needed).

It works well for my QNAP NAS model and to my best knowledge it should work on many other models as well (at least the i386-based ones), but I obviously **cannot guarantee that it will always work** (as I only have the experience with the single model indeed).

Also, if your machine is still covered by the warranty, you might want to think twice before doing this - it should be possible to recover to the previous state, but you **might lose the warranty** if the recovery wouldn't work for whatever reason (such recovery would most likely not be covered).

So as always: **<a name="disclaimer-2">USE AT YOUR OWN RISK, you have been warned!</a>** ;-)
{% endcapture %}

{% include notice level="danger" %}

**The motivation** to replace the original system:
- having the complete control over the system
- allowing some advanced configuration that is not possible with the standard QNAP system
- allows to use ZFS for the data (more rich features than the [Ext4 + MD-RAID](https://www.qnap.com/solution/qnap-ext4/en-us/) used by the QNAP originally)
- much faster startup times (the QNAP original system takes the astonishing ~ 5 mins to boot up completely)
- allows the whole system unattended encryption (via [downloading the encryption key from the network]({% post_url 2020-10-21-unlocking-linux-encrypted-root-from-network-usb %}))
- provides better performance eventually
- uniform setup with my other home machines (already having some [custom built machines]({% post_url 2020-07-13-project-ares-part-1-choosing-the-hardware %}) using the same or similar setup)
- and indeed the curiosity whether it can be done (_spoiler alert: it can!_)

On the other hand, this setup is more appropriate for advanced users (that already know Linux well), and the **less experienced users might actually prefer to keep the original QTS system**:
- the QTS is easier to use, having a nice graphical administrative interface (which the pure Linux Server setup will not have indeed)
- it is designed to run on the box, supporting all its features (the LCD panel, suspend/resume, notifications etc. - although many of these features can be supported in the Linux installation as well)
- still has very rich set of features (link aggregation, virtual networks, VM and Docker support, media and download station and a lot of other applications available)
- I myself was running the original QTS for a long time without much of complaints, including running virtual machines on there
- has a nice Android application that allows to access the QNAP NAS from the phone, checking the status, receive notifications and manage the system, including shutting it down or waking up over the network

{% capture notice_contents %}
**<a name="perf-warn">Performance warning</a>**

There is (was?) a **general recommendation** to have at least 8 GB of RAM (+ 1 GB per TB of storage) for the ZFS usage.
Although this might be a bit overstated, and there are reports of using ZFS with much less system memory (1-2 GB) [^1] [^2], I generally **wouldn't recommend to use ZFS with less than 4 GB of RAM**.

You might eventually get away with less memory for **single user NAS machines** (where simultaneous access of large amount of users isn't expected), which the home NAS boxes usually are; and eventually by setting the `"primarycache=metadata"` option (just caching the ZFS metadata, but not the actual data).
But in such "low memory" scenario you might eventually **consider using the XFS** (or Ext4) over MD-RAID/LUKS (or Truecrypt).

In  particular, I **upgraded my QNAP NAS box to 16 GB of RAM** in the past, and when using the ZFS the memory usage went to approx. 50 % (i.e. up to the 8 GB of RAM used by the ZFS) when just copying the data onto the disks over Samba (but sure, when there is a plenty of memory available, the ZFS can use more for caching the incoming data - with less memory the ZFS would most likely also use less RAM).
{% endcapture %}

{% include notice level="warning" %}

### 1.1. Alternatives

There are QNAP supported alternatives that might be interesting for the less experienced users:

**a) Virtualization station**:

- complete virtualization solution that comes integrated within the QNAP QTS operating system
- allows virtualization of various systems (Linux, Windows, Solaris, ...)
- based on QEMU (VirtIO)

**b) Linux Station**:

- allows installation of various Linux distributions (Ubuntu, Fedora, Debian) on top of QTS
- still using virtualization under the hood (LXC - Container Station required)
- it's basically Linux running as in the [Docker](https://www.docker.com/)
- can use the QTS at the same time

So depending on your needs, the above alternative solutions might be viable or even preferred for you (depending on what you want to achieve).

In my case I chose the **native Linux installation** (unsupported by QNAP) to gain the most control over the resources - the alternative solutions are still virtualized, therefore they have some limitations (for example cannot utilize ZFS on the bare metal or other direct access low level functionality).

### 1.2. QNAP NAS configuration

As already mentioned, the QNAP NAS box in question is the [QNAP TS-453 Pro](https://www.qnap.com/en-us/product/ts-453%20pro) in the following configuration:

- **CPU**: [Intel Celeron J1900](https://ark.intel.com/content/www/us/en/ark/products/78867/intel-celeron-processor-j1900-2m-cache-up-to-2-42-ghz.html) (2.0GHz, 4-core, 10 W TDP)
- **RAM**: 16 GB (2 x 8 GB) [Crucial DDR3L-1600 SODIMM](https://www.crucial.com/memory/ddr3/ct102464bf160b)
- **System disk**: [ADATA IUM01-512MFHS](https://www.amazon.co.uk/ADATA-Industrial-Grade-IUM01-Horizontal-Solid/dp/B01LQAYAPS) integrated 500 MB DOM (SLC)
- **RAID**: 4 x 4 TB 3.5" HDD (various Seagate / WD models)

The CPU is relatively weak, but at least it has 4 cores (and only 10 W TDP).
It is somewhat on the edge for the ZFS + virtualization (i5 would be much better option indeed, such QNAP machines exist, but are much more expensive).
It is a secondary machine as I have a [much more powerful one]({% post_url 2020-07-13-project-ares-part-1-choosing-the-hardware %}) that I built recently, but still want to use the QNAP box as the secondary system and backup option.

The memory should be good enough (was coming with 2 GB initially, replaced by the 2 x 8 GB modules on my own).

The main system flash disk is quite a small one in particular (only 500 MB), which is one of the most limiting parts (will be used just for the boot partition, as it cannot accommodate the whole Ubuntu installation).

### 1.3. The installation summary

In short, the article will explain the following installation of the Ubuntu Server Linux distribution:

1. The boot partition on the system 500 MB flash drive (Ext2 to not wear the flash drive too much, also disabling the access time updates)
2. Using the key vault encryption (the vault partition will reside on the 500 MB system drive as well)
3. The 4 x 4 TB drives will be fully dedicated to the ZFS pool in zraid1 (RAID-5 equivalent) configuration (will include the Ubuntu root + all data, all encrypted)
4. Unattended unlocking of the key vault partition from network (Azure blob storage) + USB fallback
5. ZFS ZVOL swap (not recommended in general because of eventual deadlock issues [^3], but having no better option for this particular configuration)
6. Hibernation (suspend to disk) not supported for now (because of the swap being on the ZFS)
7. Featuring the LCD panel usage

In particular, the intention is to **avoid creating multiple partitions** and/or ZFS pools on the RAID drives, and only have a single ZFS pool over all of them.
This has multiple reasons:
- it is very recommended to assign the whole physical disks to ZFS (the ZFS system takes over the drives completely, can manage the disk buffers, I/O scheduling etc. [^4])
- multiple ZFS pools (or ZFS pools combined with other partition types) can have performance issues (eventual simultaneous writes to different parts of the disk - when there is just a single ZFS pool, the ZFS optimizes and caches the writes and reads)
- it is the simplest to configure and install
- it is also the simplest to replace a failing disk eventually - just by replacing the drive and letting it to resilver (instead of having to setup the partitions on the replacement disk first, and eventually to resync multiple arrays)

**UEFI support**

The QNAP NAS box also supports the UEFI BIOS options.
But in my case I'm installing the system **in the legacy mode** for these particular reasons:
- the UEFI boot requires the primary drive to be GPT formatted and to have the EFI partition
- this is an issue with the only 500 MB available size of the primary flash disk
- the EFI partition minimal recommended size is 100 MB [^5]; but that applies to the Windows OS, the Ubuntu EFI takes much less space and it is possible to manage with just a 32 MB EFI partition  
(I tried and it worked, still having plenty of free space left there)
- but even with the 32 MB EFI that space could still be used for the boot partition instead, which is already on its limits with a 500 MB total primary drive (can only hold the maximum of 2 initrd boot images)
- the QTS installation on my machine also used the non-UEFI legacy startup anyway, so it is supported and working just fine

Just need to be careful to **use the non-UEFI USB boot option** (not the UEFI one) when starting the Ubuntu installation, so that the system can actually be installed (if started as UEFI, it would try to install the Ubuntu in the UEFI mode as well, complaining that the EFI partition is not available etc.).

It could be that your machine only supports UEFI and not legacy, in which case you might actually need to install in the UEFI mode.
This is however not being covered by the article at the moment.

## 2. Prerequisites

To start installing the OS, we'll need to do some initial preparation:

**Downloads**:

- [Ubuntu Desktop](https://ubuntu.com/download/desktop) Live DVD (needed for prep, moving to ZFS etc., eventually recovery)
- [Ubuntu Legacy Server](http://cdimage.ubuntu.com/ubuntu-legacy-server/releases/20.04/release/) (minimal install) or any other release you want to install (can also use the Live DVD if you'd want install the Desktop version)
- alternatively can also bootstrap a clean Ubuntu initial image (but I prefer to use the aforementioned Server image to have some basic setup done by the installer)

**Installation media**:

- empty USB flash drive for the Ubuntu Live Desktop (at least 4 GB size)
- the machine doesn't have a DVD drive, but could alternatively also use an external DVD drive
- a secondary USB flash drive for the Ubuntu Server installation image (or can reuse the Desktop installation media, but will need to switch the media multiple times)

**Disks**:

- additional USB flash as the temporary disk for the installation (the recommended size is at least 16 GB)
- this is required, because the system can't be migrated in-place (unless you'd be bootstrapping from scratch, that can be done directly)
- note that you cannot use the data disks for the temporary installation as those disk will then be fully reinitialized and used for the final ZFS install
- the data disks should ideally be in place already for creating the RAID-5 array (not possible to add disks to the ZFS VDEV later)

**Software**:

If installing from the USB, needs the software to create the bootable USB from the ISO image(s):

- Windows: [Rufus](https://sourceforge.net/projects/rufus.mirror/)
- Linux: various apps available, for example the [Startup Disk Creator](https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu#3-launch-startup-disk-creator) in Ubuntu

**Target machine peripherals**:

Not all tasks can be performed remotely, therefore the following devices are needed to perform the installation and/or migration:

- keyboard (eventually a mouse as well, but not completely necessary)
- display (monitor) with the correct connection cable (usually HDMI)
- these can be disconnected afterwards once the installation and the ZFS migration is done and the remote SSH access is working
- (local) network connection

### 2.1. Booting up the Ubuntu Live Desktop

For the backup and migration part the guide will be using the [Ubuntu Live Desktop](https://ubuntu.com/download/desktop) to perform the needed operations (doing the so-called "offline migration").

**a) Prepare the bootable USB with the Live Desktop image**:

- use the [Rufus](https://sourceforge.net/projects/rufus.mirror/) or a Linux tool to create the bootable USB from the ISO image (the Ubuntu Desktop DVD)
- alternatively burn the ISO image onto a DVD-R[W] if using an external DVD drive

**b) Boot from the USB flash** (or the DVD drive):

- connect the keyboard and screen
- connect the Ubuntu Live Desktop bootable USB (used the front USB in my case)
- (re)start the QNAP system
- on boot press Del to enter the BIOS
- locate the boot and eventually other options; in my case under "Boot":
  - **Quiet Boot**: Disabled (temporarily - better to see the bootup progress)
  - **Boot Option Priorities**: use +/- keys to move the USB as the first boot option (use the non-UEFI options)
  - "Save & Exit" / "Save Changes and Reset"

**c) When the Live Desktop boots up**:

- if offered, use the "*Try Ubuntu*" option to start the Live Desktop (use the "Tab" key if not having the mouse)

**d) Start the SSH server inside the Live Desktop** (optional, but highly recommended):

- allows to work conveniently from a remote PC (e.g. Windows), to copy the commands etc.
- open the terminal window: "*Activities*" (can use the "Windows key") / write "*term*", Enter

```bash
# set the password for the remote connection
passwd

# install the openssh-server (will start automatically)
sudo apt -y install openssh-server
```

(note that the password doesn't need to be particularly strong, only serves for the temporary Live Desktop connection over the local network)

- connect remotely (using PuTTY from Windows or ssh client from Linux)
  - host: _ubuntu_
  - user: _ubuntu_
  - pass: _the password you have set before_
- note that if using the Live Desktop multiple times, the SSH client may show the "*signature not matching*" warning, as it will create a new different identity each time the Live Desktop is started - this warning can be safely ignored in this case

## 3. Backing up the original system

### 3.1. Backup all data

Before performing the installation, make sure to backup all data from all the drives to another PC or NAS - creating the ZFS pool over the disks **will destroy all the existing data**!

### 3.2. Backup the original system (flash)

- [start the Ubuntu Live Desktop](#21-booting-up-the-ubuntu-live-desktop)
- connect an empty flash (FAT or exFAT formatted) to the back of the box (the faster blue USB 3.0 connectors recommended)
- copy the entire primary flash drive to a "_qnap-system.bin_" file on the USB:

```bash
# enter the admin console
sudo -i

# show all disks:
fdisk -l

# set the disks
# (check the disk model names)

# for example:

# - the small flash
DISK_FLASH=/dev/sdf

# - the empty USB for backup:
DISK_USB=/dev/sdg

# check the disks are the correct ones:
fdisk -l ${DISK_FLASH}
fdisk -l ${DISK_USB}

# mount the USB
mkdir -p /mnt/backup
mount ${DISK_USB}1 /mnt/backup

# backup the whole drive
dd if=${DISK_FLASH} of=/mnt/backup/qnap-system.bin bs=1M

# the sample output:
492+0 records in
492+0 records out
515899392 bytes (516 MB, 492 MiB) copied, 26.0126 s, 19.8 MB/s

# check the file
ls -al /mnt/backup

# unmount the backup USB
umount /mnt/backup
```

When having the original primary system copied:
- disconnect the backup USB drive
- connect to other PC and store the "_qnap-system.bin_" file to a safe place (ideally onto multiple locations to be sure to not lose it)

### 3.3. Restoring the original system

If anything goes wrong or you want to restore the original system for any other reason (e.g. before re-selling the unit):

- copy the previously stored "_qnap-system.bin_" file back to an USB flash drive (FAT or exFAT formatted)
- [start the Ubuntu Live Desktop](#21-booting-up-the-ubuntu-live-desktop)
- restore the primary flash drive image:

```bash
# enter the admin console
sudo -i

# show all disks:
fdisk -l

# set the disks
# (check the disk model names)

# for example:

# - the small flash
DISK_FLASH=/dev/sdf

# - the USB with the backup:
DISK_USB=/dev/sdg

# check the disks are the correct ones:
fdisk -l ${DISK_FLASH}
fdisk -l ${DISK_USB}

# mount the USB
mkdir -p /mnt/backup
mount ${DISK_USB}1 /mnt/backup

# check the backup file exists
ls -al /mnt/backup/qnap-system.bin

# restore the primary drive
dd if=/mnt/backup/qnap-system.bin of=${DISK_FLASH} bs=1M

# unmount the backup USB
umount /mnt/backup

# restart the machine
reboot
```

- the machine should reboot into the original QTS operating system now

## 4. Installing the new system

### 4.1. Preparing the disks

**a) Start the Live Desktop**:

- start the Ubuntu Live Desktop [as described before](#21-booting-up-the-ubuntu-live-desktop)

**b) Prepare the environment**:

```bash
# enter the admin console
sudo -i

# show all disks:
fdisk -l

# set the disks
# (check the disk model names)

# for example:
DISK_PRI=/dev/sdf
```

**c) Wipe all drives**:

{% capture notice_contents %}
**<a name="wipe-disk-warn">Attention</a>**

This step will irrevocably destroy all the existing data in the disks.

**Make sure you backed up all the data you need!**
{% endcapture %}

{% include notice level="danger" %}

- clearing all the drives to begin the new installation:

```bash
# wiping the primary drive
sgdisk -Z ${DISK_PRI}
wipefs --all ${DISK_PRI}

# wiping out the data drives
# (4 drives in the example)
sgdisk -Z /dev/sda
sgdisk -Z /dev/sdb
sgdisk -Z /dev/sdc
sgdisk -Z /dev/sdd

# might need to reboot first
# (the MD-RAID arrays might be locking up the drives)

wipefs --all /dev/sda
wipefs --all /dev/sdb
wipefs --all /dev/sdc
wipefs --all /dev/sdd
```

**c) Prepare the installation partitions**:

- prepare the boot partition (the legacy non-UEFI format):

```bash
# print the disk partitions
fdisk -l ${DISK_PRI}

# creating as MBR, not GPT
fdisk -c ${DISK_PRI}
```

- use the following commands:

```
# the boot partition:

n        (new partition)
<Enter>  (default: primary)
<Enter>  (default: 1)
<Enter>  (default: first available sector)
-32M     (leave 32MB for the key vault partition)
a        (make bootable)

# the key vault partition:

n        (default: new partition)
<Enter>  (default: primary)
<Enter>  (default: 2)
<Enter>  (default: first available sector)
<Enter>  (default: last available sector)

w        (write and exit)
```

- print again to check

```bash
fdisk -l ${DISK_PRI}
```

### 4.2. Installing the Ubuntu Server

In this step we will install the target system.

**a) Preparing the prerequisites**:

- if using different installation than the Ubuntu Desktop, use the [Rufus](https://sourceforge.net/projects/rufus.mirror/) or a Linux tool to create the bootable USB of the installation ISO image (e.g. the [Ubuntu Legacy Server](http://cdimage.ubuntu.com/ubuntu-legacy-server/releases/20.04/release/))
- alternatively burn the ISO image onto a DVD-R[W] if using an external DVD drive
- prepare another empty USB drive (min. 16 GB) for the temporary installation (will be migrated to the ZFS installation later)

**b) Start the Ubuntu system installer**:

- remove the Live Desktop USB and insert the Server USB  
  (or keep the Live Desktop if installing Ubuntu from there)
- in my case into the front USB connector
- insert an extra drive for the temporary installation ("backup") - in my case into the faster blue USB 3.0 connector on the back of the machine
- (re)start the machine, should start the installer in non-UEFI mode normally:
  - all the existing drives should be wiped, thus the USB installer drive should be the only viable boot option
  - eventually can [enter the BIOS and select the non-UEFI option as first](#21-booting-up-the-ubuntu-live-desktop)

**c) Installing the system**:

Will be installing the Ubuntu in this particular configuration:
- **boot**: using the primary small 500 MB flash drive /boot partition directly
  (already the final location)
- **root**: using the temporary extra USB drive (will be copied to the final ZFS location later)
- **GRUB**: installing the boot loader onto the primary flash drive directly (the final location)

That means:
- follow the Ubuntu installer as usual
  - see e.g. [here for the options I'm normally using]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}#31-installing-the-system)
- when coming to the **destination disk selection** ("partitioner") step:
  - partitioner: Manual
  - Boot:
    - the primary partition on the small flash (Adata 500 MB)
    - Use as: Ext2 file system
    - Mount point: /boot
    - Mount options: noatime, nodiratime
    - Label: boot
    - Done setting up the partition
  - Root:
    - use the temporary drive (USB)
    - if any partitions exist, remove them
    - select the free space
    - create a new partition
    - keep the size
    - keep all as root
    - Done setting up the partition
  - Finish partitioning and write changes to disk
- continue to complete the installation
- make sure to select these servers at the end:
  - **Samba file server**: needed for hostname resolution from windows  
  (and will also be used for the Windows sharing later)
  - **OpenSSH server**: needed for the remote SSH access

### 4.3. Preparing for the ZFS migration

- once the installation is done, remove the installation USB  
- **don't remove the secondary USB** (the system is currently starting from there!!!)
- restart the machine to boot it up
- should boot into the installed system
- SSH to the system from your primary PC (should already work with the user name and password set up during the installation)

**a) Update the system**:

```bash
# enter the root console for the administrative steps
sudo -i

# make sure the system is fully up-to-date
apt update
apt upgrade -y
```

**b) Install the necessary packages**:

- setting up packages for the non-UEFI (legacy) BIOS:

```bash
apt install -y \
    dosfstools mdadm linux-image-generic grub-pc \
    zfs-initramfs zsys zfs-zed zfsutils-linux \
    hwinfo gdisk vim
```

**c) Disable the hibernation**:

- as mentioned before, it will not be possible to use the hibernation in ths setup:

```bash
# disable the resume (hibernation) for now
echo "RESUME=none" > ${TARGET_PATH}/etc/initramfs-tools/conf.d/resume
```

**d) Power off fix**:

My QNAP machine needs this power off fix because of the USB3 waking the PC up immediately after shutdown:

- you can test whether ths is needed by performing `"init 0"` to shutdown the machine
- if it doesn't stay off and starts immediately again, you can try to apply this fix:

```bash
# edit the Grub configuration
vim /etc/default/grub

# add to GRUB_CMDLINE_LINUX_DEFAULT:
GRUB_CMDLINE_LINUX_DEFAULT="xhci_hcd.quirks=270336"

# update the Grub active setup
update-grub
```

Further information about this issue:
- [Ubuntu 16 reboots seconds after shutdown](https://askubuntu.com/a/1076172)
- [Bugzilla: System reboots after shutdown](https://bugzilla.redhat.com/show_bug.cgi?id=1257131)
- [Bugzilla: Poweroff doesn't work, it just reboots](https://bugzilla.kernel.org/show_bug.cgi?id=66171)
- [xhci: Switch PPT ports to EHCI on shutdown](https://git.kernel.org/pub/scm/linux/kernel/git/torvalds/linux.git/commit?id=e95829f474f0db3a4d940cae1423783edd966027)

**e) Enable sudo access without password** (optional):

- this setting allow the admin accounts to run the `"sudo"` commands without having to provide the password

{% capture notice_contents %}
**<a name="sudo-password-warn">Security warning</a>**

Allowing the admin users to run the `"sudo"` commands without password is a convenience, but **poses a security risk** - anyone who manages to access the computer with your user can then do virtually anything (without having to know the password).

However in order to do that, the attacker will need to gain the access to the admin user account in the first place.

Also keep in mind that **anyone with the physical machine access** can eventually gain the root access anyway (eg. by booting in the single user mode and changing the root password).
The encrypted root makes it a bit harder, but still not impossible (especially when the **unattended boot** is required - in such case the encryption keys still needs to be available to the boot process so they can be eventually accessed).

To truly protect the system the boot partition would either need to be password protected as well (implying having to provide the password on every boot), or the remote bootup SSH shell using eg. "dropbear" can be used (but it is still susceptible to the [MITM attack](https://en.wikipedia.org/wiki/Man-in-the-middle_attack) as the SSH server private key still needs to be in the initramfs then).
{% endcapture %}

{% include notice level="warning" %}

```bash
# open the sudoers editor
visudo

# replace the line (edit):
%sudo   ALL=(ALL:ALL) ALL
# by:
%sudo   ALL=(ALL:ALL) NOPASSWD: ALL
```

**f) Password-less SSH access** (optional):

- eventually you can setup the private key SSH login as described here: [SSH password-less login]({% post_url 2017-06-03-ssh-passwordless-login %})

**g) Connect the Live Desktop USB and restart**:

```bash
reboot
```

## 5. ZFS migration

{% capture notice_contents %}
**<a name="zfs_warn">Warning</a>**:

There steps work for the **Ubuntu** in particular, but might not work for other systems with different level of ZFS support.

For example, the **CentOS** needs somewhat different steps to install onto ZFS root.
{% endcapture %}

{% include notice level="warning" %}

{% capture notice_contents %}
**<a name="setup_warn">Important</a>**:

All the steps of the sections #5, #6 and #7 need to be performed as a single batch, the machine must not be restarted in the meantime (you'd have to start again from scratch if that happens).
{% endcapture %}

{% include notice level="danger" %}

### 5.1 Preparing the ZFS partitions and pools

**a) Boot up the Live Desktop** (if not already in there from the previous steps):

- use the steps described in the "[Booting up the Live Desktop](#21-booting-up-the-ubuntu-live-desktop)" section  
(including starting the "*ssh-server*")
- connect remotely so that you can copy the commands easily
- need to enter the BIOS and move the non-UEFI USB entries to front

**b) Enter the administrative console** (if not already there):

- this is for convenience to not have to type "`sudo`" in front of every command

```bash
sudo -i
```

**c) Install the needed packages**:

- use "*vim*" or any other editor you prefer
- note this installs the packages to the currently running Live DVD, not to the migrated installation

```bash
apt install -y zfs-initramfs vim curl

# stop the ZFS manager for now
systemctl stop zed
```

**d) Set the variables for the disks being used** (if not already done):

- **DISK_PRI**: The primary drive
- **DISK_BCK**: The secondary backup drive
- use your disk ids as appropriate

```bash
# prefer to use the disk-by-id to avoid confusion
# with the disk assignment
ls -al /dev/disk/by-id/

# set the disk ids appropriately
# (use the physical disk links without the "-partN")

# the primary boot drive
# - using the 512 MB flash drive
#   (will contain the /boot and key vault partitions)
# !!! note that this will be a different drive than before !!!
DISK_PRI=/dev/disk/by-id/<tab-complete-the-disk-path>

# the backup drive (temporary installation USB)
DISK_BCK=/dev/disk/by-id/<tab-complete-the-disk-path>

# my example:
DISK_PRI=/dev/disk/by-id/usb-ADATA_IUM01-512MFHS_120821561538001054-0\:0
DISK_BCK=/dev/disk/by-id/usb-SanDisk_Ultra_Fit_01014628e753eb95d79010b98428dd715b62d247c802f52ac4916b408e89735934dc0000000000000000000063883ea2ff1454008355810721289663-0\:0

# the root/data RAID-5 disks
DISK_RAID[1]=/dev/disk/by-id/<tab-complete-the-disk-path>
DISK_RAID[2]=/dev/disk/by-id/<tab-complete-the-disk-path>
DISK_RAID[3]=/dev/disk/by-id/<tab-complete-the-disk-path>
DISK_RAID[4]=/dev/disk/by-id/<tab-complete-the-disk-path>

# to test the order, use the "dd" and check which drive is
# blinking, then use the IDs accordingly, for example:
dd if=/dev/sda of=/dev/null bs=1M
# (break by Ctrl+C, can then also see the disks speed)

# example:
DISK_RAID[1]=/dev/disk/by-id/ata-ST4000DM000-1F2168_S301MHN3
DISK_RAID[2]=/dev/disk/by-id/ata-WDC_WD40EZRZ-00GXCB0_WD-WCC7K1KY4F1Y
DISK_RAID[3]=/dev/disk/by-id/ata-ST4000DM001-1FK17N_W4J141T2
DISK_RAID[4]=/dev/disk/by-id/ata-ST4000DM001-1FK17N_W4J13PJ7

# the encryption keys base path
KEYS_PATH=/vault/crypt

# the target path:
TARGET_PATH=/target
```

**e) Repartition the primary disk**:

- cleaning the RAID disks:

```bash
# show the current partitions on each disk
# !!! double-check that they are the correct ones !!!
# (will destroy all existing data)
# - check the IDs, disk sizes, existing partitions etc.
sgdisk -p ${DISK_RAID[1]}
sgdisk -p ${DISK_RAID[2]}
sgdisk -p ${DISK_RAID[3]}
sgdisk -p ${DISK_RAID[4]}

# clear all existing partitions
# (if having LVM volumes there, those need to be cleared first)
for n in $(seq 1 4); do wipefs --all ${DISK_RAID[$n]} ; done
```

**f) Setting up the key vault partition**:

- will be using LUKS for the small 32 MB key vault partition:

```bash
# create the vault partition LUKS key
# - using the Base64 encoding to only have printable chars in the key
mkdir -p /etc/crypt/
dd if=/dev/urandom bs=512 skip=4 count=4 iflag=fullblock | base64 -w 0 \
    > /etc/crypt/vault.key
chmod -R go-rwx /etc/crypt/

# or copy an existing key
vim /etc/crypt/vault.key

# check with "cat"
cat /etc/crypt/vault.key

# if printed a newline at the end, remove it
truncate -s -1 /etc/crypt/vault.key


# setup the vault partition encryption
cryptsetup luksFormat -q -c aes-xts-plain64 -s 512 -h sha256 \
    -d /etc/crypt/vault.key ${DISK_PRI}-part2
cryptsetup luksOpen -d /etc/crypt/vault.key ${DISK_PRI}-part2 vault

# create the vault filesystem
mkfs.ext2 /dev/mapper/vault

# mount the volume
mkdir -p /vault
mount /dev/mapper/vault /vault

# check the free space
df -h /vault
```

{% capture notice_contents %}
**<a name="key_warn">Warning</a>**:

- do not forget to copy the vault key out to a safe location (ideally also encrypted disk, password protected zip etc.) e.g. on your desktop PC
- can use e.g. [WinSCP](https://winscp.net/eng/download.php) on Windows, the "scp" command on Linux or the clipboard after printing the key on the screen by `"cat /etc/crypt/vault.key"`
- it will be needed for eventual recovery etc.
{% endcapture %}

{% include notice level="warning" %}

**g) <a name="keys_generate"></a>Generate the encryption keys**:

```bash
mkdir -p ${KEYS_PATH}/zfs

# the ZFS root partition key
dd if=/dev/urandom bs=32 skip=4 count=1 iflag=fullblock \
    > ${KEYS_PATH}/zfs/root.key

# the ZFS user partition key
dd if=/dev/urandom bs=32 skip=4 count=1 iflag=fullblock \
    > ${KEYS_PATH}/zfs/home.key

# adjust the permissions
chmod -R go-rwx ${KEYS_PATH}
```

{% capture notice_contents %}
**<a name="key_note">Notice</a>**:

As the key vault partition will not be mirrored, it is also recommended to copy the keys (or the whole partition via dd) out to another safe location, should a recovery be needed eventually.
{% endcapture %}

{% include notice level="info" %}

**h) Create the "root" ZFS pool**:

```bash
mkdir -p ${TARGET_PATH}

# create the main RAID pool
# - will be using "raidz1" which is RAID-5 effectively
# 
zpool create -f \
    -o ashift=12 \
    -O acltype=posixacl \
    -O canmount=off \
    -O compression=lz4 \
    -O dnodesize=auto \
    -O normalization=formD \
    -O atime=off \
    -O relatime=on \
    -O xattr=sa \
    -O mountpoint=/ \
    -R ${TARGET_PATH} \
    rpool \
    raidz1 \
    ${DISK_RAID[1]} \
    ${DISK_RAID[2]} \
    ${DISK_RAID[3]} \
    ${DISK_RAID[4]}

# for SSD: Set the auto-trim
zpool set autotrim=on rpool

# test the pool status
zpool status rpool
```

{% capture notice_contents %}
**<a name="rpool_note">Notes</a>**:

- the `"ashift=12"` parameter is used again to enforce the 4k sector size alignment
- the `"atime=off"` and `"relatime=on"` used to speed up the disk operations
- when installing on SSD, the auto-trim needs to be used so that the TRIM works properly  
- the array is created over the whole physical disks, which has some advantates for the ZFS maintenance
(in particular, the ZFS subsystem can manage the disk itself, reportedly switching off the "standard" IO scheduler and providing its own) [^4]
- the ZFS subsystem still creates partitions over the disks, while leaving some small space at the end to accommodate for eventual slight disk size differences
- the pool itself is not encrypted, the encryption will be done on the dataset level [^6]
{% endcapture %}

{% include notice level="info" %}

**i) Create the ZFS datasets** (this is Ubuntu specific, see the notes):

```bash
# create the rpool main datasets
zfs create \
    -o canmount=off \
    -o mountpoint=none \
    rpool/ROOT

# create the Ubuntu-specific root datasets
# - root encrypted via the "root" key
zfs create \
    -o canmount=noauto \
    -o mountpoint=/ \
    -o com.ubuntu.zsys:bootfs=yes \
    -o com.ubuntu.zsys:last-used=$(date +%s) \
    -o encryption=aes-256-gcm \
    -o keyformat=raw \
    -o keylocation=file://${KEYS_PATH}/zfs/root.key \
    rpool/ROOT/ubuntu

# create the main user dataset
# (encrypted via the "home" key)
zfs create \
    -o canmount=off \
    -o mountpoint=none \
    -o setuid=off \
    -o encryption=aes-256-gcm \
    -o keyformat=raw \
    -o keylocation=file://${KEYS_PATH}/zfs/home.key \
    rpool/USERDATA

# mount the main data sets
zfs mount rpool/ROOT/ubuntu

# list the datasets to validate
zfs list

# create the specific root datasets for /srv, /usr, /var
zfs create \
    -o com.ubuntu.zsys:bootfs=no \
    rpool/ROOT/ubuntu/srv

zfs create \
    -o com.ubuntu.zsys:bootfs=no \
    -o canmount=off \
    rpool/ROOT/ubuntu/usr

zfs create \
    -o com.ubuntu.zsys:bootfs=no \
    -o canmount=off \
    -o setuid=off \
    -o exec=off \
    rpool/ROOT/ubuntu/var

# create the underlying specific datasets
zfs create rpool/ROOT/ubuntu/usr/local

zfs create rpool/ROOT/ubuntu/var/lib
zfs create rpool/ROOT/ubuntu/var/lib/AccountServices
zfs create rpool/ROOT/ubuntu/var/lib/apt
# - the DPKG dataset needs to have exec enabled
#   so that the scripts might work correctly
zfs create -o exec=on rpool/ROOT/ubuntu/var/lib/dpkg
zfs create rpool/ROOT/ubuntu/var/lib/NetworkManager

zfs create rpool/ROOT/ubuntu/var/games
zfs create rpool/ROOT/ubuntu/var/log
zfs create rpool/ROOT/ubuntu/var/mail
zfs create rpool/ROOT/ubuntu/var/snap
zfs create rpool/ROOT/ubuntu/var/spool
zfs create rpool/ROOT/ubuntu/var/www

# - the tmp dataset needs to have exec enabled so that the eventual scripts might work
zfs create -o exec=on rpool/ROOT/ubuntu/var/tmp
chmod 1777 ${TARGET_PATH}/var/tmp

zfs create -o com.ubuntu.zsys:bootfs=no rpool/ROOT/ubuntu/tmp
chmod 1777 ${TARGET_PATH}/tmp

# create the user directories
zfs create \
    -o com.ubuntu.zsys:bootfs-datasets=rpool/ROOT/ubuntu \
    -o canmount=on \
    -o mountpoint=/root \
    rpool/USERDATA/root

TARGET_USER=<admin-user>

zfs create \
    -o com.ubuntu.zsys:bootfs-datasets=rpool/ROOT/ubuntu \
    -o canmount=on \
    -o mountpoint=/home/${TARGET_USER} \
    rpool/USERDATA/${TARGET_USER}

# eventually repeat for each existing user
# (check the original system for the user names and ids)

# list the datasets to validate again
zfs list

# list the folders to check properly mounted
ls -al ${TARGET_PATH}/
ls -al ${TARGET_PATH}/home/
ls -al ${TARGET_PATH}/home/${TARGET_USER}

# check the encryption:
#   1. Expect root encrypted
zfs get encryption ${TARGET_PATH}/
#   2. Expect home/$TARGET_USER encrypted
zfs get encryption ${TARGET_PATH}/home/${TARGET_USER}
```

{% capture notice_contents %}
**<a name="datasets_note">Notes</a>**:

- note that the original guides use additional UUID for the datasets, likely to be able to install multiple different versions, eventually a dual-boot; however I was installing only one system here, therefore I wasn't using the UUID part
- the server dataset layout is following the Ubuntu recommendations [^7]
- we are creating separate dataset per user, which is the recommended setup [^8] (this allows to have per-user snapshots and also to remove the user dataset completely including the snapshots when the user is to be removed)
{% endcapture %}

{% include notice level="info" %}

### 5.2. Copying the existing installation back from the temporary drive

**a) Mount the temporary installation volume**:

```bash
# create the mount point
mkdir -p /mnt/backup

# mount the backup volume
mount ${DISK_BCK}-part1 /mnt/backup
```

**b) Copy the installation onto the ZFS**:

- using the `"rsync"` to copy all including the permissions, attributes etc.)

```bash
# copy the installation root
rsync -avhPXI --info=progress2 --exclude=/swapfile /mnt/backup/. ${TARGET_PATH}/

# list the folders to check the data are copied properly
ls -al ${TARGET_PATH}/
ls -al ${TARGET_PATH}/home/
ls -al ${TARGET_PATH}/home/${TARGET_USER}

# unmount the backup
umount /mnt/backup
```

- note that the "_/boot_" partition is not mounted and copied as it is already in its final location

## 6. Preparing the boot environment (encryption)

This part is preparing the initramfs modules for using the root partition encryption on ZFS and using the key vault partition to store the keys.

The encryption setup will use the unattended encryption unlocking method described in this previous article: [Unlocking Linux encrypted root from network / USB]({% post_url 2020-10-21-unlocking-linux-encrypted-root-from-network-usb %})

### 6.1. Setting up the USB access

The USB serves as the fallback option (primarily the encryption keys should be retrieved from the network).

**a) Enabling USB drives automount**:

- this rule enables the key to be retrieved from any USB drive:

```bash
# set the rule for all USB disks
echo 'SUBSYSTEMS=="usb", DRIVERS=="usb", SYMLINK+="usbkey%n"' \
    > ${TARGET_PATH}/etc/udev/rules.d/99-custom-usb.rules
```

**b) The initramfs modules**:

- some extra modules are needed for setting up the network connection and the USB device mounting

```bash
  cat <<EOF >> ${TARGET_PATH}/etc/initramfs-tools/modules
sha256
aes-x86_64
aes_generic
crypto_api
dm-crypt
dm-mod
netboot
scsi_dh
usbcore
usbhid
usb_storage
EOF
```

- eventually check the setup

```bash
less ${TARGET_PATH}/etc/initramfs-tools/modules
```

### 6.2. The network setup

This is needed to have the network access during the bootup (to retrieve the key vault encryption key from the network).

**a) The network initramfs initialization script**:

- the initramfs module to configure/initialize the network during the boot phase:

```bash
  cat <<EOF > ${TARGET_PATH}/etc/initramfs-tools/scripts/init-premount/configure-network
#!/bin/sh
# Initialize the network
PREREQ=""
prereqs()
{
    echo "\$PREREQ"
}
case \$1 in
prereqs)
    prereqs
    exit 0
    ;;
esac

. /scripts/functions

configure_networking
EOF
```

- adjust the permissions:

```bash
chmod +x ${TARGET_PATH}/etc/initramfs-tools/scripts/init-premount/configure-network
```

**b) The network setup initramfs hook**:

- needed for copying the additional tools and setting up the network options  
(like DNS and the resolver)

```bash
  cat <<EOF > ${TARGET_PATH}/etc/initramfs-tools/hooks/init-network
#!/bin/sh

PREREQS=""

case \$1 in
    prereqs) echo "\${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

# the commands needed
copy_exec $(which curl)
copy_exec $(which ping)

# the DNS libraries
add_dns "\${DESTDIR}/"

# the resolver configuration
copy_file config "$(readlink -f /etc/resolv.conf)" "/etc/resolv.conf"
EOF
```

(ping is just added so that it can be eventually used to check the network status if something fails)

- adjust the permissions:

```bash
chmod +x ${TARGET_PATH}/etc/initramfs-tools/hooks/init-network
```

### 6.3. The key vault initramfs setup

**a) Adding the vault partition key onto network**:

- add the key vault partition encryption key into some internet blob storage solution (for example Microsoft Azure) as described in this article:  
  [Unlocking Linux encrypted root from network]({% post_url 2020-10-21-unlocking-linux-encrypted-root-from-network-usb %}#4-setting-up-the-azure-blob-storage)
- [Security considerations]({% post_url 2020-10-21-unlocking-linux-encrypted-root-from-network-usb %}#2-security-considerations)

**b) The key vault unlock script**:

- this script will retrieve the encryption key from the network or USB and unlock the key vault partition

```bash
# set the SAS token URL
# (replace the <sas_token_url> by your URL)
# !!! IMPORTANT: Use the apostrophes to avoid special char interpretation !!!
VAULT_KEY_SAS='<sas_token_url>'

# create the key vault unlock script
mkdir -p ${TARGET_PATH}/etc/crypt/scripts

# update the key vault script
  cat <<EOF > ${TARGET_PATH}/etc/crypt/scripts/unlock-vault.sh
#!/bin/sh
# Load the key vault partition key.

PREREQ=""
prereqs() {
    echo "\$PREREQ"
}

case \$1 in
    prereqs)
        prereqs
        exit 0
    ;;
esac


load_az() {
    # eventual errors should go to stderr
    curl --insecure -s -N '${VAULT_KEY_SAS}'
}

load_usb() {
    [ -b /dev/usbkey ] || return 1

    # if device exists then output the keyfile from the usb key 
    dd if=/dev/usbkey bs=512 skip=4 count=4 | base64 -w 0
}

get_key() {
    load_az || load_usb
}

export PATH=/sbin:/usr/sbin:/bin:/usr/bin

get_key
EOF
```

- adjust the permissions:

```bash
chmod +x ${TARGET_PATH}/etc/crypt/scripts/unlock-vault.sh
chmod -R go-rwx ${TARGET_PATH}/etc/crypt/
```

**c) The key vault mount script**:

```bash
  cat <<EOF > ${TARGET_PATH}/etc/initramfs-tools/scripts/local-top/vault-init
#!/bin/sh
PREREQ=""
prereqs()
{
    echo "\$PREREQ"
}
case \$1 in
prereqs)
    prereqs
    exit 0
    ;;
esac

. /scripts/functions

export PATH=/sbin:/usr/sbin:/bin:/usr/bin

mkdir -p /vault

VAULT_DEV="\$(findfs PARTUUID="$(blkid -s PARTUUID -o value ${DISK_PRI}-part2)")"

/etc/crypt/scripts/unlock-vault.sh | cryptsetup luksOpen -d - \${VAULT_DEV} vault

# open read-only by default
mount -t ext2 -o ro /dev/mapper/vault /vault
EOF
```

- set the permissions:

```bash
chmod +x ${TARGET_PATH}/etc/initramfs-tools/scripts/local-top/vault-init
```

**d) The key vault unmount script**:

```bash
  cat <<EOF > ${TARGET_PATH}/etc/initramfs-tools/scripts/local-bottom/vault-fini
#!/bin/sh
PREREQ=""
prereqs()
{
    echo "\$PREREQ"
}
case \$1 in
prereqs)
    prereqs
    exit 0
    ;;
esac

. /scripts/functions

export PATH=/sbin:/usr/sbin:/bin:/usr/bin

# move to the root location (kept read-only)
mkdir -p \${rootmnt}/vault
mount -o move /vault \${rootmnt}/vault
EOF
```

- set the permissions:

```bash
chmod +x ${TARGET_PATH}/etc/initramfs-tools/scripts/local-bottom/vault-fini
```

**e) The initramfs hook to copy the unlock scripts**:

- copying the vault init dependencies

```bash
  cat <<EOF > ${TARGET_PATH}/etc/initramfs-tools/hooks/init-vault
#!/bin/sh

PREREQS=""

case \$1 in
    prereqs) echo "\${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

copy_exec $(which base64)
copy_exec $(which findfs)

copy_file script /etc/crypt/scripts/unlock-vault.sh
EOF
```

- set the permissions:

```bash
chmod +x ${TARGET_PATH}/etc/initramfs-tools/hooks/init-vault
```

## 7. Finalizing the system setup

### 7.1 Finalizing the ZFS migration

**a) Setup the fstab**:

```bash
vim ${TARGET_PATH}/etc/fstab
```

- remove or comment out the original mountpoins for root etc. and swap 
  (you can just comment out the existing entries by using the `"#"` character)
- keep just the the "_/boot_" partition and eventually EFI if you need to have one (for UEFI installation)
- the swap file will not be used for now
- the ZFS mountpoints are not needed because of being mounted automagically (via zsys)

**b) Update the Grub boot loader configuration**:

```bash
vim ${TARGET_PATH}/etc/default/grub

# add the line
GRUB_RECORDFAIL_TIMEOUT=5

# add to the GRUB_CMDLINE_LINUX_DEFAULT options
# (to allow start from a degraded array, should a failure happen)
GRUB_CMDLINE_LINUX_DEFAULT="bootdegraded=true init_on_alloc=0"

# disable the OS prober by adding:
GRUB_DISABLE_OS_PROBER=true
```

**e) Chroot to the target system to finalize the setup**:

```bash
# prepare to enter the target system chroot
mount --rbind /dev     ${TARGET_PATH}/dev
mount --rbind /dev/pts ${TARGET_PATH}/dev/pts
mount --rbind /proc    ${TARGET_PATH}/proc
mount --rbind /sys     ${TARGET_PATH}/sys
mount --rbind /run     ${TARGET_PATH}/run

# enter the chroot
chroot ${TARGET_PATH} /usr/bin/env \
    DISK=${DISK_PRI} \
    bash --login

# mount all the boot partitions according to fstab
mount -a
# check if mounted properly
mount | grep boot
ls -al /boot
```

**f) Update the Grub configuration and the boot image**:

```bash
# test if the Grub understands the boot filesystem
grub-probe /boot
# - should print "ext2"

# update the Grub configuration
update-grub

# re-create the initramfs image
update-initramfs -c -k all
# ignore the eventual cryptsetup warnings - should still work

# check if the cryptkeys are present in the initramfs
lsinitramfs -l /boot/initrd.img-<tab-complete-the-img-path> | less
# check for:
# - the "etc/crypt/scripts/unlock-vault.sh" file
# - the "scripts/init-premount/configure-network" file
# - the "scripts/local-top/vault-init" file
# - the "scripts/local-bottom/vault-fini" file
# - the "usr/bin/curl" file
```

**g) Install the GRUB boot loader**:

- non-UEFI Bios (optional, not needed actually if installed directly):

```bash
# install the grub
grub-install --target=i386-pc ${DISK}
```

**h) Finalize the ZFS setup**:

```bash
# enable the zfs zed service
# (to auto-mount the ZFS pools properly on startup)
systemctl enable zfs-zed.service
systemctl enable zfs.target
systemctl start zfs-zed.service

# disable the grub fallback service
systemctl mask grub-initrd-fallback.service

# ZFS cache cleanup
mkdir -p /etc/zfs/zfs-list.cache
touch /etc/zfs/zfs-list.cache/rpool

# create the cacher link
ln -s /usr/lib/zfs-linux/zed.d/history_event-zfs-list-cacher.sh /etc/zfs/zed.d

# create the cache files
zed -F &

cat /etc/zfs/zfs-list.cache/rpool

fg
# press Ctrl-C

# update the mountpoints
sed -Ei "s|/target/?|/|" /etc/zfs/zfs-list.cache/*

df -h /boot

umount /boot

# exit the chroot
exit

# unmount and export the zpool
mount | grep -v zfs | tac | awk '/\/target/ {print $3}' | \
    xargs -i{} umount -lf {}

zpool export -a

# reboot and test
reboot
```

- make sure to remove the Ubuntu Live USB or DVD before rebooting

Now the basic system installation should be ready.

{% capture notice_contents %}
**<a name="zfs-reboot-info">Warning</a>**

For some reason it is necessary to reboot twice, as for the first time the user directories (root, <target_user>) do not mount for some reason.

Therefore it is necessary to reboot once again, at the second boot all the disks should be mounted.

The likely reason are the ZFS mount ordering and caches, it might need to adjust the mount order for the first time (in particular with regard to the key location). But not completely sure about the exact reason yet.
{% endcapture %}

{% include notice level="warning" %}

### 7.2. Creating the ZFS data sets

**a) Creating the data encryption key**:

- the key vault partition is mounted read-only by default, we will remount it as read-write and generate the new key:

```bash
# remount the key vault read/write
mount -o remount,rw /vault

# generate the data partition key
# (stored on the encrypted vault partition)
dd if=/dev/urandom bs=32 skip=4 count=1 iflag=fullblock \
    > /vault/crypt/zfs/data.key
chmod go-rwx /vault/crypt/zfs/data.key

# remount again back read-only
mount -o remount,ro /vault
```

**b) Creating the ZFS datasets for data**:

- in particular, to organize stuff keep in mind that each separate dataset can create its own COW snapshots
- note that you can eventually even use different keys for separate encrypted datasets

```bash
# create the main dataset
zfs create \
    -o canmount=off \
    -o mountpoint=none \
    rpool/DATA

# create the main mount data set
# (setting up the encryption)
zfs create \
    -o canmount=noauto \
    -o mountpoint=/data \
    -o encryption=aes-256-gcm \
    -o keyformat=raw \
    -o keylocation=file:///vault/crypt/zfs/data.key \
    rpool/DATA/data

# mount the data set
zfs mount rpool/DATA/data
```

### 7.3. Setting up the SWAP

In this section we'll setup the ZFS ZVOL swap.

As already mentioned in the beginning, this is generally not recommended because of eventual deadlock issues [^3].
But for this setup we don't have any better option (if we want to use the whole disks for the ZFS pool), therefore we'll still do it (in fact, after running te machine in this setup for some months, I didn't see any issue with this).

- this is an optional step as the swap should only be needed on rare occasions, provided you have enough RAM (might eventually be needed if VM&zwj;s are used and the memory is overcommitted)

- note that in this configuration the swap is unencrypted (this is not addressed for now, might have a further look on this later)

```bash
# create the swap ZVOL
zfs create -V 8G -b $(getconf PAGESIZE) -o compression=zle \
      -o logbias=throughput -o sync=standard \
      -o primarycache=metadata -o secondarycache=none \
      -o com.sun:auto-snapshot=false rpool/swap

# format the swap volume
mkswap -f /dev/zvol/rpool/swap

# add the swap volume to the fstab
echo "/dev/zvol/rpool/swap none swap defaults 0 0" >> /etc/fstab

# switch the swap on
swapon -av
```

### 7.4. Maintenance and reporting configuration

The following articles describe the setup necessary for maintenance, monitoring and reporting:
- [Linux postfix SMTP relay e-mail delivery]({% post_url 2020-10-26-linux-postfix-smtp-relay %})
- [Disk health monitoring and reporting]({% post_url 2020-11-14-disk-health-monitoring-and-reporting %})
- [Using desktop drives in RAID: TLER (ERC), APM, AAM]({% post_url 2020-11-18-desktop-drives-raid-tler-apm-aam %})
- [Automatic shutdown and wake-up with rtcwake]({% post_url 2020-11-10-automatic-shutdown-and-wakeup-with-rtcwake %})

The options used for my particular QNAP setup:

- disk monitoring and auto-testing ("_/etc/smartd.conf_"):

```
DEFAULT -d removable -n standby -W 0,35,40 -H -l error -l selftest -f -m root -M diminishing -M exec /usr/share/smartmontools/smartd-runner
/dev/sda -s (L/../(01|16)/./11|S/../../6/15)
/dev/sdb -s (L/../(04|19)/./11|S/../../6/17)
/dev/sdc -s (L/../(07|22)/./11|S/../../7/15)
/dev/sdd -s (L/../(10|25)/./11|S/../../7/17)
```

- the automatic suspend and resume (suspending to RAM - can use "off" instead of "mem" for more safe full shutdown):

```bash
# the scheduling script
  cat <<EOF > /usr/local/sbin/wakemgr
#!/bin/sh

schedule_wake() {
    rtcwake -m mem -t $(date -d  "$1" +%s)
}

export PATH=/sbin:/usr/sbin:/bin:/usr/bin

schedule_wake "$1" >> /var/log/suspend-resume.log
EOF


# set the permissions
chmod +x /usr/local/sbin/wakemgr

# setup the crontab
crontab -e


MAILTO=<your-user-mail>


# setup the power management schedule
 0  3  *  * 1-5   /usr/local/sbin/wakemgr  "9:00"
 0  3  *  * 6-7   /usr/local/sbin/wakemgr  "9:00"
```

### 7.5. QNAP LCD setup

The [QNAP TS-453 Pro](https://www.qnap.com/en-us/product/ts-453%20pro) NAS box has the front LCD panel, that can be utilized in th Ubuntu Linux configuration by using the [LCDproc](http://manpages.ubuntu.com/manpages/bionic/man1/lcdproc.1.html).

**a) LCDproc installation**:

```bash
apt install -y lcdproc
```

**b) The daemon setup**:

```bash
vim /etc/LCDd.conf
```

- setting up for the QNAP LCD:

```
# add or change
[server]
Driver=icp_a106

ServerScreen = no

Backlight = open
Heartbeat = open

WaitTime = 10

[icp_a106]
Device=/dev/ttyS1
Size = 16x2
```

- note the display size setting at the end

- the screen switch time has been increased to 10s

**c) Enabling the service**:

```bash
systemctl enable LCDd
systemctl restart LCDd
```

**d) The screens setup**:

- the screens setup:

```bash
# setup the config
vim /etc/lcdproc.conf
```

- my specific setings:

```
[Iface]
# Show screen
Active=True

# main network interface
Interface0=enp5s0
Alias0=LAN1

# eventually other LAN connections
# (but usually better to just show the primary)
#Interface1=enp6s0
#Alias1=LAN2
#Interface2=enp9s0
#Alias2=LAN3
#Interface3=enp10s0
#Alias3=LAN4

[Load]
# Show screen
Active=True
# Min Load Avg at which the backlight will be turned off [default: 0.05]
LowLoad=1.0
# Max Load Avg at which the backlight will start blinking [default: 1.3]
HighLoad=12.0

# enable the other screens you want:
- CPU
- Memory
- TimeDate
- UpTime
```

**e) The systemd service setup**:

- not done by default by the package

```bash
# create the service file
  cat <<EOF >> /etc/systemd/system/lcdproc.service
[Unit]
Description=lcdproc Service
After=syslog.target LCDd.service

[Service]
Type=forking
# execute the lcdproc (use -c for non-default config)
ExecStart=/usr/bin/lcdproc
# wait to show the LCCd welcome message for 10 sec
ExecStartPre=/bin/sleep 10
TimeoutStartSec=30
# restart if crashed
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

- enable the service:

```bash
systemctl enable lcdproc
systemctl restart lcdproc
```

## 8. System recovery

This section describes the steps for eventual recovery of the system, should something go wrong.

Note that if using the encryption, you'll need the encryption key (that you hopefully saved to a secure location beforehand).

**a) Boot up the Live Desktop**:

- [start the Ubuntu Live Desktop](#21-booting-up-the-ubuntu-live-desktop) section (including starting the "*ssh-server*")
- connect remotely so that you can copy the commands easily
- enter the administrative console

```bash
sudo -i
```

**b) Install the needed packages**:

- use "*vim*" or any other editor you prefer
- note this installs the packages to the currently running Live DVD, not to the migrated installation

```bash
apt install -y zfs-initramfs vim

# stop the ZFS manager for now
systemctl stop zed
```

**c) Set the variables for the disk being used**:
- **DISK**: The primary drive
- use your disk ids as appropriate

```bash
# prefer to use the disk-by-id to avoid confusion with the disk assignment
ls -al /dev/disk/by-id/

# set the disk ids appropriately
# (use the physical disk links without the "-partN")

# the primary drive
DISK_PRI=/dev/disk/by-id/<tab-complete-the-disk-path>

# my example:
DISK_PRI=/dev/disk/by-id/usb-ADATA_IUM01-512MFHS_120821561538001054-0\:0
```

**d) Decrypt the key vault partition**:

- copy the key back to the Live Desktop environment e.g. via [WinSCP](https://winscp.net/eng/download.php) on Windows, the "scp" command on Linux or via the clipboard by:

```bash
# restore the vault key
mkdir -p /etc/crypt/

vim /etc/crypt/vault.key

# copy the key via the clipboard or alike

# check with "cat"
cat /etc/crypt/vault.key

# if printed a newline at the end, remove it
truncate -s -1 /etc/crypt/vault.key
```

- decrypt and mount the key vault partition

```bash
# open the vault
cryptsetup luksOpen -d /etc/crypt/vault.key ${DISK_PRI}-part2 vault

# mount the volume
mkdir -p /vault
mount -o ro /dev/mapper/vault /vault
```

**e) Mound the ZFS volumes**:

```bash
# create the target dir
mkdir /target

# import the ZFS pool
zpool export -a
zpool import -f -N -R /target rpool

# check the pool status
zpool status rpool

# load all the ZFS keys
# (should load the keys from the key vault)
zfs load-key -a

# mount all datasets
zfs mount rpool/ROOT/ubuntu

zfs mount -a

# validate the USER datasets mounted
mount | grep ROOT
mount | grep USER
mount | grep DATA

# check if mounted properly
ls -al /target
ls -al /target/home
```

**f) Chroot to the target system to repair**:

```bash
# prepare to enter the target system chroot
mount --rbind /dev     /target/dev
mount --rbind /dev/pts /target/dev/pts
mount --rbind /proc    /target/proc
mount --rbind /sys     /target/sys
mount --rbind /run     /target/run

# enter the chroot
chroot /target /usr/bin/env \
    DISK=${DISK_PRI} \
    bash --login

# mount all additional disks from the /etc/fstab
mount -a

# check boot partitions mounted
mount | grep boot
```

**g) Perform the fixes**:

- do the actions necessary to recover / fix the system

**h) Finalize when done**:

```bash
# unmount the boot partitions
umount /boot

# exit the chroot
exit

# unmount and export the zpool
mount | grep -v zfs | tac | awk '/\/target/ {print $3}' | \
    xargs -i{} umount -lf {}

zpool export -a

# reboot and see if everything works
reboot
```

## Resources and references

- [Ubuntu: How can I control HDD spin down time?](https://askubuntu.com/a/767616)
- [zfsonlinux: HOWTO use a zvol as a swap device](https://github.com/zfsonlinux/pkg-zfs/wiki/HOWTO-use-a-zvol-as-a-swap-device)

LCDproc:
- [LCDproc on Ubuntu](https://htpc.xyz/en/2019/11/05/how-to-install-lcdproc-on-ubuntu/)
- [Debian-Installation auf dem QNAP TS-509 Pro](https://www.pro-linux.de/artikel/2/1530/3,treiber-teil-1.html)
- [man: LCDproc](http://manpages.ubuntu.com/manpages/bionic/man1/lcdproc.1.html)
- [man: LCDd](http://manpages.ubuntu.com/manpages/bionic/man8/LCDd.8.html)
- [man: lcdproc-config](https://www.mankier.com/5/lcdproc-config)

Notes:

[^1]: [reddit: So you think ZFS needs a ton of RAM for a simple file server? Think again.](https://www.reddit.com/r/DataHoarder/comments/3s7vrd/so_you_think_zfs_needs_a_ton_of_ram_for_a_simple/)
[^2]: [Linux Tech Tips: ZFS Memory requirements](https://linustechtips.com/topic/738402-zfs-memory-requirements/)
[^3]: [OpenZFS: Swap deadlock in 0.7.9](https://github.com/openzfs/zfs/issues/7734)
[^4]: [reddit: Formatting ZFS to use whole disk vs. partition](https://www.reddit.com/r/zfs/comments/enxxyx/formatting_zfs_to_use_whole_disk_vs_partition/)
[^5]: [Microsoft: UEFI/GPT-based hard drive partitions](https://docs.microsoft.com/en-us/windows-hardware/manufacture/desktop/configure-uefigpt-based-hard-drive-partitions)
[^6]: [reddit: ZoL 0.8.0 encryption: don't encrypt the pool root!](https://www.reddit.com/r/zfs/comments/bnvdco/zol_080_encryption_dont_encrypt_the_pool_root/)
[^7]: [ZFS focus on Ubuntu 20.04 LTS: ZSys dataset layout](https://didrocks.fr/2020/06/16/zfs-focus-on-ubuntu-20.04-lts-zsys-dataset-layout/)
[^8]: [ZFS NFS home directories: Each user as a separate ZFS dataset?](https://www.reddit.com/r/zfs/comments/bgsr0l/zfs_nfs_home_directories_each_user_as_a_separate/)
[^9]: [Ubuntu: Full disk encryption howto](https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019)

{% include abbrev domain="computers" %}
