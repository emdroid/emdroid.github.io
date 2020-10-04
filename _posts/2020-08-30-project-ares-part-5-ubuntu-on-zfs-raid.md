---
title: "Project Ares: Part V - RAID and data drives"
header:
  teaser: /assets/images/posts/nas-ares/raid-5.png
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
  - truecrypt
---

![NAS OS](/assets/images/posts/nas-ares/raid-5.png){: .align-center .img-large}

Sharing the experience of building a home NAS/VM server.

The part 5 completes the system installation with setting up the system drives RAID-1 and the data drives in RAID-5 configuration over encrypted drives.

<!-- more -->

## 1. Introduction

In the [previous part]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}) the Ubuntu system was installed on top of ZFS file system.

Now we will complete the system disks setup for RAID-1 redundancy configuration and the data drives with RAID-5 setup.

{% capture notice_contents %}
**<a name="zfs-native-info">Important changes</a>**

Contrary to the initial intent described in the [Part 3 (OS and filesystem)]({% post_url 2020-07-18-project-ares-part-3-os-and-file-system-considerations %}#52-the-data-disks), I ended up using the native ZFS encryption and RAID.

**The original version of the article is still available here:  
[Project Ares: Part V - RAID and data drives (original version)]({% post_url 2020-08-30-project-ares-part-5-ubuntu-on-zfs-raid-initial %})**

See the note in the [previous article]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}#zfs-native-info) for native ZFS encryption details.

With the full knowledge about the ZFS native RAID limitations, I still decided to also use the native RAID-5 for the data disks because of the **MD-RAID being way too slow with my 4 x 8 TB disk setup**:
- when I tried to create the MD RAID array over the disk setup, the initial array synchronization was at approx. 4 % after 1 hour
- that would mean the entire array sync would take maybe around 25 hours (= 1 day) of array rebuild time
- however this was when idle, in the real scenario the disks might still be used while the degraded array is eventually being rebuilt, which increases the rebuild time  
(depending on the usage pattern the increase might be 2x - 10x more)
- and if another disk (stressed by the rebuilding) fails in this period, all the data are lost
- note in particular, that the MD-RAID needs to synchronize the whole disks (even if empty) because of working on the block level thus doesn't have the information which data are in use

**The ZFS native RAID has a huge advantage in this respect**:
- it knows which data are used (synchronizes on the file system level) so it only needs to write the minimal necessary data
- therefore it doesn't even need to do any significant initial synchronization when the RAID is created (as there is no data yet, except for the basic ZFS metadata)
- thus in general the array rebuild ("resilvering") is significantly faster compared to the the block-level MD-RAID, especially when the disks are not completely full

**The main disadvantage of using the native ZFS RAID** is the inability to add an additional disk to the array (cannot add disks, needs to create a new RAID VDEV group instead) [^1].
However that actually doesn't seem to pose a too big issue for my setup right now, as the 4 x 8 TB disks (= 24 TB in RAID-5) should offer enough disk space for at least the next few years.

And there actually seems to be some **significant effort to offer the possibility of expanding the ZFS raid-z arrays** [^7] - although it seems to be somewhat stale, but there already is the Alpha version out.
So it isn't too far stretched to expect the feature being already available at the time I might eventually need to extend the array anyway.
{% endcapture %}

{% include notice level="warning" %}

## 2. Setting up the system disk RAID-1

In this section we'll setup the mirroring (RAID-1) of the system disk.

This can be eventually skipped if you only want a single primary drive without mirroring.

The advantage of setting this up is that the system will boot up even in case the primary drive fails (from the secondary drive mirror). The disadvantage is that you'll need to spend another drive at least the same or bigger size of the primary drive (if bigger, can use the rest for some additional data partition that is not being backed up).

What will be done here:
- copying the EFI/GPT partition
- adding mirrors of the boot and root ZFS volumes (with the root still encrypted)
- adding mirror of the swap partition for hibernation setup (optional)

### 2.1. Preparing the environment

**a) Connect remotely over SSH:**

- as the SSH server should be running, you should be able to connect remotely via the SSH client (eg. PuTTY on Windows)

- use the hostname, username and password you've set up when installing the system

**b) Enter the administrative console**:

- this is for convenience to not have to type `"sudo"` in front of every command

```bash
sudo -i
```

**c) Set the variables for the disks being used**:
- **DISK[1]**: The primary drive
- **DISK[2]**: The secondary drive
- use your disk ids as appropriate

```bash
# prefer to use the disk-by-path to avoid confusion with the disk assignment
ls -al /dev/disk/by-path/

# set the disk ids appropriately
# (use the physical disk links without the "-partN")

# the primary drive
DISK[1]=/dev/disk/by-path/<tab-complete-the-disk-path>
# the secondary drive
DISK[2]=/dev/disk/by-path/<tab-complete-the-disk-path>
```

### 2.2. Partitioning the secondary drive

- here we should create the exact same partitions as on the primary drive

- using similar commands that were used for the primary drive (change to your setup if necessary)

```bash
# list the existing partitions on the primary disk
sgdisk -p ${DISK[2]}
# !!! double check it is the correct drive you want to use !!!
# (otherwise change the DISK[2] variable appropriately)

# remove any existing partitions
# (note that if there are any LVM volumes currently active,
#  they should be removed first)
wipefs --all ${DISK[2]}

# clone the partitions of the primary drive
sgdisk -R ${DISK[2]} ${DISK[1]}

# set a new random disk GUID of the secondary drive
sgdisk -G ${DISK[2]}

# check by listing the partitions
sgdisk -p ${DISK[2]}
# compare with the primary drive
sgdisk -p ${DISK[1]}
```

### 2.3. Copying the EFI / GPT partition and install the boot loader

- note that the EFI / GTP partition will not be mirrored, just copied over
- it is eventually possible to mirror the EFI partition, however it is tedious to setup and might still have compatibility issues
- in addition the EFI partition rarely changes (once the system is installed it will never change again, at least not for Ubuntu - in CentOS the situation might be different because it contains the GRUB menu there which might eventually change more frequently)
- so we'll save the hassle and just copy the partition over for now

**a) Copy the EFI / GPT partition**:

```bash
dd if=${DISK[1]}-part1 of=${DISK[2]}-part1 bs=1M
```

**b) Install the GRUB boot loader**:

- for UEFI Bios:

```bash
# mount the secondary EFI
mkdir -p /boot/efi2
mount ${DISK[2]}-part1 /boot/efi2

# install the EFI
grub-install --target=x86_64-efi --efi-directory=/boot/efi2 \
    --bootloader-id=ubuntu --recheck --no-floppy

# check the grub config has been created on the EFI
less /boot/efi2/EFI/ubuntu/grub.cfg

# check the EFI entries
efibootmgr -v

# eventually add a nicer Ubuntu entry
efibootmgr --create --disk ${DISK[2]}-part1 --loader '\EFI\UBUNTU\SHIMX64.EFI' --label "Ubuntu Linux (Backup)"

# eventually delete the other unwanted entries
# !!! make sure to keep the primary and secondary entry !!!
efibootmgr -Bb nnnn
```

- non-UEFI Bios:

```bash
# install the grub
grub-install --target=i386-pc ${DISK[2]}
```

### 2.4. Adding the Boot pool mirror

```bash
# check the current pool status
zpool status bpool

# attach the secondary disk partition
zpool attach -f bpool ${DISK[1]}-part2 ${DISK[2]}-part2

# check the pool status to verify
# - should see both attached
zpool status bpool
```

- you should see output similar to (the following is from a VM testing machine):

```
pool: bpool
 state: ONLINE
  scan: resilvered 277M in 0 days 00:00:01 with 0 errors on Thu Sep 17 23:27:31 2020
config:

        NAME                                     STATE     READ WRITE CKSUM
        bpool                                    ONLINE       0     0     0
          mirror-0                               ONLINE       0     0     0
            pci-0000:00:10.0-scsi-0:0:0:0-part2  ONLINE       0     0     0
            pci-0000:00:10.0-scsi-0:0:1:0-part2  ONLINE       0     0     0
```

### 2.5. Adding the Root pool mirror

```bash
# check the current pool status
zpool status rpool

# attach the secondary disk partition
zpool attach -f rpool ${DISK[1]}-part3 ${DISK[2]}-part3

# check the pool status to verify
# - should see both attached
zpool status rpool
```

- the output will be like:

```
zpool status rpool
  pool: rpool
 state: ONLINE
  scan: resilvered 2.15G in 0 days 00:00:14 with 0 errors on Thu Sep 17 23:29:49 2020
config:

        NAME                                     STATE     READ WRITE CKSUM
        rpool                                    ONLINE       0     0     0
          mirror-0                               ONLINE       0     0     0
            pci-0000:00:10.0-scsi-0:0:0:0-part3  ONLINE       0     0     0
            pci-0000:00:10.0-scsi-0:0:1:0-part3  ONLINE       0     0     0
```

- note in particular, that the RAID only had to synchronize 2.15 GB which it has been able to do really fast  
(MD-RAID would have to sync the whole partition, which is about 450 GB large)

### 2.6. Setting up the SWAP file

- the swap file also needs to be mirrored if we want to be able to run from either of the drives (otherwise the swap file would not be found and the boot loading will fail)
- this in particular is necessary if the hibernation is used
- if no hibernation is used, it might also be possible the ignore the swap partition mount error by using the "nofail" fstab/crypttab flags
- we will use mdadm as the mirroring option here
- we will re-do the complete swap setup now

{% capture notice_contents %}
**<a name="swap_notes">Warning</a>**:

- some early guides recommended putting the swap onto ZFS (ZVOL), however that is currently not recommended as it might create eventual deadlocks [^6]
- for non-ZFS setup the Ubuntu recommends using a swap file instead of the swap partition now, but that is not recommended either when using the ZFS (for the same reason as above)
{% endcapture %}

{% include notice level="warning" %}

**a) The swap partition and RAID setup**:

```bash
# unmount the current swap
swapoff -a

# create the swap MD array
mdadm --create /dev/md0 -l 1 -n 2 -e 1.2 ${DISK[1]}-part4 ${DISK[2]}-part4
# check the array status
mdadm --detail /dev/md0
```

**b) The swap encryption setup**:

{% capture notice_contents %}
**<a name="swap_warn">Important</a>**:

- the "standard" non-hibernation encrypted swap setup is usually done via the "_/dev/urandom_" for generating the one-time encryption key
- this doesn't work properly with ZFS root (even without hibernation!), because it creates a module load cycle during the bootup, resulting in some ZFS pools not being mounted randomly  
(depending where the cycle is broken by the boot loader)
- in the principle this is caused by the cryptsetup needing the root partition (for "_/dev/urandom_") but the ZFS target to mount the root partition requiring cryptsetup, creating the cyclic dependency
- therefore we use an **explicit generated encryption key** that is used for the swap encryption
{% endcapture %}

{% include notice level="danger" %}

- for the hibernation to work the swap encryption key is added to the initramfs
- if you don't plan to use the hibernation, the encryption key can be loaded from the root directly (without putting it to the initramfs)

```bash
# create the swap partition LUKS key
# - using the Base64 encoding to only have printable chars in the key
dd if=/dev/urandom bs=512 skip=4 count=4 iflag=fullblock | base64 -w 0 \
    > /etc/crypt/init/swap.key
chmod go-rwx /etc/crypt/init/swap.key

# setup the swap partition encryption
cryptsetup luksFormat -q -c aes-xts-plain64 -s 512 -h sha256 \
    -d /etc/crypt/init/swap.key /dev/md0
cryptsetup luksOpen -d /etc/crypt/init/swap.key /dev/md0 swap

# make the swap filesystem
mkswap /dev/mapper/swap

# add the crypttab entry
# (using "discard" to allow SSD TRIM commands)
echo "swap /dev/md0 /etc/crypt/init/swap.key luks,discard" \
    >> /etc/crypttab

# add the key pattern to be included in the initramfs
echo 'KEYFILE_PATTERN="/etc/crypt/init/*.key"' \
    >> /etc/cryptsetup-initramfs/conf-hook
# check it has been added properly
less /etc/cryptsetup-initramfs/conf-hook
```

**c) Finalizing the swap partition setup**:

```bash
# add the encrypted swap entry to the fstab
echo "/dev/mapper/swap none swap sw,discard 0 0" \
    >> /etc/fstab

# remove the original swap entry in crypttab if any
vim /etc/crypttab
vim /etc/fstab

# re-enable the swap
swapon -a

# check the status
swapon --summary
```

**d) Enabling the hibernation**:

- this is optional - the hibernation can be enabled so that the system can wake up where it was stopped

```bash
# add the resume option
echo "RESUME=/dev/mapper/swap" > /etc/initramfs-tools/conf.d/resume
```

### 2.7. Fixing the degraded array bootup

There are currently some bugs in Ubuntu which prevent booting from degraded arrays.
To fix, the following initramfs file needs to be created ([see here for details](https://feeding.cloud.geek.nz/posts/installing-ubuntu-bionic-on-encrypted-raid1/)):

```bash
cat <<EOF > /etc/initramfs-tools/scripts/local-top/md-boot
#!/bin/sh
PREREQ="mdadm"
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

mdadm --run /dev/md0
EOF

chmod +x /etc/initramfs-tools/scripts/local-top/md-boot
```

### 2.8. Updating the initram to contain all LUKS key entries

```bash
# re-create the initramfs image
update-initramfs -c -k all
# ignore the eventual cryptsetup warnings - should still work

# check if the cryptkeys still present in the initramfs
lsinitramfs -l /boot/initrd.img-<tab-complete-the-img-path> | less
# check for:
# - the "cryptroot/crypttab" size
# - the "etc/crypt/init/root.key" file
# - the "etc/crypt/init/swap.key" file

# reboot and test
reboot
```

## 3. Testing the system disk setup

### 3.1. Testing the hibernation

- checking the hibernation support:

```bash
# install the power management utils
apt install -y pm-utils

# check the hibernation / suspend support:
# pm-is-supported [{--suspend | --hibernate | --suspend-hybrid}]
for state in suspend hibernate suspend-hybrid ; do
    pm-is-supported --$state && echo "$state: supported" || echo "$state: not supported"
done
```

- performing the hibernation:

```bash
# test the hibernation
systemctl hibernate
# or
pm-hibernate

# start the PC again after the hibernation
# and check the state restored
```

In particular, the "_suspend-hybrid_" state is especially interesting:
- it allows enter the sleep mode but the state is saved to the disk (hibernated) at the same time
- that means the wake up is fast in the usual case (waking up just from sleep mode)
- but if the power is lost, the state can still be restored from disk

The following table shows the differences between the power saving modes:

| Mode      | To RAM | To HDD | Suspend   | Restore   | Consumption | Survives power down |
| :-------- | :----: | :----: | :-------: | :-------: | :---------: | :-----------------: |
| Suspend   |   Yes  |   No   | Very fast | Very fast |     Low     |         No          |
| Hibernate |   No   |   Yes  |   Fast    |   Fast    |   Very low  |         Yes         |
| Hybrid    |   Yes  |   Yes  |   Fast    | Very fast |     Low     |         Yes         |

### 3.2. Testing the redundancy

Optionally (but strongly recommended) you can test if the system boots up in case one if the mirrored system drives fails:
1. Shut down the machine (e.g. by using `"init 0"`)
2. **Physically disconnect one of the drives**
3. Try to turn the machine on, to see if it boots up without any issues
4. **Connect the drive back and boot again**
5. Repeat for the other drive

Check the status of the ZFS pools and MD-RAID, you should see results like:

```bash
zpool status
```

```
  pool: bpool
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
        invalid.  Sufficient replicas exist for the pool to continue
        functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: http://zfsonlinux.org/msg/ZFS-8000-4J
  scan: resilvered 216K in 0 days 00:00:01 with 0 errors on Sat Sep 19 00:50:08 2020
config:

        NAME                                     STATE     READ WRITE CKSUM
        bpool                                    DEGRADED     0     0     0
          mirror-0                               DEGRADED     0     0     0
            pci-0000:00:10.0-scsi-0:0:0:0-part2  ONLINE       0     0     0
            2663801463946793175                  UNAVAIL      0     0     0  was /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:1:0-part2

errors: No known data errors

  pool: rpool
 state: DEGRADED
status: One or more devices could not be used because the label is missing or
        invalid.  Sufficient replicas exist for the pool to continue
        functioning in a degraded state.
action: Replace the device using 'zpool replace'.
   see: http://zfsonlinux.org/msg/ZFS-8000-4J
  scan: resilvered 8.92M in 0 days 00:00:00 with 0 errors on Sat Sep 19 00:50:05 2020
config:

        NAME                                     STATE     READ WRITE CKSUM
        rpool                                    DEGRADED     0     0     0
          mirror-0                               DEGRADED     0     0     0
            pci-0000:00:10.0-scsi-0:0:0:0-part3  ONLINE       0     0     0
            11116708511012010870                 UNAVAIL      0     0     0  was /dev/disk/by-path/pci-0000:00:10.0-scsi-0:0:1:0-part3

errors: No known data errors
```

```bash
mdadm --detail /dev/md0
```

```
/dev/md0:
           Version : 1.2
     Creation Time : Sun Aug 30 17:41:16 2020
        Raid Level : raid1
        Array Size : 8383424 (8.00 GiB 8.58 GB)
     Used Dev Size : 8383424 (8.00 GiB 8.58 GB)
      Raid Devices : 2
     Total Devices : 1
       Persistence : Superblock is persistent

       Update Time : Fri Sep 18 22:41:03 2020
             State : clean, degraded
    Active Devices : 1
   Working Devices : 1
    Failed Devices : 0
     Spare Devices : 0

Consistency Policy : resync

              Name : ubuntu-test:0  (local to host ubuntu-test)
              UUID : e7eec08c:dd48273c:d753a245:07b201ea
            Events : 111

    Number   Major   Minor   RaidDevice State
       0       8        4        0      active sync   /dev/sda4
       -       0        0        1      removed
```

- after done, clear the ZFS pool error status by:

```bash
zpool clear bpool
zpool clear rpool
```

## 4. Setting up the RAID-5 data disks

This section explains the RAID-5 data disks setup.

### 4.1. Preparing the environment

- **DISK[1]** - **DISK[4]**: The disks to be used for the array
- use your disk ids as appropriate

```bash
# prefer to use the disk-by-path to avoid confusion with the disk assignment
ls -al /dev/disk/by-path/

# set the disk ids appropriately
# (use the physical disk links without the "-partN" if any)

DISK[1]=/dev/disk/by-path/<tab-complete-the-disk-path>
DISK[2]=/dev/disk/by-path/<tab-complete-the-disk-path>
DISK[3]=/dev/disk/by-path/<tab-complete-the-disk-path>
DISK[4]=/dev/disk/by-path/<tab-complete-the-disk-path>
```

### 4.2. Cleaning the disks

- we will create a single partition covering the entire disk for each of the RAID disks
- the disk partition will then be used to create the array

```bash
# show the current partitions on each disk
# !!! double-check that they are the correct ones !!!
# (will destroy all existing data)
# - check the IDs, disk sizes, existing partitions etc.
sgdisk -p ${DISK[1]}
sgdisk -p ${DISK[2]}
sgdisk -p ${DISK[3]}
sgdisk -p ${DISK[4]}

# clear all existing partitions
# (if having LVM volumes there, those need to be cleared first)
for n in $(seq 1 4); do wipefs --all ${DISK[$n]} ; done
```

### 4.3. Creating the ZFS pool

```bash
# create the "data" pool
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
    -O mountpoint=none \
    dpool \
    raidz1 \
    ${DISK[1]} \
    ${DISK[2]} \
    ${DISK[3]} \
    ${DISK[4]}

# for SSD: Set the auto-trim
zpool set autotrim=on dpool

# test the pool status
zpool status dpool
```

{% capture notice_contents %}
**<a name="data_pool_note">Notes</a>**:

- the array is created over the whole physical disks, which has some advantates for the ZFS maintenance
(in particular, the ZFS subsystem can manage the disk itself, reportedly switching off the "standard" IO scheduler as it provides its own) [^8]
- the ZFS subsystem still creates partitions over the disks, while leaving some small space at the end to accommodate for eventual slight disk size differences
- the pool itself is not encrypted, the encryption will be done on the dataset level [^9]
{% endcapture %}

{% include notice level="info" %}

### 4.4. Creating the encryption key

- the key is stored on the root partition (which itself is encrypted)
- the maximum key size for the native ZFS encryption is 32 bytes

```bash
# generate the encryption key
# (stored on the encrypted root partition)
dd if=/dev/urandom bs=32 skip=4 count=1 iflag=fullblock | hexdump -ve '/1 "%02x"' \
    > /etc/crypt/zfs/data.key
chmod go-rwx /etc/crypt/zfs/data.key
```

### 4.5. Creating the ZFS data sets

- now we can create some datasets
- in particular, to organize stuff keep in mind that each separate dataset can create its own COW snapshots
- note that you can eventually use different keys for separate encrypted datasets

```bash
# create the main dataset
zfs create \
    -o canmount=off \
    -o mountpoint=none \
    dpool/DATA

# create the main mount data set
# (setting up the encryption)
zfs create \
    -o canmount=noauto \
    -o mountpoint=/data \
    -o encryption=aes-256-gcm \
    -o keyformat=hex \
    -o keylocation=file:///etc/crypt/zfs/data.key \
    dpool/DATA/data

# create media data sets
zfs create \
    -o com.ubuntu.zsys:bootfs=no \
    dpool/DATA/data/media

zfs create dpool/DATA/data/media/movies
zfs create dpool/DATA/data/media/pictures
zfs create dpool/DATA/data/media/music

# etc. - create data sets as needed
# ...

# eventually setup some permissions
chown root:sambashare /data/media/*
chmod -R g+w /data/media/*
# etc.

# check with:
ls -al /data/media
```

- note that all the data sets share the total space, so there is no need for thin provisioning  
(it is eventually also possible to setup quota for particular data sets)

### 4.6. Load the encryption key on boot

- script to connect "regular" encrypted files:

```bash
  cat <<EOF > /etc/systemd/system/zfs-load-key.service

[Unit]
Description=Load encryption keys
DefaultDependencies=no
After=zfs-import.target
Before=zfs-mount.service

[Service]
Type=oneshot
RemainAfterExit=yes
ExecStart=zfs load-key -a

[Install]
WantedBy=zfs-mount.service
EOF


# enable the service
systemctl enable zfs-load-key
```

## 5. Next steps

In the next part we'll setup the reporting (SMART etc.) and complete the basic system setup.

## Resources and references

[^1]: [The 'Hidden' Cost of Using ZFS for Your Home NAS](https://louwrentius.com/the-hidden-cost-of-using-zfs-for-your-home-nas.html)
[^2]: [Encrypted dataset still slow and high load with ZoL 0.8.3](https://www.reddit.com/r/zfs/comments/f3w6v0/encrypted_dataset_still_slow_and_high_load_with/)
[^3]: [Serverfault: Debian server has degraded mdadm array on every boot](https://serverfault.com/questions/722360/debian-server-has-degraded-mdadm-array-on-every-boot)
[^4]: [StackExchange: What's the difference between creating mdadm array using partitions or the whole disks directly](https://unix.stackexchange.com/a/323425)
[^5]: [StackExchange: ZFS stripe on top of hardware RAID 6. What could possibly go wrong?](https://serverfault.com/a/816314)
[^6]: [OpenZFS: Swap deadlock in 0.7.9](https://github.com/openzfs/zfs/issues/7734)
[^7]: [OpenZFS: raidz expansion, alpha preview 1](https://github.com/openzfs/zfs/pull/8853)
[^8]: [reddit: Formatting ZFS to use whole disk vs. partition](https://www.reddit.com/r/zfs/comments/enxxyx/formatting_zfs_to_use_whole_disk_vs_partition/)
[^9]: [reddit: ZoL 0.8.0 encryption: don't encrypt the pool root!](https://www.reddit.com/r/zfs/comments/bnvdco/zol_080_encryption_dont_encrypt_the_pool_root/)

{% include abbrev domain="computers" %}
