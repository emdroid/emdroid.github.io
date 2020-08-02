---
title: "Project Ares: Part III - OS and file system considerations"
header:
  teaser: /assets/images/posts/nas-ares/3/nas-os.jpg
toc: true
sidebar:
  nav: "fs-ares"
categories:
  - Hardware
  - Storage
tags:
  - nas
  - server
  - os
  - linux
  - ubuntu
  - centos
  - freenas
  - truenas
  - openmediavault
  - rockstor
  - amahi
  - unraid
  - zfs
  - ext4
  - xfs
  - btrfs
  - lvm
  - luks
---

![NAS OS](/assets/images/posts/nas-ares/3/nas-os.jpg){: .align-center .img-large}

Sharing the experience of building a home NAS/VM server.

The 3rd part discusses the operating and file system of the new server, the requirements and the available alternatives.

<!-- more -->

Now that the hardware is ready, the next quest is to choose **The Right Operating System™** to be installed onto the server.
This decision has some important implications and consequences, as it might not be entirely easy to switch the OS afterwards.
Not that it is totally impossible, however once everything is set up and configured, we could find ourselves very hesitant to switch to a completely different system, especially when we don't know the new system well yet, whether or not it will have any issues e.g. with the HW and support everything what's needed, and in addition have to reconfigure everything again.
Such step might also include the need of data migration (if the new system doesn't support the current file system setup).

It is eventually also possible to try to setup a dual-boot system, however it is a bit tedious to begin with (as we'd be basically setting up everything and looking for the solutions twice, not even mentioning setting up the boot loader properly so that it can boot both systems), only to find out later that we anyway ended up with using just the first of the systems and even afraid to boot the other one as it is heavily outdated after not booting it up for the whole last year.

## 1. The requirements

The planned usage also implies some requirements on the operating system features and capabilities, some of which are essential, others nice to have.

1. **File system**:
  - RAID support: RAID-1 (mirror) for the system disk, RAID-5/6 for data
  - flexibility in adding the drives to the disk pool RAID (especially for the data drives)
  - encryption support (ideally including root/boot partition)
  - large volume support
  - data safety
  - thin provisioning support or equivalent to allow re-distribution of the free space
  - SSD support (TRIM etc.)
  - nice to have: snapshots, error detection/correction
2. **Network sharing**:
  - essential: Windows sharing (CIFS/samba)
  - nice to have: other sharing options like NFS, (S)FTP etc.
3. **Virtualization**:
  - running virtual machines of different systems (Linux, Windows, Solaris, eventually Mac OSX)
  - allocation of resources to particular VMs (memory, CPU cores, virtual disks, ...), HW pass-through
  - nice to have: Docker support
4. **Hardware friendly**:
  - support of all hardware used currently or eventually in the future
5. **Remote administration**:
  - being able to be managed remotely over the network
  - nice to have: administrative GUI
6. **Performance**:
  - the OS itself should take as less resources as possible
  - disk access performance
  - good performance of the hosted virtual machines
  - network card link aggregation support
7. **Flexibility**:
  - possibility to perform advanced tasks, like getting the root partition encryption key from the USB / network during booting etc.
  - nice to have: choice in using different file system formats, encryption solution providers etc.

## 2. The available options

There are 3 principal options (categories) of the operating systems that can be used here:

### 2.1. Regular operating systems

This category includes any non-specific server or desktop systems that include various Linux distributions (Ubuntu, RHEL/CentOS, OpenSUSE, ...) or Windows (desktop/server).

{% capture os_global_label %}
**Globally**
{% endcapture %}

{% capture os_global_advantages %}
- more flexibility (compared to NAS dedicated systems)
{% endcapture %}

{% capture os_global_limits %}
- less user friendly interface for NAS purposes
- require more advanced user to administer
{% endcapture %}

{% capture os_ubuntu_label %}
**[Ubuntu Linux](https://ubuntu.com/)**
{% endcapture %}

{% capture os_ubuntu_advantages %}
- most mainstream of Linux distros - lots of information and help available
- one of the most complete HW support
- packages generally very up to date
- most complete ZFS (ZoL) support (including boot partition)
- LTS editions available
{% endcapture %}

{% capture os_ubuntu_limits %}
- might be less stable at times because of the quicker adoption of new package versions and features (nevertheless still pretty stable according to the experience)
- still can be behind with HW support compared to Windows
{% endcapture %}

{% capture os_centos_label %}
**[CentOS Linux](https://www.centos.org/)**
{% endcapture %}

{% capture os_centos_advantages %}
- rockstable
- all editions are LTS
- mostly the same as the paid RHEL
{% endcapture %}

{% capture os_centos_limits %}
- somewhat behind with features and package versions
- less complete ZFS support
- some of my NAS HW not supported out of the box<br />(HBA disk controller driver to be side-loaded)
{% endcapture %}

{% capture os_linux_label %}
**Other Linux distributions**
{% endcapture %}

{% capture os_linux_advantages %}
- some other distros might have various advantages (very small, very customizable to exactly match the used HW etc.)
- a lot of options to choose from (might actually also be seen as a disadvantage by some)
{% endcapture %}

{% capture os_linux_limits %}
- less mainstream = less thoroughly tested by other users
- generally less information and help available (= require more advanced user)
- HW support can be suboptimal  if more exotic devices are used
{% endcapture %}

{% capture os_windows_label %}
**Windows** (Server / Desktop)
{% endcapture %}

{% capture os_windows_advantages %}
- more familiar to less advanced users
- maintenance might be easier for some
- ultimate hardware support
- compatible with most SSD manufacturers (TRIM, HW encryption etc.) - some might have issues in Linux
{% endcapture %}

{% capture os_windows_limits %}
- cannot be installed/used headless (without the GUI)
- higher HW requirements of the system alone (memory, CPU, system disk space)
- limited data sharing options (no NFS etc.)
- no text-only remote administration (requires RDP/VNC/...)
- VM+Docker don't work at the same time (VMware/VirtualBox don't run with Hyper-V, Docker doesn't run without Hyper-V)
- (very) expensive: need at least Pro version for RDP (single user), Server licenses very expensive (easily more than the whole hardware, + CAL licenses per remote user)
- difficult to make it reasonably secure (compared to Linux)
- no possibility of password-less startup (e.g. key from USB/network) on SW encrypted system partition
- snapshots (shadow copy) removed in windows 10
{% endcapture %}

<table>
  <thead>
    <tr>
      <th width="20%">Option</th>
      <th width="40%">Advantages</th>
      <th width="40%">Limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="1" class="box-top">{{ os_global_label      }}</td>
      <td markdown="1" class="box-top">{{ os_global_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_global_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_ubuntu_label      }}</td>
      <td markdown="1" class="box-top">{{ os_ubuntu_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_ubuntu_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_centos_label      }}</td>
      <td markdown="1" class="box-top">{{ os_centos_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_centos_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_linux_label      }}</td>
      <td markdown="1" class="box-top">{{ os_linux_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_linux_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_windows_label      }}</td>
      <td markdown="1" class="box-top">{{ os_windows_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_windows_limits     }}</td>
    </tr>
  </tbody>
</table>

### 2.2. NAS specific operating systems

Here we can have a look onto various NAS dedicated systems [^1] [^2] that usually provide nice administrative GUIs (generally via web interface). Most of these are Linux or FreeBSD based.

{% capture os_nas_global_label %}
**Globally**
{% endcapture %}

{% capture os_nas_global_advantages %}
- more user-friendly interface (usually via web administration)
- out-of-the-box experience
- require less advanced user
- usually some kind of plugin support
{% endcapture %}

{% capture os_nas_global_limits %}
- less flexible (mostly only what the web GUI allows)
- some advanced setup options (root encryption etc.) might be very difficult or impossible to achieve
{% endcapture %}

{% capture os_nas_freenas_label %}
**[FreeNAS](https://www.freenas.org/)** / **[TrueNAS](https://www.truenas.com/)**
{% endcapture %}

{% capture os_nas_freenas_advantages %}
- out-of-the-box full ZFS support
- very nice user interface close to dedicated NAS manufacturers (QNAP, Synology etc.)
- VM / Docker support
- link aggregation support
{% endcapture %}

{% capture os_nas_freenas_limits %}
- FreeBSD based - slower than Linux nowadays [^3] [^4]
- the GUI is relatively complex thus harder to use for beginners
- ZFS support only - no support of ZFS over MD RAID etc.
- no RAID-5/6 disk addition/removal (ZFS limitation)
- VM performance is [reportedly suboptimal](https://www.google.com/search?q=freenas+vm+slow)
- doesn't support HW pass-through (directly in GUI)
{% endcapture %}

{% capture os_nas_omvault_label %}
**[OpenMediaVault](https://www.openmediavault.org/)**
{% endcapture %}

{% capture os_nas_omvault_advantages %}
- based on Debian Linux
- low system requirements
- VM support (via the Cocpit plugin)
{% endcapture %}

{% capture os_nas_omvault_limits %}
- no ZFS or BtrFS support (only EXT, XFS)
- initially successor of FreeNAS, but still has less features
- only single main developer
{% endcapture %}

{% capture os_nas_rockstor_label %}
**[RockStor](http://rockstor.com/)**
{% endcapture %}

{% capture os_nas_rockstor_advantages %}
- GUI interface on top of CentOS
{% endcapture %}

{% capture os_nas_rockstor_limits %}
- only BtrFS support (no ZFS)
- less features than FreeNAS
- no direct support for VM / Docker
- might have some issues eventually because of the CentOS dropping the BtrFS support
{% endcapture %}

{% capture os_nas_amahi_label %}
**[Amahi](https://www.amahi.org/)**
{% endcapture %}

{% capture os_nas_amahi_advantages %}
- media server focused
- large MediaApp and WebApp store
{% endcapture %}

{% capture os_nas_amahi_limits %}
- no ZFS or BtrFSsupport (only Ext, XFS)
- less features than FreeNAS
- no direct support for VM / Docker
{% endcapture %}

{% capture os_nas_unraid_label %}
**[UnRAID](https://unraid.net/)**
{% endcapture %}

{% capture os_nas_unraid_advantages %}
- better data safety: separate parity drive (data of the non-failed disks still fully readable)
- supports VM, Docker, HW pass-through
{% endcapture %}

{% capture os_nas_unraid_limits %}
- slower disk speed than RAID (maxed by single drive speed as there is no striping involved)
- not free
- limited max number of drives in the cheaper pricing plans
- requires USB with the license key for every boot
- doesn't seem to support root partition encryption
{% endcapture %}

{% capture os_nas_other_label %}
**Other NAS distributions**
{% endcapture %}

{% capture os_nas_other_advantages %}
- various features and quality of GUI
{% endcapture %}

{% capture os_nas_other_limits %}
- less used distributions may be less battle tested
- smaller projects tend to lose interest and support after some time
{% endcapture %}

<table>
  <thead>
    <tr>
      <th width="20%">Option</th>
      <th width="40%">Advantages</th>
      <th width="40%">Limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="1" class="box-top">{{ os_nas_global_label      }}</td>
      <td markdown="1" class="box-top">{{ os_nas_global_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_nas_global_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_nas_freenas_label      }}</td>
      <td markdown="1" class="box-top">{{ os_nas_freenas_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_nas_freenas_limits }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_nas_omvault_label      }}</td>
      <td markdown="1" class="box-top">{{ os_nas_omvault_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_nas_omvault_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_nas_rockstor_label      }}</td>
      <td markdown="1" class="box-top">{{ os_nas_rockstor_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_nas_rockstor_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_nas_amahi_label      }}</td>
      <td markdown="1" class="box-top">{{ os_nas_amahi_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_nas_amahi_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_nas_unraid_label      }}</td>
      <td markdown="1" class="box-top">{{ os_nas_unraid_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_nas_unraid_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ os_nas_other_label      }}</td>
      <td markdown="1" class="box-top">{{ os_nas_other_advantages }}</td>
      <td markdown="1" class="box-top">{{ os_nas_other_limits     }}</td>
    </tr>
  </tbody>
</table>

### 2.3. Virtualization platforms

Bare metal hypervisors used to launch other systems in the virtualized environment.
Popular options are **KVM**, **Xen**, **ESXi**, **Hyper-V**.

Although this category might be beneficial for the planned usage as a virtual machine platform, I didn't evaluate this option in detail as I currently don't have much experience in this area and for what I know they can be quite difficult to install and maintain (and use properly) - at least as the "bare metal" option (also taking into account that I'm setting up a single server in a home environment, for a business running multiple servers the requirements and the decision may be very different).

One nice example is also [Proxmox](https://www.proxmox.com/), which can serve as a virtualization platform (using KVM+QEMU on top of a Debian base system) or the [ESOS](http://www.esos-project.com/) project.

## 3. The operating system chosen

After carefully evaluating all the options, I decided to use a **Linux server distribution**, mainly because of the flexibility they provide.
The NAS-dedicated solutions like FreeNAS offer a very nice administrative interface and are thus easier to setup and maintain, but it might be difficult to achieve some advanced scenarios (like root file system encryption, they have RAID layout and file system choice limitations etc.).

On the other side, such decision implies more work setting up and maintaining the system, changing the configuration etc. - so if you are not a "geeky geek" wanting to control each and every aspect of the system and just want "something that works" (and works pretty well actually), I would definitely recommend choosing one of the NAS dedicated system like FreeNAS (would be my preferred choice) or OpenMediaVault.
They have a very nice interface, plugin systems etc. and are very user friendly.

To be honest, I must admit that I actually have a strong bias towards the "regular" Linux distributions for 2 main reasons:

- I already have a good amount of experience running (another) NAS machine using the Ubuntu Server (that is over 10 years old and still running, although it will be decommissioned once this project is complete)
- and I don't currently have much experience with the NAS dedicated systems (except for the other QNAP NAS box, where I find the QTS interface pretty nice and user friendly, but indeed limited in some aspects)

### 3.1. The distribution chosen

So after I made the decision to use a Linux distribution, now the question was which one to choose.
I have some experience with both Ubuntu and CentOS, my old NAS box uses Ubuntu but the CentOS is even more "server oriented" and designed for stability, so both choices have their merits.

To see which one will fit better, I actually tried to install both (even in a dual boot configuration initially, but as I also wanted to use ZFS, I found out that they somewhat clash if used within the same ZFS pool, despite being installed onto different datasets).
In particular, I was testing these versions:

- [Ubuntu Legacy Server 20.04 LTS](http://cdimage.ubuntu.com/ubuntu-legacy-server/releases/20.04/release/) (the pure Server version without GUI)
- [CentOS 8.2.2004 Minimal Server](https://www.centos.org/download/)

As you can notice, I tend to install the minimal server versions (without the GUI first, as the server will be managed remotely over the SSH anyway, although later on I might install some kind of GUI in minimal configuration for eventual VNC access).

During testing of both of the systems I realized that for the requirements I wanted to achieve (boot/root on ZFS in RAID-1, encrypted root) the <span style="color:red;">**CentOS has various limitations**</span> that make it a bit harder to use in such configuration:

- **does support ZFS, but not for the boot partition** (in particular, the boot partition can't even be part of LVM, or at least the installer doesn't support it) - it might eventually be possible to do it somehow (as the Grub boot loader should support ZFS), but it doesn't work out of the box (whereas it works in the Ubuntu)
- puts the **Grub menu configuration directly onto the EFI** partition (which makes it tedious to mirror as it is updated after each Grub change e.g. after any  kernel update) - Ubuntu only puts a "link" file on the EFI that never changes afterwards
- it **doesn't** (directly) **support the HBA LSI-2008 card** (the one I use for the data disks) out of the box, as the [SAS-2 drivers (mpt2sas) were removed in the RHEL / CentOS 8](https://access.redhat.com/discussions/3722151) - it is still possible to [side-load them via the El-Repo repository](https://elrepoproject.blogspot.com/2019/08/rhel-80-and-support-for-removed-adapters.html), however it is a bit tedious (especially if it needs to boot from there, although that is not required for my use-case), and it is also a sign that this hardware might not be supported that well in the future in this distribution - whereas the Ubuntu still supports it out of the box

So after this testing and evaluation, the winner is:

<span style="font-size:larger;">**[Ubuntu Legacy Server 20.04 LTS](http://cdimage.ubuntu.com/ubuntu-legacy-server/releases/20.04/release/)**</span>
{: .text-center}

## 4. The file system layout

The operating system choice affects the available choices of the file system features and layout. The Ubuntu Linux distribution selected actually offers one of the most wide range of possible choices, but of course there are still some limits.

The following tables show the various possibilities and features available.

### 4.1. File system format

{% capture fs_fmt_ext_label %}
**Ext3** / **Ext4**
{% endcapture %}

{% capture fs_fmt_ext_advantages %}
- fast(est) on regular size drives
- most well known, proven file system
- easiest to recover in case of issues
- Ext4 includes journaling  
(and much faster file system check)
{% endcapture %}

{% capture fs_fmt_ext_limits %}
- starts being slow on very large volumes
- various file system limits (fixed inode size)
- missing advanced features of more modern formats  
(COW, snapshots, error detection etc.)
{% endcapture %}

{% capture fs_fmt_xfs_label %}
**XFS** [^5]
{% endcapture %}

{% capture fs_fmt_xfs_advantages %}
- faster than Ext4 on very large volumes [^6]
- faster for big sizes and with parallel writes [^7] [^8]
- larger caches (see limitations)
- larger file system limits than Etx4
{% endcapture %}

{% capture fs_fmt_xfs_limits %}
- more susceptible to failures because of larger caches  
(UPS recommended)
- no CoW, snapshots (yet)
{% endcapture %}

{% capture fs_fmt_btrfs_label %}
**BtrFS** [^5]
{% endcapture %}

{% capture fs_fmt_btrfs_advantages %}
- modern feature-rich file system
- copy-on-write snapshots
- file system level snapshots
- compression
- data integrity checking
- SSD support (TRIM etc.)
- SW RAID features included
{% endcapture %}

{% capture fs_fmt_btrfs_limits %}
- slow compared to Ext4, XFS, ZFS
- large memory footprint
- less mature so far
- hard to recover data if something goes wrong
- only supported by some newer distributions
- some distributions dropping its support (RHEL/CentOS)
- COW might be problematic for VM machine disks (fragmentation issues)
{% endcapture %}

{% capture fs_fmt_zfs_label %}
**ZFS**
{% endcapture %}

{% capture fs_fmt_zfs_advantages %}
- most feature rich (similar to BtrFS)
- faster than BtrFS in most use-cases
- copy-on-write snapshots
- file system-level snapshots
- compression, encryption support
- data integrity checking and recovery
- thin provisioning
- datasets - mitigate the need of thin provisioning
- SW RAID features included
{% endcapture %}

{% capture fs_fmt_zfs_limits %}
- large memory footprint
- mature on Solaris, but relatively new on Linux
- lack of full support - need special steps to be used for root/boot partitions
- COW might be problematic for VM machine disks (fragmentation issues)
{% endcapture %}

<table>
  <thead>
    <tr>
      <th width="20%">Candidate</th>
      <th width="40%">Advantages</th>
      <th width="40%">Limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="1" class="box-top">{{ fs_fmt_ext_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_fmt_ext_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_fmt_ext_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_fmt_xfs_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_fmt_xfs_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_fmt_xfs_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_fmt_btrfs_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_fmt_btrfs_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_fmt_btrfs_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_fmt_zfs_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_fmt_zfs_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_fmt_zfs_limits     }}</td>
    </tr>
  </tbody>
</table>

### 4.2. Volume management

{% capture fs_part_raw_label %}
**Raw partitions**
{% endcapture %}

{% capture fs_part_raw_advantages %}
- most straightforward to create and use
- easiest recovery
{% endcapture %}

{% capture fs_part_raw_limits %}
- poor flexibility (difficult to move, resize etc.)
{% endcapture %}

{% capture fs_part_lvm_label %}
**LVM**
{% endcapture %}

{% capture fs_part_lvm_advantages %}
- flexibility in creating volume groups, volumes, resizing, moving etc.
- thin provisioning support
- CoW snapshots (volume level)
- SW RAID support included
{% endcapture %}

{% capture fs_part_lvm_limits %}
- volume level snapshots are reportedly slow  
(except when thin provisioning used) [^9]
- RAID support less flexible (uses mdraid under the hood, but can't use the md tools directly and the LVM tools are less mature)
- need to keep separate free space for the snapshots  
(requires pre-planning) [^9]
{% endcapture %}

{% capture fs_part_zfs_label %}
**ZFS**
{% endcapture %}

{% capture fs_part_zfs_advantages %}
- dataset functionality - can have one large volume with datasets as particular mountpoints  
(= no actual need of thin provisioning, they share all the available space)
- fast CoW file-level snapshots
- very flexible in adding/removing volumes
- volumes (datasets) are hierarchical (can be ordered like a tree, inheriting options etc.)
{% endcapture %}

{% capture fs_part_zfs_limits %}
- incomplete Linux support (boot/root partition)
{% endcapture %}

<table>
  <thead>
    <tr>
      <th width="20%">Candidate</th>
      <th width="40%">Advantages</th>
      <th width="40%">Limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="1" class="box-top">{{ fs_part_raw_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_part_raw_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_part_raw_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_part_lvm_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_part_lvm_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_part_lvm_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_part_zfs_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_part_zfs_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_part_zfs_limits     }}</td>
    </tr>
  </tbody>
</table>

### 4.3. RAID solution

{% capture fs_raid_none_label %}
**No RAID**
{% endcapture %}

{% capture fs_raid_none_advantages %}
- easiest to recover
{% endcapture %}

{% capture fs_raid_none_limits %}
- no redundancy
{% endcapture %}

{% capture fs_raid_md_label %}
**MD RAID** (SW)
{% endcapture %}

{% capture fs_raid_md_advantages %}
- native Linux option
- mature and proven option
- can be managed separately
- flexibility - possible to create degraded array, add/remove drive from the array etc.
{% endcapture %}

{% capture fs_raid_md_limits %}
- separate tool: extra maintenance needed
- doesn't understand the underlying file system: slower to replicate and rebuild (need to replicate everything including the free space)
{% endcapture %}

{% capture fs_raid_lvm_label %}
**LVM RAID** (SW)
{% endcapture %}

{% capture fs_raid_lvm_advantages %}
- if LVM used for partitioning as well, just a single integrated tool
{% endcapture %}

{% capture fs_raid_lvm_limits %}
- mostly the same as MD RAID
- still less mature and flexible than the raw MD RAID [^10]
{% endcapture %}

{% capture fs_raid_zfs_label %}
**ZFS RAID** (SW)
{% endcapture %}

{% capture fs_raid_zfs_advantages %}
- direct file system-level RAID support
- knows about used/free space - significantly faster replication and rebuilding
- supports most RAID types
{% endcapture %}

{% capture fs_raid_zfs_limits %}
- can only extend array, not shrink (remove drives)
- for stripped redundant RAID types (5/6/...) cannot expand existing RAID, must add a new RAID VDEV group [^11]
{% endcapture %}

{% capture fs_raid_unraid_label %}
**UnRAID** (SW)
{% endcapture %}

{% capture fs_raid_unraid_advantages %}
- separate parity drive(s)
- the data is not striped, ie. if multiple drives fail, the data of the working drives can still be read in full (contrary to regular RAID-5/6)
- there are free alternatives (SnapRAID + MergeFS)
{% endcapture %}

{% capture fs_raid_unraid_limits %}
- the UnRAID OS is not free (and the licenses limit the max amount of disks to use)
- the free alternative (SnapRAID) involves explicit sync (e.g. via cron)
{% endcapture %}

{% capture fs_raid_hw_label %}
**Hardware RAID**
{% endcapture %}

{% capture fs_raid_hw_advantages %}
- speed (a separate chip handles the RAID operations)
- no extra CPU load
{% endcapture %}

{% capture fs_raid_hw_limits %}
- special HW required
- very expensive (the cheaper controllers are not HW and/or don't support parity RAID levels like 5/6)
- difficult to restore data if the controller fails (need another same or compatible controller)
- need special drivers to install the system onto them
- sometimes difficulties to access disk info (SMART status etc.)
{% endcapture %}

<table>
  <thead>
    <tr>
      <th width="20%">Candidate</th>
      <th width="40%">Advantages</th>
      <th width="40%">Limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="1" class="box-top">{{ fs_raid_none_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_none_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_none_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_raid_md_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_md_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_md_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_raid_lvm_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_lvm_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_lvm_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_raid_zfs_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_zfs_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_zfs_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_raid_unraid_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_unraid_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_unraid_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_raid_hw_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_hw_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_raid_hw_limits     }}</td>
    </tr>
  </tbody>
</table>

### 4.4. Encryption

{% capture fs_enc_luks_label %}
**LUKS**
{% endcapture %}

{% capture fs_enc_luks_advantages %}
- Linux native solution
- proven and reliable
- supports SSD (TRIM etc.)
- up to 8 different keys to unlock (each key can unlock)
{% endcapture %}

{% capture fs_enc_luks_limits %}
- specific header format - can tell that the drive is encrypted by LUKS
- no layered encryption support
- no hidden volume support
{% endcapture %}

{% capture fs_enc_tc_label %}
**TrueCrypt** / **VeraCrypt**
{% endcapture %}

{% capture fs_enc_tc_advantages %}
- encrypted disk looks like random data (can't tell it is encrypted and how)
- can combine multiple keyfiles / passwords
- layered encryption support
- hidden volume support
{% endcapture %}

{% capture fs_enc_tc_limits %}
- might not support SSD trim well (although VeraCrypt claims TRIM support to some extent)
{% endcapture %}

{% capture fs_enc_zfs_label %}
**ZFS native encryption**
{% endcapture %}

{% capture fs_enc_zfs_advantages %}
- integrated into ZFS (when ZFS is used)
- various pasphrase/key possibilities (prompt, file, https)
{% endcapture %}

{% capture fs_enc_zfs_limits %}
- reportedly poor performance [^12]
- cannot encrypt the root file system / datasets
{% endcapture %}

{% capture fs_enc_hw_label %}
**Hardware encryption**
{% endcapture %}

{% capture fs_enc_hw_advantages %}
- provided by some disks (mostly SSD)
- handled by the disk itself (TPM module) - no CPU load
- full disk encryption (including boot partition etc.)
- the OS "doesn't know" it runs on an encrypted drive - completely transparent
{% endcapture %}

{% capture fs_enc_hw_limits %}
- security weaknesses found in many self-encrypting SSD firmware implementations [^13]
- either only TPM is used, or the user must always provide the password on boot (cannot get the key e.g. via network) - might not be suitable for "remote" servers
{% endcapture %}

<table>
  <thead>
    <tr>
      <th width="20%">Candidate</th>
      <th width="40%">Advantages</th>
      <th width="40%">Limitations</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="1" class="box-top">{{ fs_enc_luks_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_enc_luks_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_enc_luks_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_enc_tc_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_enc_tc_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_enc_tc_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_enc_zfs_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_enc_zfs_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_enc_zfs_limits     }}</td>
    </tr>
    <tr>
      <td markdown="1" class="box-top">{{ fs_enc_hw_label      }}</td>
      <td markdown="1" class="box-top">{{ fs_enc_hw_advantages }}</td>
      <td markdown="1" class="box-top">{{ fs_enc_hw_limits     }}</td>
    </tr>
  </tbody>
</table>

### 4.5. What others use

Let's check what file systems the well-known NAS device manufacturers use:

- **QNAP**: [Ext4: The preferred file system](https://www.qnap.com/solution/qnap-ext4/en-us/) (with their own *[proprietary block-based snapshot technology](https://www.qnap.com/solution/snapshots/en-us/)*)
- **Synology**: [How Btrfs protects your company's data](https://www.synology.com/en-us/dsm/Btrfs)
- **TerraMaster**: [BtrFS](https://www.terra-master.com/global/tos/)
- **Netgear**: older ReadyNAS devices vere using Ext2/3, newer support BtrFS

## 5. The file system layout decisions

Considering the above options, I've picked the following options:

### 5.1. The system disk

- 2 disks, mirrored on the partition level (no full-disk encryption)
- GPT partition table (large drive support, automatic backup)
- **EFI System Partition**:
  - raw GPT partition
  - no RAID - copied to other drives manually once complete  
(RAID-ing the EFI is possible with MD RAID and metadata 1.0, but creates some issues with OS installers etc.)
- **boot partition**:
  - ZFS, separate pool (limited features neede for Grub to boot)
  - no encryption (would always need to provide password for Grub)
  - ZFS mirror (mirroring works fine in ZFS)
- **root/home partitions**:
  - LUKS encrypted partition on each mirror drive
  - ZFS (full features) over the LUKS layer  
(poor performance of the ZFS native encryption + can't encrypt root anyway)
  - ZFS-level mirror over the LUKS partitions
  - no thin provisioning needed

### 5.2. The data disks

- 4 x 8 TB via the HBA adapter (+ various 1-2 TB disks from the old NAS box once decommissioned)
- GPT partitioning
- single GPT partition over the each whole drive
- RAID-5 configuration with Linux mdraid (ZFS raidz doesn't allow to extend the array) on the partition level (not physical disk level [^14])
- single LUKS or Truecrypt/Veracrypt (not decided yet) layer over the whole mdraid (poor performance of the ZFS encryption [^12])
- ZFS over the encryption layer
- no thin provisioning needed (data sets share the whole space anyway)

Note that this setup also has some downsides, in particular not using the native ZFS RAID (for the data drives) and encryption but using ZFS on top of other layers somewhat nullifies the ZFS safety, error detection and recovery (there may be errors happening upstream that the ZFS cannot detect and/or repair).

It has some performance implications as well - the ZFS-level RAID would be able to synchronize/rebuild the array on the file system level (i.e. knows which data are in use and only copies those), whereas the MD RAID working on the block level doesn't have that knowledge and must therefore (re)sync the whole drive(s).

## 6. Next steps

Stay tuned for the next part, where I will get into the details of installing the chosen OS, in particular onto an encrypted ZFS root partition in the RAID-1 mode.

## Resources and references

[^1]: [Top 20 Best Linux NAS Solutions and Linux SAN Software](https://www.ubuntupit.com/best-linux-nas-solutions-and-linux-san-software/)
[^2]: [8 Free NAS Storage OS / Software For Small Business Enterprise](https://www.geckoandfly.com/24910/nas-storage-os-small-business-enterprise/)
[^3]: [FreeBSD vs. Linux Scaling Up To 128 Threads With The AMD Ryzen Threadripper 3990X](https://www.phoronix.com/scan.php?page=article&item=3990x-freebsd-bsd&num=5)
[^4]: [DragonFlyBSD vs. FreeBSD vs. Ubuntu 20.04 On Intel's Core i9 10900K Comet Lake](https://www.phoronix.com/scan.php?page=article&item=comet-lake-bsd)
[^5]: [What’s the Difference Between Linux EXT, XFS, and BTRFS Filesystems](https://www.electronicdesign.com/industrial-automation/article/21804944/whats-the-difference-between-linux-ext-xfs-and-btrfs-filesystems)
[^6]: [EXT4 / XFS / Btrfs RAID Performance](https://www.phoronix.com/scan.php?page=article&item=linux54-hdd-raid)
[^7]: [How to Choose Your Red Hat Enterprise Linux File System](https://access.redhat.com/articles/3129891)
[^8]: [XFS: the filesystem of the future?](https://lwn.net/Articles/476263/)
[^9]: [Filesystem vs Volume Level Snapshots](https://serverfault.com/a/300966)
[^10]: [LVM RAID vs MD RAID](https://unix.stackexchange.com/a/182503)
[^11]: [The 'Hidden' Cost of Using ZFS for Your Home NAS](https://louwrentius.com/the-hidden-cost-of-using-zfs-for-your-home-nas.html)
[^12]: [Encrypted dataset still slow and high load with ZoL 0.8.3](https://www.reddit.com/r/zfs/comments/f3w6v0/encrypted_dataset_still_slow_and_high_load_with/)
[^13]: [Flaws in Popular SSD Drives Bypass Hardware Disk Encryption](https://www.bleepingcomputer.com/news/security/flaws-in-popular-ssd-drives-bypass-hardware-disk-encryption/)
[^14]: [What's the difference between creating mdadm array using partitions or the whole disks directly](https://unix.stackexchange.com/a/323425)

{% include abbrev domain="computers" %}
