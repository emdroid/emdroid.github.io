---
title: "Minimal installation of Solaris 10 with Sun Studio: System installation"
toc: true
sidebar:
  nav: "sol-10"
hidden: true
categories:
  - Operating Systems
  - UNIX
---

## 5. Extras

### 5.1. CSW repository

The [Open Community SoftWare](https://www.opencsw.org/) (OpenCSW) project is an additional repository of software packages for Solaris.
It provides newer package versions and/or packages which do not exist in Solaris originally (Git, CMake, etc.).

The repository is added as an additional package (needs to be done under the superuser - "`su -`"):

```
# pkgadd -d http://get.opencsw.org/now

## Downloading...

..............25%..............50%..............75%..............100%

## Download Complete


The following packages are available:
  1  CSWpkgutil     pkgutil - Installs Solaris packages easily
                    (all) 2.6.7,REV=2014.10.16

Select package(s) you wish to process (or 'all' to process
all packages). (default: all) [?,??,q]:
```

The package installs the CSW package management utility ("_pkgutil_"), which works separately from the original Solaris package manager.

The packages are installed in a separate directory ("_/opt/csw_"), therefore the paths need to be updated.
The user path (it you already didn't do that after the system installation):

```
# chmod u+w /etc/default/login
# vi /etc/default/login
```

Find the PATH setting and insert into it:

```bash
PATH=/usr/bin:/bin:/usr/sbin:/opt/csw/bin:/opt/solarisstudio12.4/bin:/usr/sfw/bin:/usr/sfw/i386-sun-solaris2.10/bin
```

The superuser ("_root_") paths are kept separately in the "_/etc/default/su_" file, so the CSW path should be added there as well:

```
# chmod u+w /etc/default/su
# vi /etc/default/su
```

The setting is kept in the "_SUPATH_" variable:

```bash
SUPATH=/usr/sbin:/usr/bin:/opt/csw/sbin
```

Note that the Sun Studio / SFW paths are normally not added for the root (as those are better to only run under the regular user), but the CSW path should be added there to have direct access to the CSW "_pkgutil_".
Note also that the order of the dirs for the superuser is different.

When done, it is recommended to change the permissions back to read-only:

```
# chmod u-w /etc/default/login
# chmod u-w /etc/default/su
```

Logout and login back (or restart the machine) for the settings to take effect.

Next initialize the CSW database with the "`-U`" parameter:

```
# pkgutil -U
=> Fetching new catalog and descriptions (http://mirror.opencsw.org/opencsw/testing/i386/5.10) if available ...
==> 3986 packages loaded from /var/opt/csw/pkgutil/catalog.mirror.opencsw.org_opencsw_testing_i386_5.10
```

Then the status of packages can be checked, for example:

```
# pkgutil -a vim
common               package              catalog                        size
gvim                 CSWgvim              8.0.238,REV=2017.01.30       1.2 MB
vim                  CSWvim               8.0.238,REV=2017.01.30       1.1 MB
vimrt                CSWvimrt             8.0.238,REV=2017.01.30       9.0 MB
```

And a particular package installed:

```
# pkgutil -y -i vim
Solving needed dependencies ...
Solving dependency order ...
Install 11 NEW packages:
        CSWcommon-1.5,REV=2010.12.11 (opencsw/testing)
        CSWggettext-data-0.19.8,REV=2016.09.08 (opencsw/testing)
        CSWlibgcc-s1-5.2.0,REV=2015.07.31 (opencsw/testing)
        CSWlibiconv2-1.14,REV=2011.08.07 (opencsw/testing)
        CSWlibintl9-0.19.8,REV=2016.09.08 (opencsw/testing)
        CSWlibncurses6-6.0,REV=2016.04.01 (opencsw/testing)
        CSWlibpython2-7-1-0-2.7.11,REV=2016.03.14 (opencsw/testing)
        CSWterminfo-6.0,REV=2016.04.01 (opencsw/testing)
        CSWterminfo-rxvt-unicode-9.20,REV=2014.10.31 (opencsw/testing)
        CSWvim-8.0.238,REV=2017.01.30 (opencsw/testing)
        CSWvimrt-8.0.238,REV=2017.01.30 (opencsw/testing)
Total size: 15.7 MB
=> Fetching CSWcommon-1.5,REV=2010.12.11 (1/11) ...
=> Fetching CSWterminfo-rxvt-unicode-9.20,REV=2014.10.31 (2/11) ...
...
Installation of <CSWvim> was successful.
```

See also here for additional details: [OpenCSW - Getting started](https://www.opencsw.org/manual/for-administrators/getting-started.html)

### 5.2. Secondary disk

**a) Preparing the disk partition**

A secondary drive can be formatted/partition created by using the "_format_" command:

```
# format
Searching for disks...
Inquiry failed for this logical diskInquiry failed for this logical diskdone

AVAILABLE DISK SELECTIONS:
       0. c0d0 <▒x▒▒▒▒▒▒▒▒▒@▒▒▒ cyl 2607 alt 2 hd 255 sec 63>
          /pci@0,0/pci-ide@7,1/ide@0/cmdk@0,0
       1. c0d1 <▒x▒▒▒▒▒▒▒▒▒@▒▒▒ cyl 13052 alt 2 hd 255 sec 63>
          /pci@0,0/pci-ide@7,1/ide@0/cmdk@1,0
Specify disk (enter its number):
```

If the secondary disk was newly added after the installation, it might not show in the list.
In that case, a "_devfsadm_" cleanup is necessary:

```
# devfsadm -C -c disk -v
devfsadm[429]: verbose: symlink /dev/dsk/c0d1s0 -> ../../devices/pci@0,0/pci-ide@7,1/ide@0/cmdk@1,0:a
devfsadm[429]: verbose: symlink /dev/dsk/c0d1s1 -> ../../devices/pci@0,0/pci-ide@7,1/ide@0/cmdk@1,0:b
...
```

See here for further troubleshooting details: [How to make Solaris rescan disk info after hotswap](https://serverfault.com/questions/556929/how-to-make-solaris-rescan-disk-info-after-hotswap)

In the "_format_" utility, select the particular disk (in the above case, disk number 1):

```
Specify disk (enter its number): 1
selecting c0d1
Controller working list found
[disk formatted, defect list found]

FORMAT MENU:
        disk       - select a disk
        type       - select (define) a disk type
        partition  - select (define) a partition table
        current    - describe the current disk
        format     - format and analyze the disk
        fdisk      - run the fdisk program
        repair     - repair a defective sector
        show       - translate a disk address
        label      - write label to the disk
        analyze    - surface analysis
        defect     - defect list management
        backup     - search for backup labels
        verify     - read and display labels
        save       - save new disk/partition definitions
        volname    - set 8-character volume name
        !     - execute , then return
        quit
format>
```

Run the "_fdisk_" command. If the disk is new, fdisk will ask to create a new Solaris partition (enter '_y_' to create the default partition):

```
format> fdisk
No fdisk table exists. The default partition for the disk is:

  a 100% "SOLARIS System" partition

Type "y" to accept the default partition,  otherwise type "n" to edit the
 partition table.
y
format>
```

Use '_p_' to enter the "_partition_" menu:

```
format> p

PARTITION MENU:
        0      - change `0' partition
        1      - change `1' partition
        2      - change `2' partition
        3      - change `3' partition
        4      - change `4' partition
        5      - change `5' partition
        6      - change `6' partition
        7      - change `7' partition
        select - select a predefined table
        modify - modify a predefined partition table
        name   - name the current table
        print  - display the current table
        label  - write partition map and label to the disk
        ! - execute , then return
        quit
partition>
```

The partition list can be printed by the "_print_" command:

```
partition> print
Current partition table (original):
Total disk cylinders available: 13051 + 2 (reserved cylinders)

Part      Tag    Flag     Cylinders         Size            Blocks
  0 unassigned    wm       0                0         (0/0/0)             0
  1 unassigned    wm       0                0         (0/0/0)             0
  2     backup    wu       0 - 13050       99.98GB    (13051/0/0) 209664315
  3 unassigned    wm       0                0         (0/0/0)             0
  4 unassigned    wm       0                0         (0/0/0)             0
  5 unassigned    wm       0                0         (0/0/0)             0
  6 unassigned    wm       0                0         (0/0/0)             0
  7 unassigned    wm       0                0         (0/0/0)             0
  8       boot    wu       0 -     0        7.84MB    (1/0/0)         16065
  9 alternates    wm       1 -     2       15.69MB    (2/0/0)         32130

partition>
```

Enter "_0_" to edit the first partition:

```
partition> 0
Part      Tag    Flag     Cylinders         Size            Blocks
  0 unassigned    wm       0                0         (0/0/0)             0

Enter partition id tag[unassigned]:
```

- The "_partition id tag_" and "_flags_" can be left as is (just press '_Enter_').
- The "_starting cylinder_" will have the value of "_3_" (see in the print output above - there are two partitions "_boot_" and "_alternates_" taking together 3 cylinders, therefore we will use 3 so that the new partition is put after those two).
- The "_partition size_" needs to be calculated as the total number of cylinders (according to the previous "_print_" output - "_13050_" in the above case) minus the star cylinder (_3_), so it will be "_13047c_" (the '_c_' at the end denotes the number specifying the cylinder count).

```
Enter partition id tag[unassigned]:
Enter partition permission flags[wm]:
Enter new starting cyl[0]: 3
Enter partition size[0b, 0c, 3e, 0.00mb, 0.00gb]: 13047c
partition>
```

The status can be checked by "_print_" again:

```
partition> print
Current partition table (unnamed):
Total disk cylinders available: 13051 + 2 (reserved cylinders)

Part      Tag    Flag     Cylinders         Size            Blocks
  0 unassigned    wm       3 - 13049       99.95GB    (13047/0/0) 209600055
  1 unassigned    wm       0                0         (0/0/0)             0
  2     backup    wu       0 - 13050       99.98GB    (13051/0/0) 209664315
  3 unassigned    wm       0                0         (0/0/0)             0
  4 unassigned    wm       0                0         (0/0/0)             0
  5 unassigned    wm       0                0         (0/0/0)             0
  6 unassigned    wm       0                0         (0/0/0)             0
  7 unassigned    wm       0                0         (0/0/0)             0
  8       boot    wu       0 -     0        7.84MB    (1/0/0)         16065
  9 alternates    wm       1 -     2       15.69MB    (2/0/0)         32130

partition>
```

The partition map and label is written back to the disk by the "_label_" command:

```
partition> label
Ready to label disk, continue? y

partition>
```

Exit by using the "_q_" command (twice - exit the "_partition_" and "_format_"):

```
partition> q
format> q
#
```

**b) Formatting the partition to ZFS and adding to the disk pool**

This is done by using the "_zpool_" command:

```
# zpool create -f srv c0d1s0
```

In the above, the "_c0d1s0_" is the first partition ("_s0_") of the secondary disk ("_d1_") on the first controller ("_c0_"). The "`-f`" parameter specifies the mount point of the new pool ("_/srv_" in this case).

The successful creation of the pool can be checked by the "`df`" command:

```
# df -h
Filesystem             size   used  avail capacity  Mounted on
rpool/ROOT/s10x_u11wos_24a
                        20G   3.7G    14G    22%    /
/devices                 0K     0K     0K     0%    /devices
...
srv                     98G    31K    98G     1%    /srv
```

In case of a mistake, the pool can be removed by:

```
# zpool destroy srv
```

(where "_srv_" is the name of the pool created before)

### 5.3. Jenkins node setup

Creating the special user for the Jenkins node (Jenkis agent can run under any user, but it is generally recommended to have a separate user running the Jenkins agents):

```
# useradd -m -d /srv/jenkins -s /usr/bin/bash jenkins
# passwd jenkins
New Password:
Re-enter new Password:
passwd: password successfully changed for jenkins
```

Note the setting of the home directory for the Jenkins user, which points to the "_/srv/jenkins_" on a separate data drive - the Jenkins workspace data will be kept there.

The most convenient option is to make the Jenkins node a SSH agent, which is maintained by the Jenkins master automatically.
For it to work, the Jenkins master SSH key needs to be inserted to the Jenkins node user "*~/.ssh/authorized_keys*" file.

On the Jenkins master:

```
[jenkins@master]$ cat ~/.ssh/id_rsa.pub | ssh jenkins@test-sol1 'mkdir -p .ssh; cat >> .ssh/authorized_keys'
```

Resetting the file permissions on the Solaris node:

```
# chmod 700 /srv/jenkins/.ssh
# chmod 600 /srv/jenkins/.ssh/authorized_keys
```

With this setup the Solaris machine should be prepared to be used as SSH Jenkins agent.

### 5.4. Additional development tools

When using the Solaris machine for C/C++ development (e.g. as a Jenkins node), some additional tools are usually needed (_Git_, _CMake_, _SVN_ etc.).
Many of those are provided by the [OpenCSW repository](#51-csw-repository).

**a) Installing Git**

```
# pkgutil -y -i git
Solving needed dependencies ...
Solving dependency order ...
6 CURRENT packages:
        CSWcommon-1.5,REV=2010.12.11
        CSWggettext-data-0.19.8,REV=2016.09.08
        CSWlibgcc-s1-5.2.0,REV=2015.07.31
        CSWlibiconv2-1.14,REV=2011.08.07
        CSWterminfo-6.0,REV=2016.04.01
        CSWterminfo-rxvt-unicode-9.20,REV=2014.10.31
Install 39 NEW packages:
        CSWbash-4.3.33,REV=2015.02.15 (opencsw/testing)
...
Installation of <CSWgit> was successful.
# which git
/opt/csw/bin/git
# git --version
git version 2.3.1
```

**b) Installing CMake**

```
# pkgutil -y -i cmake
Solving needed dependencies ...
Solving dependency order ...
17 CURRENT packages:
        CSWcacertificates-20160830,REV=2016.08.30
        CSWcas-migrateconf-1.50,REV=2015.01.17
...
Installation of <CSWcmake> was successful.
# which cmake
/opt/csw/bin/cmake
# cmake --version
cmake version 3.4.3

CMake suite maintained and supported by Kitware (kitware.com/cmake).
```

**c) Installing Subversion (SVN)**

```
# pkgutil -y -i subversion
Solving needed dependencies ...
Solving dependency order ...
17 CURRENT packages:
        CSWbdb48-4.8.30,REV=2010.12.06_rev=p0
        CSWcommon-1.5,REV=2010.12.11
        CSWggettext-data-0.19.8,REV=2016.09.08
...
Installation of <CSWsubversion> was successful.
# which svn
/opt/csw/bin/svn
# svn --version
svn, version 1.9.4 (r1740329)
   compiled Sep 19 2016, 14:50:04 on i386-pc-solaris2.10
```

**d) Installing GCC v.5**

```
# pkgutil -y -i gcc5g++
Solving needed dependencies ...
Solving dependency order ...
11 CURRENT packages:
        CSWcas-texinfo-1.50,REV=2015.01.17
        CSWcommon-1.5,REV=2010.12.11
...
Installation of <CSWgcc5g++> was successful.
# which g++
/opt/csw/bin/g++
# g++ --version
g++ (GCC) 5.2.0
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
```

(similar for GCC v.4)

{% include abbrev domain="computers" %}
