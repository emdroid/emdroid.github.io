---
title: "Project Ares: Part IV - Ubuntu Server on ZFS (encrypted root)"
header:
  teaser: /assets/images/posts/nas-ares/zol.png
toc: true
sidebar:
  nav: "fs-ares"
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

![NAS OS](/assets/images/posts/nas-ares/zol.png){: .align-center .img-large}

Sharing the experience of building a home NAS/VM server.

The part 4 describes the steps taken to install the chosen OS (Ubuntu Server 20.04 LTS) onto a LUKS encrypted ZFS root partition in RAID-1 (mirror) configuration.

<!-- more -->

## 1. Introduction

In the [previous part]({% post_url 2020-07-18-project-ares-part-3-os-and-file-system-considerations %}) the decision has been made to use the **[Ubuntu Legacy Server 20.04 LTS](http://cdimage.ubuntu.com/ubuntu-legacy-server/releases/20.04/release/)** as the NAS operating system.
In addition, it should be installed onto a ZFS file system, using the encrypted root partition.

{% capture notice_contents %}
**<a name="zfs-native-info">Important changes</a>**

Contrary to the initial intent described in the [Part 3 (OS and filesystem)]({% post_url 2020-07-18-project-ares-part-3-os-and-file-system-considerations %}#52-the-data-disks), I ended up using the native ZFS encryption.

**The original version of the article is still available here:  
[Project Ares: Part IV - Ubuntu Server on ZFS (original version)]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root-initial %})**

I made this decision after performing the speed measurements and finding out that the **native ZFS encryption is not as bad** as some of the Internet reports suggested [^3], and in fact using the native ZFS encryption + RAID is performing significantly better than using the non-native combination of MD-RAID + LUKS/VeraCrypt + pure ZFS:
- _MD RAID/VeraCrypt/ZFS write speed_: 220 - 250 MB/s
(depending on the MD-RAID strip size)
- _native ZFS/raid/encryption write speed_: 300 - 310 MB/s
(using the currently most secure "aes-256-gcm" encryption)
- these are results for the data drives (magnetic rotational 4 x 8 TB HDD&zwj;s in RAID-5 configuration)

However this is valid for my specific hardware setup where I did the measurements, and I cannot guarantee that it will be always true with any hardware setup. 

The use of the native ZFS encryption has the following significant advantages:
- the data are only encrypted once and on the filesystem level  
(with external encryption + RAID they might be encrypted multiple times e.g. for the mirrored disks)
- the native ZFS RAID can also be used  
(use of external encryption like LUKS/VeraCrypt enforces the usage of external RAID solution like MD-RAID)
- this simplifies the whole setup heavily

The ZFS encryption seems to be reasonably safe, is using the AES standard (currently up to 256-bit master key) and to the date I didn't find any published flaws or issues, see also these videos for the details about the encryption implementation:
- [ZFS Native Encryption by Tom Caputi](https://www.youtube.com/watch?v=frnLiXclAMo)
- [Securing the Cloud with ZFS Encryption by Jason King](https://www.youtube.com/watch?v=kFuo5bDj8C0)
{% endcapture %}

{% include notice level="warning" %}

This guide is primary targeted to the [NAS build described in the previous parts]({% post_url 2020-07-15-project-ares-part-2-putting-it-together %}), however the steps are not too build-specific so they can be used on virtually any machine.

The following steps are primarily based on the various Ubuntu ZoL articles:

- [OpenZFS: Ubuntu 20.04 Root on ZFS](https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2020.04%20Root%20on%20ZFS.html) [^1]
- [Installing Ubuntu on a ZFS root, with encryption and mirroring](https://saveriomiroddi.github.io/Installing-Ubuntu-on-a-ZFS-root-with-encryption-and-mirroring/) [^2]

**Extras to the above guides:**

- using the LUKS encryption instead of the native ZFS encryption (which still suffers of a poor performance [^3])
- setting up unattended boot of the encrypted root instead of requiring a password for each boot

Note that the **Ubuntu Desktop Installer now supports direct installation onto ZFS** as an experimental feature (since version 19.04) [^4], however it doesn't fit this usecase for multiple reasons:

- only supported in the Desktop image (no Server image support)
- only full-disk installation with the default partitioning
- root encryption not supported

{% capture notice_contents %}
**<a name="sec-warn">Security warning</a>**

- note that the unattended boot is required when the machine sometimes needs to be e.g. restarted remotely  
(or be switching on/off automatically according to the schedule)
- but as the password will be stored inside the initramfs on the unencrypted boot partition, it can be eventually retrieved, so this is not completely safe
- this can be mitigated by setting up to retrieve the root key from the network (e.g. blob storage like Azure, constraining the requester IP so that it can be only read from a particular network) and/or a key stored on the USB stick (as a backup option) during the boot process, which is my actual setup and will be addressed in a follow-up article
{% endcapture %}

{% include notice level="danger" %}

## 2. Prerequisites

If you'd like to convert an existing system installation to ZFS, it is also possible, you'd just need a secondary disk to be used as temporary to do the conversion.

To start installing the OS, we'll need to do some initial preparation:

**Downloads**:

- [Ubuntu Desktop](https://ubuntu.com/download/desktop) Live DVD (needed for prep, moving to ZFS etc., eventually recovery)
- [Ubuntu Legacy Server](http://cdimage.ubuntu.com/ubuntu-legacy-server/releases/20.04/release/) (minimal install) or any other release you want to install (can also be the Live DVD if you'd install the Desktop version)
- alternatively can also use bootstrap a clean Ubuntu initial image (but I prefer to use the aforementioned Server image to have some basic setup done byt the installer)

**Disks**:

- the primary disk to install
- secondary disk to serve as temporary for moving to the ZFS  
(can also be the one to be used as the system disk mirror later)
- you can eventually also use an additional USB flash as the temporary disk  
(the recommended size is at least 16 GB)

**Installation media**:

- empty USB flash drive for the installation image (at least 4 GB size)
- the machine doesn't have a DVD drive, but could alternatively also use an external DVD drive

**Software**:

If installing from USB, needs SW to create a bootable USB from the ISO image:

- Windows: [Rufus](https://sourceforge.net/projects/rufus.mirror/)
- Linux: various apps available, for example the [Startup Disk Creator](https://ubuntu.com/tutorials/create-a-usb-stick-on-ubuntu#3-launch-startup-disk-creator) in Ubuntu

**Target machine peripherals**:

Not all tasks can be performed remotely, therefore the following devices are needed to perform the installation and/or migration:

- keyboard (eventually a mouse as well)
- display (monitor)
- these can be disconnected afterwards once the installation and the ZFS migration is done and the remote SSH access is working
- (local) network connection

## 3. The OS installation

### 3.1. Installing the system

{% capture notice_contents %}
**<a name="install_note">Note</a>**:

If you'll be converting an pre-existing installation, you can skip to the step 3.2: "[Preparing for the ZFS migration](#32-preparing-for-the-zfs-migration)".

In such case the following steps might need to be changed accordingly (if using LVM, multiple partitions etc.).
{% endcapture %}

{% include notice level="info" %}

{% capture notice_contents %}
**<a name="install_note">Warning</a>**:

These steps are specific to the Ubuntu installation. The steps for other systems might be different.
{% endcapture %}

{% include notice level="warning" %}

**a) Prepare the bootable USB with the installer image**:

- use the [Rufus](https://sourceforge.net/projects/rufus.mirror/) or a Linux tool to create the bootable USB from the installation ISO image (e.g. the Ubuntu Legacy Server)
- alternatively burn the ISO image onto a DVD-R[W] if using a DVD drive

**b) Boot from the USB flash (or the DVD)**:

- connect the USB disk / DVD drive to the target machine
- start the PC
- if no OS is installed yet, should boot from the USB automatically; otherwise need to select it in the boot selection menu (usually opens after pressing the `F8` key) or in the Bios (`Del`, `F2`)
- the USB boot sometimes also needs to be enabled in the Bios (when you disabled it before - is usually enabled by default)

**c) Start the installer**:

- use the "*Install Ubuntu Server*" option (or the available install option if another image is used)

**d) Set the initial install options** (here the options I used for the Ubuntu Server):

- **Language**: *English / English*  
(I prefer to keep the system in English; or select your preferred language)
- **Location**: *XXXX / XXX*  
(select the location the machine will be used, in particular it determines the timezone)
- **Locale**: *English / US UTF-8*  
(highly recommended as primary even though you might be using a different language normally)
- **Hostname**: *hhhhhhhh*  
(hostname only without the domain - this is preferred in Ubuntu)
  - use only characters and numbers (no diacritic), eventually dashes (hyphens)
  - the first character should be a letter
  - 15 chars max (NetBios limit when accessed from Windows)
- **Username and password** of the main user (will become the administrator)
- **Timezone**: asks if you want to use the auto-detected timezone  
(if installing for a different timezone than where it will be used later, select "No")

**e) Select the destination disk**:

- use the "*Guided - use entire disk*" option - the partitioning not important for now, as we will redo it later anyway during the ZFS migration
- select the primary disk the system should be installed onto  
(do not install onto the secondary disk to save the copying step later - we need all the boot records to point to the first disk properly)
- let the installer remove any previous partitions on the drive (but double check it is the correct drive!)
- this will install Ubuntu onto a **single Ext4 partition**, which is the easiest option for the later migration

**f) Continue the installation**:

- **Proxy**: Enter the proxy if using any, otherwise leave empty
- **Updates**: *Install security updates automatically*  
(highly recommended - otherwise need to install any updates manually)

**g) Selection of the server packages**:

- **Samba file server**: Needed for hostname resolution from windows  
(and will also be used for the Windows sharing later)
- **OpenSSH server**: Needed for the remote SSH access

**h) Let the installation finish, eventually restart to check that it is booting**:
- try to connect with SSH to test that the machine is found on the network  
(note that if using Windows, it might take some time for the hostname to become available)

After this we are done with the basic installation and can continue with the ZFS migration step.

If the system works, we will no longer need the Server installation USB.
So the USB flash can be re-purposed for the Live DVD, which will be used in the following steps.

### 3.2. Preparing for the ZFS migration

To perform the ZFS migration of the installed system, we'll need to perform some extra preparation steps.

In general we'll need to install additional packages to support the ZFS file system (and some tools to be available for the migration itself).

**a) Update the system**:

```bash
# enter the root console for the administrative steps
sudo -i

# make sure the system is fully up-to-date
apt update
apt upgrade -y
```

**b) Install the necessary packages**:

- for UEFI Bios:

```bash
apt install -y \
    dosfstools mdadm linux-image-generic shim-signed \
    grub-efi-amd64 grub-efi-amd64-signed \
    zfs-initramfs zsys zfs-zed zfsutils-linux \
    gdisk vim
```

- for non-UEFI (legacy) Bios:

```bash
apt install -y \
    dosfstools mdadm linux-image-generic grub-pc \
    zfs-initramfs zsys zfs-zed zfsutils-linux \
    gdisk vim
```

**c) Enable sudo access without password** (optional):

- this setting allow the admin accounts to run the `"sudo"` commands without having to provide the password

{% capture notice_contents %}
**<a name="zfs-native-info">Security warning</a>**

Allowing the admin users to run the `"sudo"` commands without password is a convenience, but **poses a security risk** - anyone who manages to access the computer with your user can then do virtually anything (without having to know the password).

However in order to do that, the attacker still needs to gain access to the admin user account anyway.

Also keep in mind that **anyone with the physical machine access** can eventually gain the root access anyway (eg. by booting in the single user mode and changing the root password to gain the access).
The encrypted root makes it a bit harder, but still not impossible (especially when the **unattended boot** is required - in such case the encryption keys still needs to be available to the boot process so they can be eventually accessed).

To truly protect the system the boot partition would either need to be password protected as well (implying having to provide the password on every boot), or the remote bootup SSH shell using eg. "dropbear" can be used (but it is still susceptible to the MITM attack as the SSH server private key then still needs to be in the initramfs).
{% endcapture %}

{% include notice level="danger" %}

```bash
# open the sudoers editor
visudo

# replace the line (edit):
%sudo   ALL=(ALL:ALL) ALL
# by:
%sudo   ALL=(ALL:ALL) NOPASSWD: ALL
```

**d) Connect the Live Desktop USB and restart**:

```bash
reboot
```

### 3.3. Booting up the Live Desktop

For the migration part the guide will be using the [Ubuntu Desktop](https://ubuntu.com/download/desktop) Live USB to perform the needed operations (doing a so-called "offline migration").

Steps to create and use the Live DVD:

**a) Prepare the bootable USB with the Live DVD image**:

- use the [Rufus](https://sourceforge.net/projects/rufus.mirror/) or a Linux tool to create the bootable USB from the ISO image (the Ubuntu Desktop DVD)
- alternatively burn the ISO image onto a DVD-R[W] if using an internal or external DVD drive

**b) Boot from the USB flash** (or the DVD drive):

- use the same steps as in the [system installation](#31-installing-the-system) part

**c) When the Live Desktop boots up**:

- use the "*Try Ubuntu*" option to start the Live Desktop

**d) Start the SSH server inside the Live Desktop** (optional, but highly recommended):

- allows to work conveniently from a remote PC (e.g. Windows), to copy the commands etc.
- open the terminal window: "*Activities*" / write "*term*", Enter

```bash
# set the password for remote connection
passwd

# install the openssh-server (will start automatically)
sudo apt -y install openssh-server
```

(note that the password doesn't need to be particularly strong, only serves for the temporary Live DVD connection)

- connect remotely (using PuTTY from Windows or ssh client from Linux)
  - host: ubuntu
  - user: ubuntu
  - pass: the password you have set before
- note that if using the Live DVD multiple times, the SSH client may show the "*signature not matching*" warning, as it will create a new different identity each time the Live DVD is started - this warning can be safely ignored in this case

### 3.4. Backing up the original installation

In this part we will prepare the pre-existing system for the ZFS migration. What we generally need to do is the following:

- copy the existing installation to the secondary drive
- repartition the primary drive for the ZFS
- copy the installation back to the new ZFS partitions
- re-create the initramfs images so that the system can boot from the ZFS

**Prerequisites**:

- a secondary disk to backup the existing installation
- the Live DVD bootable USB (or burned DVD if using an external DVD drive)

**a) Boot up the Live Desktop**:

- use the steps described in the "[Booting up the Live Desktop](#33-booting-up-the-live-desktop)" section  
(including starting the "*ssh-server*")
- connect remotely so that you can copy the commands easily

**b) Enter the administrative console**:

- this is for convenience to not have to type `"sudo"` in front of every command

```bash
sudo -i
```

**c) Set the variables for the disks being used**:
- **DISK[1]**: The primary drive
- **DISK[2]**: The secondary backup drive
- use your disk ids as appropriate

```bash
# prefer to use the disk-by-id to avoid confusion with the disk assignment
ls -al /dev/disk/by-id/

# set the disk ids appropriately
# (use the physical disk links without the "-partN")

# the primary drive
DISK[1]=/dev/disk/by-id/<tab-complete-the-disk-path>
# the secondary drive
DISK[2]=/dev/disk/by-id/<tab-complete-the-disk-path>
```

**d) Prepare the backup partition on the secondary disk**:
- will be using the "`sgdisk`" command

```bash
# list the existing partitions
sgdisk -p ${DISK[2]}
# !!! double check it is the correct secondary drive you want to use !!!
# (otherwise change the DISK[2] variable appropriately)

# remove any existing partitions
# (note that if there are any LVM volumes currently active, they'd need to be removed first)
wipefs --all ${DISK[2]}

# create a new LVM partition for backing up the primary disk
sgdisk -n1:0:0 -t1:8300 ${DISK[2]}

# check by listing the partitions
sgdisk -p ${DISK[2]}

# format the backup volume
mkfs.xfs -f ${DISK[2]}-part1
```

**e) Prepare the mount points and mount the volumes**:

- note that the first primary disk partition (marked as EPS / EFI) is not copied, we will re-create it afterwards  
(if migrating a dual boot system, this might not work so well, then you actually might need to copy that partition too - this is outside of the scope of this guide)

```bash
# create the mount points
for s in system backup; do mkdir -p /mnt/$s ; done

# list the primary disk partitions
sgdisk -p ${DISK[1]}
# - if you have just one, it will be the root partition
# - otherwise check the sizes which one is the root/boot/eventually home etc.
#   (skip the first EFI / Microsoft 512 MB partition)

# mount the primary volumes
# (default setup creates just a single ${DISK[1]}-part5 ext4 partition)
mount ${DISK[1]}-part5 /mnt/system
# eventually the boot, home and other volumes, if any
mount ${DISK[1]}-part3 /mnt/system
mount ${DISK[1]}-part2 /mnt/system/boot
mount ${DISK[1]}-part4 /mnt/system/home
# etc.

# if having LVM volumes, mount them accordingly

# mount the backup volume
mount ${DISK[2]}-part1 /mnt/backup

# test that the volumes are mounted
mount | grep /mnt/system
mount | grep /mnt/backup
```

**f) Copying the data**:
- using "`rsync`" (copy including any attributes)

```bash
# copy the data including all attributes
rsync -avPX --exclude=/swapfile /mnt/system/ /mnt/backup/

# check the volume data look the same
ls -al /mnt/system
ls -al /mnt/backup
# eventually the boot and root volumes, if any
ls -al /mnt/system/boot
ls -al /mnt/backup/boot
ls -al /mnt/system/home
ls -al /mnt/backup/home
# etc.

# unmount the volumes
for s in system backup; do umount /mnt/$s ; done
```

**g) Cleaning up the original data**:

- this is only necessary if you originally had LVM volumes used in the existing installation

```bash
# if primary disk used LVM, remove it's volume groups
# !!! make sure that everything is copied properly to the secondary drive !!!
# (if something missing, will be lost after this step)

# list the current volumes
lvs --noheadings -o <lv_path>

# remove the volumes and group of the primary disk (if any)
# !!! be careful to NOT remove the "backup" volumes and volume group !!!
lvremove <vg_name>
vgremove <vg_name>

# remove the physical volume assignment
pvremove /dev/<tab-complete-the-partition-path>
```

Now we are done with the system backup and prepared to perform the ZFS migration.

## 4. ZFS installation

### 4.1 Preparing the ZFS partitions and pools

{% capture notice_contents %}
**<a name="zfs_warn">Warning</a>**:

There steps work for the **Ubuntu** in particular, but might not work for other systems with different level of ZFS support.

For example, the **CentOS** cannot boot from ZFS at the moment and requires a "regular" boot partition (can't even boot from LVM).
{% endcapture %}

{% include notice level="warning" %}

{% capture notice_contents %}
**<a name="zfs_warn">Note</a>**:

Only the UEFI systems are covered here - for non-UEFI (legacy) systems the GRUB boot loader installation and the partitioning might be different.
{% endcapture %}

{% include notice level="info" %}

**a) Boot up the Live Desktop** (if not already in there from the previous steps):

- use the steps described in the "[Booting up the Live Desktop](#33-booting-up-the-live-desktop)" section  
(including starting the "*ssh-server*")
- connect remotely so that you can copy the commands easily

**b) Enter the administrative console** (if not already there):

- this is for convenience to not have to type "`sudo`" in front of every command

```bash
sudo -i
```

**c) Install the needed packages**:

- use "*vim*" or any other editor you prefer
- note this installs the packages to the currently running Live DVD, not to the migrated installation

```bash
apt install -y zfs-initramfs mdadm vim

# stop the ZFS manager for now
systemctl stop zed
```

**d) Set the variables for the disks being used** (if not already done):

- **DISK[1]**: The primary drive
- **DISK[2]**: The secondary backup drive
- use your disk ids as appropriate

```bash
# prefer to use the disk-by-id to avoid confusion with the disk assignment
ls -al /dev/disk/by-id/

# set the disk ids appropriately
# (use the physical disk links without the "-partN")

# the primary drive
DISK[1]=/dev/disk/by-id/<tab-complete-the-disk-path>
# the secondary drive
DISK[2]=/dev/disk/by-id/<tab-complete-the-disk-path>
```

**e) Repartition the primary disk**:

- will be using the `"sgdisk"` command
- this presumes the disks will use mirroring at the end
- see the swap partition notes and other information below

```bash
# list the existing partitions on the primary disk
sgdisk -p ${DISK[1]}
# !!! double check it is the correct drive you want to use !!!
# (otherwise change the DISK[1] variable appropriately)

# remove any existing partitions
# (note that if there are any LVM volumes currently active,
#  they should be removed first)
wipefs --all ${DISK[1]}

# UEFI Bios: create the EFI partition
sgdisk -n1:0:+512M -t1:EF00 -c1:"EFI System Partition" ${DISK[1]}

# Non-UEFI Bios: create the GRUB partition
# (needed for GPT)
sgdisk -n1:0:+512M -t1:EF02 -c1:"BIOS Boot" ${DISK[1]}

# create the ZFS boot partition
sgdisk -n2:0:+2G -t2:BE00 -c2:"ZFS Boot" ${DISK[1]}

# create the ZFS root partition
# leaving space for the SWAP hibernation partition
# (see notes below)
sgdisk -n3:0:-72G -t3:BF00 -c3:"ZFS Root" ${DISK[1]}

# create the SWAP partition
sgdisk -n4:0:0 -t4:8200 -c4:"Swap" ${DISK[1]}

# check by listing the partitions
sgdisk -p ${DISK[1]}

# UEFI only: format the EFI partition
mkdosfs -F 32 -s 1 -n EFS ${DISK[1]}-part1
```

{% capture notice_contents %}
**<a name="part_note">Notes</a>**:

- the EFI partition is not necessary if the computer is not using the UEFI Bios; most modern PCs have UEFI though  
(one particular example where UEFI still might not be used are virtual machines)
- we create the "EFI-like" partition though, as there is a partition needed for the GRUB boot loader (although only approx 1 MB is actually needed, but we are creating an "EFI-like" size to be able to eventually migrate to UEFI Bios if needed)
- if you have different size drives, you need to make sure that the root partition on the larger drive matches the root partition on the smaller drive, otherwise the RAID-1 could not be created  
(you can then use the rest or part of the larger drive as a SWAP partition)
- the **SWAP partition is highly recommended** even if you have a lot of memory [^5]
- the **SWAP partition can be used for hibernation**, so if you plan to use it, size it  appropriately  
(in the above example I reserved 72 GB for the SWAP partition, which is the recommended size for 64 GB RAM when hibernation is to be used, although currently only having 32 GB RAM installed; but I anticipate that I might eventually want to increase the RAM to 64 GB and I don't want to have to re-partition everything then)
- if the root partition is encrypted, it is highly recommended to encrypt the SWAP partition as well, which will be part of the following setup
- it is possible to hibernate to an encrypted SWAP partition [^6], however it is a bit more complex to setup and is not part of this guide (I may create an article on that later)
- note that if using 2 drives for mirror recovery, it is recommended to also "mirror" the SWAP partition onto the secondary drive, otherwise the system will not boot if the first drive fails - will not find the swap partition on the secondary drive  
(will not use an exact mirror, as the secondary drive is of a somewhat different size, but will setup the secondary disk swap the same way so it can be used if the first drive fails - that is also why I actually made the SWAP partition 80 GB on the primary instead of just the 72 GB recommended, as my secondary drive is approx. 10 GB smaller than the primary one thus want to have at least 64 GB of swap available on the secondary drive so that it can hibernate as well)
- if you don't want to use the hibernation, you can eventually make the swap partition much smaller [^5], or not use it altogether
{% endcapture %}

{% include notice level="info" %}

{% capture notice_contents %}
**<a name="part_note">Warning</a>**:

If you'll be using a secondary drive for mirror, **make sure that the secondary drive is either of the same or bigger size than the primary drive, or size the partitions appropriately** (so that they can fit the secondary drive).

If the secondary drive is smaller, you can eventually use the rest of the primary drive as swap (if not mirrored) and/or additional non-redundant volume(s).

However note that to be able to start if the primary disk fails, those extra partitions will need to use the `"nofail"` fstab flag to not break the startup in the case of failover scenario.
{% endcapture %}

{% include notice level="warning" %}

**f) <a name="keys_generate"></a>Generate the encryption keys**:

- using the `"hexdump"` command to convert the raw binary keys to hex format  
(in order to only include "printable" chars)

```bash
# generate the ZFS root partition key
mkdir -p /etc/crypt/zfs/init
dd if=/dev/urandom bs=32 skip=4 count=1 iflag=fullblock | hexdump -ve '/1 "%02x"' \
    > /etc/crypt/zfs/init/root.key

# the home partition key
dd if=/dev/urandom bs=32 skip=4 count=1 iflag=fullblock | hexdump -ve '/1 "%02x"' \
    > /etc/crypt/zfs/home.key
```

{% capture notice_contents %}
**<a name="key_warn">Warning</a>**:

- do not forget to copy the key out to a safe location (ideally also encrypted disk, password protected zip etc.) e.g. on your desktop PC
- can use e.g. [WinSCP](https://winscp.net/eng/download.php) on Windows, the "scp" command on Linux or the clipboard after printing the key on the screen by `"cat /etc/crypt/zfs/init/root.key"`
- it will be needed for eventual recovery etc.
{% endcapture %}

{% include notice level="warning" %}

{% capture notice_contents %}
**<a name="swap_warn">Important</a>**:

- the previous version of the article contained the "standard" encrypted swap partition setup by using the "_/dev/urandom_" for encryption
- this doesn't work properly with ZFS root, because it created a module load cycle during the bootup, resulting in some ZFS pools not being mounted randomly  
(depending where the cycle is broken by the boot loader)
- in the principle this is caused by the cryptsetup needing the root partition (for "_/dev/urandom_") but the ZFS target to mount the root partition requiring cryptsetup, creating the cyclic dependency
{% endcapture %}

{% include notice level="danger" %}

**g) Create the "boot" ZFS pool**:

```bash
# prepare the temporary target directory
mkdir -p /target

# create the "boot" pool
# - the mirror will be attached later
zpool create -f \
    -o ashift=12 \
    -d \
    -o feature@async_destroy=enabled \
    -o feature@bookmarks=enabled \
    -o feature@embedded_data=enabled \
    -o feature@empty_bpobj=enabled \
    -o feature@enabled_txg=enabled \
    -o feature@extensible_dataset=enabled \
    -o feature@filesystem_limits=enabled \
    -o feature@hole_birth=enabled \
    -o feature@large_blocks=enabled \
    -o feature@lz4_compress=enabled \
    -o feature@spacemap_histogram=enabled \
    -o feature@zpool_checkpoint=enabled \
    -O acltype=posixacl \
    -O compression=lz4 \
    -O normalization=formD \
    -O atime=off \
    -O relatime=on \
    -O xattr=sa \
    -O canmount=off \
    -O devices=off \
    -O mountpoint=/boot \
    -R /target \
    bpool \
    ${DISK[1]}-part2

# test the pool status
zpool status bpool
```

{% capture notice_contents %}
**<a name="bpool_note">Notes</a>**:

- the boot pool is created with limited set of ZFS features (only those that the Grub bootloader can understand) - this is done by disabling all features first via the `"-d"` option and then enabling just the Grub-supported features selectively
- the `"ashift=12"` parameter particularly important: it enforces the 4k sector size alignment - even though you might not have a 4k sector disk now, you might in the future when doing a disk replacement, therefore it is recommended to use this parameter
- the `"atime=off"` and `"relatime=on"` speed up the disk operations, at the expense of not storing the last access time
{% endcapture %}

{% include notice level="info" %}

**h) Create the "root" ZFS pool** (on top of the LUKS encrypted partition):

```bash
# create the "root" pool
# - the mirror will be attached later
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
    -R /target \
    rpool \
    ${DISK[1]}-part3

# for SSD: Set the auto-trim
zpool set autotrim=on rpool

# test the pool status
zpool status rpool
```

{% capture notice_contents %}
**<a name="rpool_note">Notes</a>**:

- we use all available features for the root pool
- the `"ashift=12"` parameter is used again to enforce the 4k sector size alignment
- the `"atime=off"` and `"relatime=on"` used to speed up the disk operations
- when installing on SSD, the auto-trim needs to be used so that the TRIM works properly  
{% endcapture %}

{% include notice level="info" %}

**i) Create the ZFS datasets** (this is Ubuntu specific, see the notes):

```bash
# create the bpool and rpool main datasets
zfs create \
    -o canmount=off \
    -o mountpoint=none \
    bpool/BOOT

zfs create \
    -o canmount=off \
    -o mountpoint=none \
    rpool/ROOT

# create the Ubuntu-specific boot and root datasets
zfs create \
    -o canmount=noauto \
    -o mountpoint=/boot \
    bpool/BOOT/ubuntu

# root encrypted via the "root" key
zfs create \
    -o canmount=noauto \
    -o mountpoint=/ \
    -o com.ubuntu.zsys:bootfs=yes \
    -o com.ubuntu.zsys:last-used=$(date +%s) \
    -o encryption=aes-256-gcm \
    -o keyformat=hex \
    -o keylocation=file:///etc/crypt/zfs/init/root.key \
    rpool/ROOT/ubuntu

# create the main user dataset
# (encrypted via the "home" key)
zfs create \
    -o canmount=off \
    -o mountpoint=none \
    -o setuid=off \
    -o encryption=aes-256-gcm \
    -o keyformat=hex \
    -o keylocation=file:///etc/crypt/zfs/home.key \
    rpool/USER

# mount the main data sets
# (the order is important!)
zfs mount rpool/ROOT/ubuntu
zfs mount bpool/BOOT/ubuntu

# list the datasets to validate
zfs list

# for mirror/raid topology
zfs create bpool/BOOT/ubuntu/grub

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
chmod 1777 /target/var/tmp

zfs create -o com.ubuntu.zsys:bootfs=no rpool/ROOT/ubuntu/tmp
chmod 1777 /target/tmp

# create the user directories
zfs create \
    -o com.ubuntu.zsys:bootfs-datasets=rpool/ROOT/ubuntu \
    -o canmount=on \
    -o mountpoint=/root \
    rpool/USER/root

TARGET_USER=<admin-user>

zfs create \
    -o com.ubuntu.zsys:bootfs-datasets=rpool/ROOT/ubuntu \
    -o canmount=on \
    -o mountpoint=/home/$TARGET_USER \
    rpool/USER/$TARGET_USER

# eventually repeat for each existing user
# (check the original system for the user names and ids)

# list the datasets to validate again
zfs list

# list the folders to check properly mounted
ls -al /target/
ls -al /target/boot/
ls -al /target/home/

# check the encryption:
#   1. Expect root encrypted
zfs get encryption /target/
#   2. Expect home/$TARGET_USER encrypted
zfs get encryption /target/home/$TARGET_USER
#   3. Expect boot not encrypted
zfs get encryption /target/boot
```

{% capture notice_contents %}
**<a name="datasets_note">Notes</a>**:

- note that the original guides use additional UUID for the datasets, likely to be able to install multiple different versions, eventually a dual-boot; however I was installing only one system here, therefore I wasn't using the UUID part
- the server dataset layout is following the Ubuntu recommendations [^7]
- the main user dataset ("rpool/USER") could eventually also be on a separate zpool, if expecting larger USER dirs and putting them e.g. on separate data disks; for my project I don't expect to have much of user data so they are kept on the root pool
- we are creating separate dataset per user, which is the recommended setup [^8] (this allows to have per-user snapshots and also to remove the user dataset completely including the snapshots when the user is to be removed)
{% endcapture %}

{% include notice level="info" %}

### 4.2. Copy the existing installation back from the backup

**a) Mount the backup volume**:

```bash
# create the mount point
mkdir -p /mnt/backup

# mount the backup volume
mount ${DISK[2]}-part1 /mnt/backup
```

**b) Copy the installation onto the ZFS**:

- using the `"rsync"` to copy all including the permissions, attributes etc.)

```bash
# copy the installation root
rsync -avhPXI --info=progress2 --exclude=/swapfile /mnt/backup/. /target/

# list the folders to check the data are copied properly
ls -al /target/
ls -al /target/boot/
ls -al /target/home/
```

### 4.3. Preparing the boot partition environment

**a) Copy the encryption keys to the root partition**:

```bash
# copy the encryption keys
cp -r /etc/crypt /target/etc/
# make it only readable for the root user
chmod -R go-rwx /target/etc/crypt
```

**b) Setup the fstab and encryption** [^9]:

```bash
# UEFI Bios only: add the new EFI partition to the fstab
echo "UUID=$(blkid -s UUID -o value ${DISK[1]}-part1) /boot/efi vfat noauto,umask=0077,fmask=0077,dmask=0077 0 1" \
    >> /target/etc/fstab

# setup some basic protection of the encryption keys
echo "UMASK=0077" \
    >> /target/etc/initramfs-tools/initramfs.conf

# disable the resume (hibernation) for now
echo "RESUME=none" > /target/etc/initramfs-tools/conf.d/resume
```

- <a name="keys_initramfs"></a>create the initrd script to copy the init level key files:

(note this step should be skipped if the remote key loading is being used - the keys shouldn't go into the initramfs then)

```bash
  cat <<EOF > /target/etc/initramfs-tools/hooks/zfs-crypt
#!/bin/sh

PREREQS=""

case \$1 in
    prereqs) echo "\${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

mkdir -p \${DESTDIR}/etc/crypt/zfs || true
cp -ar /etc/crypt/zfs/init \${DESTDIR}/etc/crypt/zfs/
chmod -R go-rwx \${DESTDIR}/etc/crypt
EOF


# set the permissions
chmod +x /target/etc/initramfs-tools/hooks/zfs-crypt
```

**c) Setup the fstab for the ZFS migration**:

```bash
vim /target/etc/fstab
```

- remove the original volumes mountpoins for root etc.
- keep just the the EFI partition  
(you can just comment out the existing entries by using the `"#"` character)
- the swap file will not be used for now, even though we allocated the space  
(the setup will be done in the next part)
- the ZFS mountpoints are not needed because of being mounted automatically (via zsys)

**d) Update the Grub boot loader configuration**:

```bash
vim /target/etc/default/grub

# add the line
GRUB_RECORDFAIL_TIMEOUT=5

# add to the GRUB_CMDLINE_LINUX_DEFAULT options
# (to allow start from a degraded array, should a failure happen)
GRUB_CMDLINE_LINUX_DEFAULT="bootdegraded=true init_on_alloc=0"

# disable the OS prober by adding:
GRUB_DISABLE_OS_PROBER=true
```

**e) Chroot to the target system to complete the setup**:

```bash
# prepare to enter the target system chroot
mount --rbind /dev     /target/dev
mount --rbind /dev/pts /target/dev/pts
mount --rbind /proc    /target/proc
mount --rbind /sys     /target/sys
mount --rbind /run     /target/run

# enter the chroot
chroot /target /usr/bin/env \
    DISK=$DISK \
    bash --login

# UEFI Bios only: mount the EFI partition to set it up
mount /boot/efi
# check it mounted properly
mount | grep efi
```

**f) Update the Grub configuration and the boot image**:

```bash
# test if the Grub understands the boot filesystem
grub-probe /boot
# - should print "zfs"

# update the Grub configuration
update-grub

# re-create the initramfs image
update-initramfs -c -k all
# ignore the eventual cryptsetup warnings - should still work

# check if the cryptkeys are present in the initramfs
lsinitramfs -l /boot/initrd.img-<tab-complete-the-img-path> | less
# check for:
# - the "etc/crypt/init/root.key" file
```

**g) Install the GRUB boot loader**:

- for UEFI Bios:

```bash
# install the EFI
grub-install --target=x86_64-efi --efi-directory=/boot/efi \
    --bootloader-id=ubuntu --recheck --no-floppy

# check the grub config has been created on the EFI
less /boot/efi/EFI/ubuntu/grub.cfg

# check the EFI entries
efibootmgr -v

# eventually add a nicer Ubuntu entry
efibootmgr --create --disk ${DISK[1]}-part1 --loader '\EFI\UBUNTU\SHIMX64.EFI' --label "Ubuntu Linux"

# eventually delete the other unwanted entries
efibootmgr -Bb nnnn
```

- non-UEFI Bios:

```bash
# install the grub
grub-install --target=i386-pc ${DISK[1]}
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
mkdir /etc/zfs/zfs-list.cache
touch /etc/zfs/zfs-list.cache/bpool
touch /etc/zfs/zfs-list.cache/rpool

# create the cacher link
ln -s /usr/lib/zfs-linux/zed.d/history_event-zfs-list-cacher.sh /etc/zfs/zed.d

# create the cache files
zed -F &

cat /etc/zfs/zfs-list.cache/bpool
cat /etc/zfs/zfs-list.cache/rpool

# if either is empty, force a cache update and check again:
zfs set canmount=noauto bpool/BOOT/ubuntu
zfs set canmount=noauto rpool/ROOT/ubuntu

fg
# press Ctrl-C

# update the mountpoints
sed -Ei "s|/target/?|/|" /etc/zfs/zfs-list.cache/*

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

## 5. System recovery

This section describes the steps for eventual recovery of the system, should something go wrong.

Note that if the disk is encrypted with LUKS, you'll need the encryption key (that you hopefully downloaded to a secure location beforehand).

**a) Boot up the Live Desktop**:

- use the steps described in the "[Booting up the Live Desktop](#33-booting-up-the-live-desktop)" section  
(including starting the "*ssh-server*")
- connect remotely so that you can copy the commands easily

**b) Enter the administrative console**:

- this is for convenience to not have to type `"sudo"` in front of every command

```bash
sudo -i
```

**c) Install the needed packages**:

- use "*vim*" or any other editor you prefer
- note this installs the packages to the currently running Live DVD, not to the migrated installation

```bash
apt install -y zfs-initramfs vim

# stop the ZFS manager for now
systemctl stop zed
```

**d) Set the variables for the disk being used**:
- **DISK[1]**: The primary drive
- **DISK[2]**: The secondary drive
- use your disk ids as appropriate

```bash
# prefer to use the disk-by-id to avoid confusion with the disk assignment
ls -al /dev/disk/by-id/

# set the disk ids appropriately
# (use the physical disk links without the "-partN")

# the primary drive
DISK[1]=/dev/disk/by-id/<tab-complete-the-disk-path>
# the secondary drive
DISK[2]=/dev/disk/by-id/<tab-complete-the-disk-path>
```

**e) Decrypt the root partition**:

- copy the key back to the Live CD environment e.g. via [WinSCP](https://winscp.net/eng/download.php) on Windows, the "scp" command on Linux or the clipboard by `"mkdir -p /etc/crypt/zfs/init && vim /etc/crypt/zfs/init/root.key"`

**f) Mound the ZFS volumes**:

```bash
# create the target dir
mkdir /target

# import the pools
zpool export -a
zpool import -f -N -R /target rpool
zpool import -f -N -R /target bpool

# import the root partition key
# (will load from the current live CD "/etc/crypt/zfs/init/root.key")
zfs load-key rpool/ROOT/ubuntu
# mount the root
zfs mount -a

# check if mounted properly
ls -al /target/boot/grub
ls -al /target/home
```

**g) Chroot to the target system to repair**:

```bash
# prepare to enter the target system chroot
mount --rbind /dev     /target/dev
mount --rbind /dev/pts /target/dev/pts
mount --rbind /proc    /target/proc
mount --rbind /sys     /target/sys
mount --rbind /run     /target/run

# enter the chroot
chroot /target /usr/bin/env \
    DISK=$DISK \
    bash --login

# import the home partition key
# (will load from the target root "/etc/crypt/zfs/home.key")
zfs load-key rpool/USER
# mount the home
zfs mount -a

# validate the USER datasets mounted
mount | grep USER

# mount all additional disks from /etc/fstab
mount -a
```

**h) Finalize when done**:

```bash
# exit the chroot
exit

# unmount and export the zpool
mount | grep -v zfs | tac | awk '/\/target/ {print $3}' | \
    xargs -i{} umount -lf {}

zpool export -a

# reboot and test
reboot
```

## 6. Next steps

In the next part we will complete the system disk setup (mirroring of the primary drive) and setup the data drives.

[Next: RAID and data drives]({% post_url 2020-08-30-project-ares-part-5-ubuntu-on-zfs-raid %}){: .btn .btn--info}
{: .align-right}

## Resources and references

[^1]: [OpenZFS: Ubuntu 20.04 Root on ZFS](https://openzfs.github.io/openzfs-docs/Getting%20Started/Ubuntu/Ubuntu%2020.04%20Root%20on%20ZFS.html)
[^2]: [Installing Ubuntu on a ZFS root, with encryption and mirroring](https://saveriomiroddi.github.io/Installing-Ubuntu-on-a-ZFS-root-with-encryption-and-mirroring/)
[^3]: [Encrypted dataset still slow and high load with ZoL 0.8.3](https://www.reddit.com/r/zfs/comments/f3w6v0/encrypted_dataset_still_slow_and_high_load_with/)
[^4]: [ZFS focus on Ubuntu 20.04 LTS: what’s new?](https://didrocks.fr/2020/05/21/zfs-focus-on-ubuntu-20.04-lts-whats-new/)
[^5]: [Ubuntu: SwapFaq](https://help.ubuntu.com/community/SwapFaq)
[^6]: [Ubuntu: Hibernate with encrypted swap](https://help.ubuntu.com/community/EnableHibernateWithEncryptedSwap)
[^7]: [ZFS focus on Ubuntu 20.04 LTS: ZSys dataset layout](https://didrocks.fr/2020/06/16/zfs-focus-on-ubuntu-20.04-lts-zsys-dataset-layout/)
[^8]: [ZFS NFS home directories: Each user as a separate ZFS dataset?](https://www.reddit.com/r/zfs/comments/bgsr0l/zfs_nfs_home_directories_each_user_as_a_separate/)
[^9]: [Ubuntu: Full disk encryption howto](https://help.ubuntu.com/community/Full_Disk_Encryption_Howto_2019)

{% include abbrev domain="computers" %}
