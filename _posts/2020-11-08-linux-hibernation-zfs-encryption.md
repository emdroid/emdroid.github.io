---
title: "Linux hibernation setup (ZFS, encryption)"
header:
  teaser: /assets/images/posts/hibernate.jpg
toc: true
categories:
  - Operating Systems
  - Linux
tags:
  - os
  - linux
  - swap
  - hibernation
  - encryption
---

![Lock](/assets/images/posts/hibernate.jpg){: .align-center .img-large}

Setting up the the Linux swap file for hibernation (Ubuntu), including ZFS/encryption setup.

<!-- more -->

## 1. Overview

The [Project Ares]({% post_url 2020-08-30-project-ares-part-5-ubuntu-on-zfs-raid %}#31-testing-the-hibernation) articles touched the hibernation setup together with the Ubuntu ZFS installation.

This is a more general article describing just the hibernation setup.

{% capture notice_contents %}
**<a name="ubuntu-note">Warning</a>**:

Note that the instructions are specific to the **Ubuntu distribution** and might not work for other distributions where I didn't test them.
{% endcapture %}

{% include notice level="warning" %}

### 1.1. What is the hibernation and how it works

**Hibernation** (or "suspend to disk") is a functionality allowing to save the computer state (mostly the contents of the RAM) on the disk, so that the machine can be shut down and resumed later from the disk image.

This has the following main benefits:
- the computer can be shut down and restored into its previous state later
- the start of the machine is faster (usually [^1]), as it doesn't need to load everything from scratch

This is similar to sleep or suspend-to-RAM modes (which can resume even faster), however the advantage of hibernation is that it survives the eventual AC power failure (power loss) as the state is saved in a persistent storage (i.e. the disk).

There is also the so-called **hybrid-sleep** (suspend-hybrid) mode, that is the combination of hibernation and sleep:
- the state is written to the disk first (so that it can eventually survive the AC power loss)
- then enters the sleep/suspend to RAM mode (so it can resume instantaneously in case the AC power wasn't lost)

See also the Wikipedia article for some more details and general overview: [Wikipedia: Hibernation](https://en.wikipedia.org/wiki/Hibernation_(computing))

[^1]: This however depends of the RAM size - generally with larger memory size more data need to be written to / read from the disk, thus with a huge amount of RAM the startup can be eventually even slower than when starting completely from scratch.

### 1.2. Hibernation implementation in Linux

Contrary to the Windows OS (that uses a separate "_hiberfil.sys_" file to save the hibernation state), the Linux uses the **swap partition** or file for that.

This adds some **constraints on the swap partition / file** in case the hibernation is to be used:
- **the size** of the swap partition or file needs to be at least as big as the RAM size (can eventually be the issue if using the swap partiton and upgrading the RAM size)
- the hibernation swap partition or file needs to be **accessible very early in the bootup** process (adds constraints to the partition/file location, encryption etc.)

## 2. Setting up the swap

### 2.1. Choosing the swap location

There are two possibilities to setup the swap location:

**a) Separate partition**:

- generally recommended approach, as it is much **easier to set up** for hibernation
- the disadvantage is that you **need to allocate the swap file size upfront** and it is difficult to change later (for example after you upgrade the RAM - repartitioning needed)

**b) File on the root partition**:

- the Ubuntu (and likely other distros as well) now also **supports hibernation to a file** on the root partition
- creating the swap file on the root partition is now the **default and recommended approach in Ubuntu**
- the advantage is that it is **easier to resize** later if e.g. RAM gets upgraded
- but it is **more difficult to setup for hibernation**
- in addition, the hibernation to a file is **not supported in some configurations**:
  - not supported on BtrFS
  - not supported on ZFS
  - in general, only supported with the "simple" file systems (where the swap file can be allocated contiguously and it's possible the retrieve its exact location on the disk)
- it is also even more **difficult to set it up when encryption is used**

### 2.2. The swap size

As mentioned, the swap size for hibernation needs to be **at least the size of RAM** (as the whole contents of the RAM needs to fit there).

There is the Ubuntu recommendation for the swap size: [How much swap do I need?](https://help.ubuntu.com/community/SwapFaq#How_much_swap_do_I_need.3F)

In addition to that it is recommended to **plan forward** in case you might eventually want to increase the memory size, thus **allocating the anticipated future size** for the swap (especially when using the swap partition, that might be very difficult to change later).

**Example**:

In case of the [Project Ares]({% post_url 2020-07-13-project-ares-part-1-choosing-the-hardware %}) I ended up using this swap configuration for hibernation:
- primary system SSD disk 500 GB
- secondary system HDD 750 GB (7200 RPM)
- first 500 GB of the secondary HDD used as the primary SSD mirror (fallback recovery)
- the remainder of the secondary HDD (250 GB left) used for the swap partition
- the initial RAM size 32 GB, but can eventually upgrade to 64 or even 128 GB without having to change the swap partition
- having "too much swap" doesn't hurt too much (except when short of disk space, which isn't the case on that machine)
- might be eventually better to use SSD for the hibernation (much faster save/restore), but for now the 7200 RPM HDD is still fast enough for this purpose (especially when using the hybrid sleep, when the computer actually doesn't need to restore from the disk normally)
- note that the hibernation swap is not mirrored (more about it below)

### 2.3. RAID / mirror configuration

I initially had the hibernation swap setup in mirrored configuration, using MD-RAID (in order to be able to start even in the case the primary or secondary disk fails).

However mirroring the swap partition causes some issues when hibernation is used:
- when the system hibernates, the mirror is not synchronized completely
- this causes the RAID to resync after every wake up from hibernation
- that means significant strain on the secondary disk every time the system is restored
- in particular, the MD-RAID needs to resync the whole partition (doesn't know which data are in use), which is not even necessary at that point as the data are no longer needed after the system has been resumed

For the above reasons I don't recommend mirroring the hibernation swap partition (or any swap partition as a matter of fact, even if not used for hibernation) - the swap is not essential for running the machine, and in the case of a disk failure, the system can simply ignore the missing swap partition and skip using the swap.

So per above I ended up using just a single non-mirrored swap partition, and I've set it up the following way:
- only using the secondary disk remaining space for the swap (the secondary HDD is larger than the primary SSD in my case)
- setting the swap partition as "nofail", which means that if the partition is not found (the disk failed) it is simply ignored and no swap/hibernation is being used
- the Ubuntu skips the resume if it cannot find the "resume device"
- so in the case of secondary disk failure (the disk containing the swap partition) the hibernation will not be able resume the system to the previous state, but will still be able to boot up and perform a complete boot from scratch (i.e. the machine will still become available)

### 2.4. Swap on separate partition

Preparing the setup:

```bash
# enter the admin console
sudo -i

# prefer to use the disk-by-id to avoid confusion with the disk assignment
ls -al /dev/disk/by-id/

# set the disk id appropriately
# (use the physical disk links without the "-partN")

# the drive for setting up the swap partition
DISK=/dev/disk/by-id/<tab-complete-the-disk-path>

# the swap partition number
SWAP_PART_NO=4
```

Creating the swap partition:

```bash
# list the existing partitions on the primary disk
sgdisk -p ${DISK}
# !!! double check it is the correct drive you want to use !!!
# (otherwise change the DISK variable appropriately)

# create the swap partition
sgdisk -n${SWAP_PART_NO}:0:0 -t${SWAP_PART_NO}:8300 -c${SWAP_PART_NO}:"Swap" ${DISK}

# check by listing the partitions
sgdisk -p ${DISK}

# make the swap filesystem
mkswap ${DISK}-part${SWAP_PART_NO}
```

Adding the swap partition entry:

```bash
# add the swap entry to the fstab
echo "UUID=$(blkid -s UUID -o value ${DISK}-part${SWAP_PART_NO}) none swap sw,discard,nofail,x-systemd.device-timeout=5 0 0" \
    >> /etc/fstab

# eventually edit the fstab and remove old/invalid records
vim /etc/fstab

# enable the swap
swapon -a

# check the status
swapon --summary
```

- note the "nofail" and "x-systemd.device-timeout" parameters - this is to ignore the setting if the swap partition cannot be found (e.g. when the disk failed).

### 2.2. Swap in file on root partition

{% capture notice_contents %}
**<a name="swap-file-note">Warning</a>**:

Note that although Ubuntu allows (and even recommends) using the swap file instead of a separate partition, it is somewhat **more difficult to setup** and doesn't work in all scenarios.

In particular, using a swap file for hibernation is **not supported** when the root is located on **non-standard file systems**, like BtrFS or ZFS.

In addition, if the root partition is on RAID (e.g. mirror), using hibernation could also cause it to desync, which means the mirror might need to be re-synced after the resume.
{% endcapture %}

{% include notice level="warning" %}

TBD - work in progress

## 3. Swap partition encryption

The standard encrypted swap configurations use the "_/dev/urandom_" to setup the partition with a new encryption key on every boot.

That however **doesn't work for hibernation**, because in order to load the hibernation image from the encrypted swap, the boot loader actually needs to know the encryption key.

In order to be able to resume from encrypted swap partition, these conditions apply:
- fixed encryption key is needed
- needs to be available early in the boot process
- that means the encryption key cannot even be loaded from the root partition, but must be available to the initramfs

One possible solution is to [load the key from network or USB]({% post_url 2020-10-21-unlocking-linux-encrypted-root-from-network-usb %}) or using a separate key vault partition.

Generating the swap partition key (storing in "_/etc/crypt/init/swap.key_" temporarily):

```bash
# the keys path
CRYPT_KEYS_PATH=/etc/crypt

# generate the swap partition LUKS key
# - using the Base64 encoding to only have printable chars in the key
dd if=/dev/urandom bs=512 skip=4 count=4 iflag=fullblock | base64 -w 0 \
    > ${CRYPT_KEYS_PATH}/init/swap.key
chmod go-rwx ${CRYPT_KEYS_PATH}/init/swap.key
```

Encrypting the partition (the partition itself can remain as before):

```bash
# unmount the current swap
swapoff -a

# setup the swap partition encryption
cryptsetup luksFormat -q -c aes-xts-plain64 -s 512 -h sha256 \
    -d ${CRYPT_KEYS_PATH}/init/swap.key ${DISK}-part${SWAP_PART_NO}

cryptsetup luksOpen -d ${CRYPT_KEYS_PATH}/init/swap.key ${DISK}-part${SWAP_PART_NO} swap

# create the swap filesystem
mkswap /dev/mapper/swap
```

Setting up the swap decryption:

a) Using the initramfs (the key is visible there):

```bash
# add the crypttab entry
echo "swap PARTUUID=$(blkid -s PARTUUID -o value ${DISK}-part${SWAP_PART_NO}) ${CRYPT_KEYS_PATH}/init/swap.key luks,discard,initramfs,nofail,x-systemd.device-timeout=5" \
    >> /etc/crypttab

# add the encrypted swap entry to the fstab
echo "/dev/mapper/swap none swap sw,discard,nofail,x-systemd.device-timeout=5 0 0" \
    >> /etc/fstab

# eventually edit the fstab and remove old/invalid records
vim /etc/crypttab
vim /etc/fstab
```

- if not done yet, setup the initramfs hooks for including the key:

```bash
# setup some basic protection of the encryption keys
echo "UMASK=0077" \
    >> /etc/initramfs-tools/initramfs.conf

# add the key pattern to be included in the initramfs
echo "KEYFILE_PATTERN=\"${CRYPT_KEYS_PATH}/init/*.key\"" \
    >> /etc/cryptsetup-initramfs/conf-hook
# check it has been added properly
less /etc/cryptsetup-initramfs/conf-hook
```

b) Using the key load script (e.g. when [loading the key from network or USB]({% post_url 2020-10-21-unlocking-linux-encrypted-root-from-network-usb %}))

- note that the script also needs to be added to the initramfs (see the above link)

```bash
# add the crypttab entry (using key script in ${CRYPT_KEYS_PATH}/scripts/init-swap.sh)
echo "swap PARTUUID=$(blkid -s PARTUUID -o value ${DISK}-part${SWAP_PART_NO}) none luks,key-script=${CRYPT_KEYS_PATH}/scripts/init-swap.sh,discard,initramfs,nofail,x-systemd.device-timeout=5" \
    >> /etc/crypttab

# add the encrypted swap entry to the fstab
echo "/dev/mapper/swap none swap sw,discard,nofail,x-systemd.device-timeout=5 0 0" \
    >> /etc/fstab

# eventually edit the fstab and remove old/invalid records
vim /etc/crypttab
vim /etc/fstab
```

After the encryption setup is done, the swap can be re-enabled:

```bash
# enable the swap
swapon -a

# check the status
swapon --summary
```

## 4. Enabling the hibernation

The hibernation is set up via the "RESUME" kernel parameters. This can be done either in the grub config, or in the initramfs:

a) Regular non-encrypted swap:

```bash
# add the resume option
echo "RESUME=UUID=$(blkid -s UUID -o value ${DISK}-part${SWAP_PART_NO})" > /etc/initramfs-tools/conf.d/resume
```

b) Using the encrypted swap:

```bash
# add the resume option
echo "RESUME=/dev/mapper/swap" > /etc/initramfs-tools/conf.d/resume
```

The initramfs needs to be updated afterwards:

```bash
# update the Grub configuration
update-grub

# re-create the initramfs image
update-initramfs -c -k all
```

## 5. Testing the hibernation

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

### 5.1. The hibernation/standby modes

There are various hibernation and standby modes:
- **suspend** (suspend to RAM, S3): enters the low power mode, just keeping the RAM at minimum power state to retain the data
- **hibernate** (suspend to disk, S4): saves the RAM contents to the disk and shuts down the computer
- **suspend-hybrid**: combination of the above two

The "_suspend-hybrid_" mode is especially interesting:
- it allows enter the sleep mode but the state is saved to the disk (persistent storage) at the same time
- that means the wake up is fast in the usual case (waking up just from sleep)
- but if the power is lost, the state can still be restored from the disk

The following table shows the differences between the power saving modes:

| Mode      | To RAM | To HDD | Suspend   | Restore   | Consumption | Survives power down |
| :-------- | :----: | :----: | :-------: | :-------: | :---------: | :-----------------: |
| Suspend   |   Yes  |   No   | Very fast | Very fast |     Low     |         No          |
| Hibernate |   No   |   Yes  |   Fast    |   Fast    |   Very low  |         Yes         |
| Hybrid    |   Yes  |   Yes  |   Fast    | Very fast |     Low     |         Yes         |


## Resources and references

- [Wikipedia: Hibernation](https://en.wikipedia.org/wiki/Hibernation_(computing))
- [Ubuntu: SwapFaq](https://help.ubuntu.com/community/SwapFaq)
- [Ubuntu: Hibernate with encrypted swap](https://help.ubuntu.com/community/EnableHibernateWithEncryptedSwap)

{% include abbrev domain="computers" %}
