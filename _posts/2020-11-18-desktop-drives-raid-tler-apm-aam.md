---
title: "Using desktop drives in RAID: TLER (ERC), APM, AAM"
header:
  teaser: /assets/images/posts/hdd.jpg
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
  - raid
  - tler
  - apm
  - aam
---

![HDD](/assets/images/posts/hdd.jpg){: .align-center .img-large}

Desktop vs NAS-dedicated hard drives differences, specifics of using standard desktop drives in the RAID setup.

<!-- more -->

{% capture notice_contents %}
**<a name="ubuntu-note">Warning</a>**:

Note that the instructions in the article are based on the **Ubuntu distribution** and might not work for other distributions where they weren't tested.
{% endcapture %}

{% include notice level="warning" %}

## 1. Introduction

Most of the PC users are using the "standard" desktop hard disks.
But there are other different kinds of disks for specific usage scenarios:

1. **Desktop drives**:
   - intended for standard personal computer usage
   - most sold, standard parameters, sometimes very aggressive power management (spin down timeouts)
   - e.g. WD Blue/Green/Black, Seagate/Samsung standard drives etc.

2. **NAS dedicated disks**:
   - for use in server machines like NAS, SAN etc., and in RAID in particular
   - usually increased MTBF (designed for 24/7 use)
   - often limited error recovery (see below) and less aggressive power management
   - examples are WD Red, Seagate IronWolf etc.

3. **Disks for other special purposes**:
   - e.g. WD Purple for surveillance video storage (optimized for fast constant writes, but somewhat slower read speeds)

Generally it is **recommended to use the NAS drives** for storage servers and RAID in particular:
- more resilient, designed to run 24/7
- [Error Recovery Control](https://en.wikipedia.org/wiki/Error_recovery_control) (ERC/TLER/CCTL) is important for the RAID usage (see below)
- often wider range of operating conditions (can withstand higher temperatures)

However the NAS drives are usually significantly more expensive than the desktop drives, therefore the **home users often choose the standard desktop HDD&zwj;s even for the NAS and RAID usage** (like I did in my "[Project Ares]({% post_url 2020-07-13-project-ares-part-1-choosing-the-hardware %})" NAS build) or even a [combination of various drives and capacities]({% post_url 2020-10-31-zfs-raid-over-different-drives %}).

It is then important to **understand the implications** of such decision, as it can have some unwanted effects:
- do not have the ERC enabled by default, this can cause issues in RAID
- some desktop drives have very aggressive spin-down timing (for example the WD Green drives), that can lead to increased wear and thus lifetime shortening
- more sensitive to high temperatures

## 2. Error Recovery Control (ERC/TLER/CCTL etc.)

If the disks are used in the **RAID setup**, it is particularly important to utilize the so-called [Error Recovery Control](https://en.wikipedia.org/wiki/Error_recovery_control) (ERC/TLER/CCTL).

If not supported or enabled, the drive might try to recover any read errors for **extended amount of time** (assumes no redundancy).
But in a redundant disk setup (RAID with parity/mirror) this is **neither wanted nor necessary** for multiple reasons:
- the **data can be recovered by other means** than trying to recover the damaged sectors (having the parity or mirror redundancy)
- if the disk doesn't respond for a long time (trying to recover the data on its own) the RAID controller (HW or SW) might **mark the disk "failed" and drop it from the array** (this can happens very quickly, with some controllers as soon as after 8 secs)

On the other side, if your NAS **wouldn't be using RAID** and just having single drives without a backup (not recommended), the NAS **drives with ERC might eventually be a worse choice** than using the desktop drives (for the same reason, as in such case you'd actually want the disk to try to recover the read errors a bit harder).

Note that the commands below usually require the root access, which can be entered by using:

```bash
sudo -i
```

- alternatively you'd need to prepend "sudo" to every command

### 2.1. Checking / setting the drive ERC status

The current ERC/TLER status can be checked by the "smartctl" command:

```bash
smartctl -l scterc /dev/sda

# sample output (TLER disabled)
SCT Error Recovery Control:
           Read: Disabled
          Write: Disabled
```

When it says "Disabled" like in the above example, it means that the ERC is not enabled thus the disk might try to recover for extended time (**not suitable for RAID**).

In that case the TLER can be enabled using the same utility:

```bash
# set the TLER to 7 sec
smartctl -l scterc,70,70 /dev/sda

# output:
SCT Error Recovery Control set to:
           Read:     70 (7.0 seconds)
          Write:     70 (7.0 seconds)
```

{% capture notice_contents %}
**<a name="tler-warn">TLER value</a>**:

Note that it is **not recommended to make the TLER value too small** either - if too short, the disk might not recover some easily recoverable issues, thus increasing the remapped sector count unnecessarily.
{% endcapture %}

{% include notice level="warning" %}

### 2.2. The disk init script

The issue is that in most cases the TLER value is **not preserved after reboot**, therefore it needs to re-done after every restart of the machine.

This can be done by an init script to setup the TLER on startup:

```bash
# list the IDs
ls -al /dev/disk/by-id/

# list the models that you want to setup
# for example:
DISKS_TLER="SATA_WDC_WD80EZAZ SATA_ST9750420AS"
# (use just the common parts for multiple drives)
```

- the init script:

```bash
  cat <<EOF > /usr/local/bin/init-disks
#!/bin/sh
# Initialize the disks

DISKS_TLER="${DISKS_TLER}"

enable_tler_disk() {
    DEVICE="\$1"
    if [ -b "\${DEVICE}" ]; then
        # ls -al "\${DEVICE}"
        $(which smartctl) -q silent -l scterc,70,70 "\${DEVICE}"
    fi
}

enable_tler() {
    # iterate through the disk models
    for disk in \${DISKS_TLER}
    do
        # echo "disk: \${disk}"
        ls /dev/disk/by-id | grep "\${disk}" | grep -v -- "-part" | while read disk_id
        do
            # echo "disk_id: \${disk_id}"
            enable_tler_disk /dev/disk/by-id/\${disk_id}
        done
    done
}

enable_tler
EOF
```

- setup the permissions:

```bash
chmod +x /usr/local/bin/init-disks
```

- to change the disk setup later use the "vim" or any other favorite editor of yours:

```bash
vim /usr/local/bin/init-disks
```

- testing the script:

```bash
/usr/local/bin/init-disks
```

Now we can check the status again to see if it worked (use with a drive you didn't already setup before):

```bash
smartctl -l scterc /dev/sda
```

### 2.3. Setting up the crontab

After verified that it works, the script can be added to the crontab to run after every reboot:

```bash
crontab -e

# insert:
@reboot /usr/local/bin/init-disks
```

- test by restarting the machine:

```bash
reboot
```

- check that the TLER is still set:

```bash
smartctl -l scterc /dev/sda
```

## 3. Other disk parameters

There are additional disk parameters that are relevant:
- **[Automatic acoustic management](https://en.wikipedia.org/wiki/Automatic_acoustic_management)** (AAM): Speed vs. noise
- **[Advanced Power Management](https://en.wikipedia.org/wiki/Advanced_Power_Management)** (APM): Spin-down time

Some desktop drives might have very **aggressive spin-down settings** (going to standby after a few seconds).
This is in particular the case for the **"green" drives** (WD Green) - those are less suitable for NAS boxes (also slower usually), but they are usually cheaper, so many people still might want to use them eventually.

If such disks are run in **RAID 24/7 configurations** without changing the settings, the frequent spin-downs can cause the power cycles to go up rapidly, **wearing the disk and shortening its lifetime** (also, if the power cycle count goes over the SMART threshold, you might experience the SMART warnings about the "life period exceeded" indicators).

### 3.1. Checking / setting the parameters

The settings can be sometimes changed by the manufacturer tools (wdidle3 etc.) or by the "hdparm" utility in Linux:

- to check the current settings:

```bash
# check the APM feature:
# (higher = more performant)
#   1-127: permit spin-down
# 128-254: disable spin-down
#     255: disable APM completely
hdparm -B /dev/sda

# check the AAM status
# (usually 0=off, 128=quiet/slowest, 254=loud/fastest)
hdparm -M /dev/sda

# check the "wdidle" disk status
# (only for WD drives)
hdparm -J /dev/sda
```

- to set use the parameter with the required value, for example:

```bash
# set for the max performance
hdparm -M 254 /dev/sda

# set the spindown timer to 30 mins
hdparm -S 241 /dev/sda
```

- these settings are normally nor permanent, but can be [set in the hdparm.conf](https://askubuntu.com/a/767616) to apply on every startup:

```bash
vim /etc/hdparm.conf
```

### 3.2. WD drives: wdidle utility

The "wdidle" support in Linux hdparm is still experimental, therefore it is recommended to use the original Western Digital provided utility: [wdidle3](https://support.wdc.com/downloads.aspx?p=113)

The utility can check and setup the idle spin down timer for some Western Digital disks.

{% capture notice_contents %}
**<a name="wdidle-warning">Warning</a>**:

Officially **only the disks mentioned on the wdidle3 page are supported**.
However I was successfully using it with some other drives not listed, in particular the WD Green series.

The utility **modifies the disk firmware - the changes are persistent**.
So there is a chance that the disk could be damaged or "bricked" (I never had any issue like that and newer heard of any, but it can't be ruled out entirely).

Also it **only works for (some) WD drives - do not try to use it for other manufacturers** (like Seagate, Samsung etc.).
Usually there is just an error message after an attempt to use it with unsupported drives, but an eventual damage can't be excluded.

**<a name="wdidle-warning-2">USE AT YOUR OWN RISK - you have been warned.</a>**
{% endcapture %}

{% include notice level="danger" %}

In my experience the "wdidle3" utility only works with the disks connected to the SATA/IDE directly (and you might even need to switch to the "ATA compatible" legacy mode in the BIOS).
In particular it **doesn't work for USB connected drives**.

It is recommended to ideally **disconnect all other drives** except the one(s) you want to setup - the utility will try to operate on all the connected drives (it is not possible to select just a specific drive - doesn't have any disk selection arguments).

The tool also requires the **pure MS-DOS environment, will not run e.g. under Windows** (the higher level OS will block the low level disk access the program is using).

Therefore it is **required to use a MS-DOS bootable CD, USB or alike** - in my case this was the most recent rare occasion I still used the 3.5" floppy disk on my old big home PC (where I still keep the floppy drive attached), as it was the fastest option to just get the job done.

The utility is also often part of some well-known bootable utility CD&zwj;s (like Hirens, UBCD etc.).

The "wdidle3" usage (note that it is operating on **all connected drives**):

```bash
# check the current status (all disks):
wdidle3 /R

# set the idle timer to 5 min (all disks):
wdidle3 /S300

# disable the idle timer entirely (not recommended):
wdidle3 /D
```

## Resources and references

- [Wikipedia: Error Recovery Control](https://en.wikipedia.org/wiki/Error_recovery_control)
- [Turn Off Error Recovery in RAID Drives: TLER, ERC, and CCTL](https://blog.fosketts.net/2017/05/30/turn-off-error-recovery-raid-drives-tler-erc-cctl/)
- [Hard Drive error recovery control](https://www.mattwall.co.uk/2016/10/16/Hard-Drive-TLER-Support.html)
- [Wikipedia: Automatic acoustic management](https://en.wikipedia.org/wiki/Automatic_acoustic_management)
- [Wikipedia: Advanced Power Management](https://en.wikipedia.org/wiki/Advanced_Power_Management)
- [Ubuntu: How can I control HDD spin down time?](https://askubuntu.com/a/767616)

{% include abbrev domain="computers" %}
