---
title: "Disk health monitoring and reporting"
header:
  teaser: /assets/images/posts/nas-ares/system-monitor.png
toc: true
categories:
  - Operating Systems
  - Linux
tags:
  - nas
  - server
  - os
  - linux
  - health
  - monitoring
  - reporting
  - smart
---

![NAS OS](/assets/images/posts/nas-ares/system-monitor.png){: .align-center .img-large}

Monitoring and reporting of disk health issues under Linux: SMART errors, temperatures, RAID failures.

<!-- more -->

## 1. Motivation

Hard disks are susceptible to failures, therefore it is wise to monitor their status regularly to prevent eventual data losses.
This is especially important for server-like machines that run "on their own", like disk stations and NAS boxes (for example the "[Project Ares]({% post_url 2020-07-13-project-ares-part-1-choosing-the-hardware %})" I was building recently).

But this can be beneficial for desktop machines as well, to be warned about imminent disk failures before it is too late.

In particular, most modern disks offer the so-called "[S.M.A.R.T.](https://en.wikipedia.org/wiki/S.M.A.R.T.)" technology that allows checking various reliability indicators, like for example:
- disk temperatures
- uncorrectable/remapped sectors
- designed life period exceeded indicators
- short and long self-tests

These can be generally monitored and the failures eventually reported.

{% capture notice_contents %}
**<a name="ubuntu-note">Warning</a>**:

Note that the instructions here are based on the **Ubuntu distribution** and might not work for other distributions where I didn't test them.
{% endcapture %}

{% include notice level="warning" %}

All the commands assume the root console:

```bash
sudo -i
```

- alternatively you'd need to prepend "sudo" to every command

## 2. Measuring temperatures: hddtemp

The "hddtemp" utility can be used to print the disk temperatures.

It can be also used to setup some disk parameters, but I'll be using a different package for that (see later).

- installing the package:

```bash
apt install -y hddtemp
```

- checking the disk temperatures (note that the letter list can be used for easier specification of multiple disks at once):

```bash
hddtemp /dev/sd[abcdefgh]

# Example output:
/dev/sda: WDC WD80EZAZ-11TDBA0: 39°C
/dev/sdb: SAMSUNG HD103SI: 31°C
/dev/sdc: WDC WD10JPVX-08JC3T5: 36°C
/dev/sdd: WDC WD80EZAZ-11TDBA0: 41°C
/dev/sde: WDC WD80EZAZ-11TDBA0: 41°C
/dev/sdf: WDC WD20EURS-73S48Y0: 40°C
/dev/sdg: SAMSUNG HD203WI: 35°C
/dev/sdh: WDC WD80EZAZ-11TDBA0: 43°C
```

## 3. Health monitoring and reporting: smartmontools

This part shows the setup of disk SMART status checking and eventual reporting to e-mail address by using the "smartmontools" package.

The prerequisite for reporting to an external e-mail is the Postfix setup to forward the notification e-mails normally delivered to the local root account:
[E-mail delivery with postfix SMTP relay]({% post_url 2020-10-26-linux-postfix-smtp-relay %})

### 3.1. Installing the package

```bash
apt install -y smartmontools

vim /etc/default/smartmontools

# uncomment the line
# (might not be necessary if the line is missing)
start_smartd=yes
```

### 3.2. Setting up the SMART monitoring and reporting

The daemon configuration can be setup in the "_/etc/smartd.conf_" file:

```bash
vim /etc/smartd.conf
```

The options and examples are available in the [smartd.conf manual](https://linux.die.net/man/5/smartd.conf)

A simple configuration can look like this:

```
DEVICESCAN -d removable -n standby -W 0,40,45 -H -l error -l selftest -f -s (L/../(01|15)/./11|S/../../6/17) -m root -M diminishing -M exec /usr/share/smartmontools/smartd-runner
```

{% capture notice_contents %}
**<a name="smartd-params">The parameters</a>**:

In the above example the following setup is done:
- **DEVICESCAN**: apply to all devices that were not configured before
- **-d removable**: ignore the device if missing (instead of exiting)
- **-n standby**: check the device unless it is in sleep or standby mode (do not spin the disk up for the SMART status polling)
- **-W 0,40,45**: report INFO message if the disk temperature reached 40 degrees Celsius and CRIT if reached 45 (and send the notification mail as configured)
- **-H -l error -l selftest**: report SMART health status failures - SMART errors and failed self-tests
- **-f**: check for Usage Attribute failures - does not indicate an imminent disk failure, but advisory condition ("device exceeded its design life period")
- **-s (L/../(01\|15)/./11\|S/../../6/17)**: run self tests (format "T/MM/DD/d/HH"):
  - run complete extended ("long") self-test on every 1st and 15th day of the month at 11am
  - run short self-test on every Saturday at 5pm
- **-m root**: send notification e-mails to the "root" account
- **-M diminishing**: when sending the repeated e-mails for the same failure, increase the notification interval every time
{% endcapture %}

{% include notice level="info" %}

Additional useful parameters:
- **-M test**: add temporarily to test the e-mailing setup - the smartd will send a test e-mail every time it is started (remove after veryfying that the e-mails are received properly)
- **-m &lt;your-public-mail-address&gt;**: Send to the other e-mail address directly (if not having Postfix forwarding set up for the "root")

The above is a relatively simple configuration that generally works, but has some issues:
- the disk self-tests are run for **all disks at the same time**
- this might be an issue if the disks are too close to each other, where this can eventually **raise the disk temperatures** over an acceptable level (especially with the long self-tests that run for multiple hours)

Therefore I did a more complex setup in case of the "Project Ares" NAS box, in particular the self tests timing:

```
DEFAULT -d removable -n standby -W 0,40,45 -H -l error -l selftest -f -m root -M diminishing -M exec /usr/share/smartmontools/smartd-runner
/dev/sda -s (L/../(01|16)/./11|S/../../6/15)
/dev/sdb -s (L/../(04|19)/./11|S/../../6/17)
/dev/sdc -s (L/../(07|22)/./11|S/../../7/15)
/dev/sdd -s (L/../(10|25)/./11|S/../../7/17)
/dev/sdf -s (L/../(01|16)/./11|S/../../6/15)
/dev/sdg -s (L/../(04|19)/./11|S/../../6/17)
/dev/sdh -s (L/../(07|22)/./11|S/../../7/15)
/dev/sdi -s (L/../(10|25)/./11|S/../../7/17)
DEVICESCAN -s (L/../(13|28)/./11|S/../../6/19)
```

{% capture notice_contents %}
**<a name="smartd-setup">The improved parameters</a>**:

- **DEFAULT**: sets the defaults for the following lines
- **/dev/sd# lines**: self-test setup for each disk
- **DEVICESCAN ...**: all the remaining disks not mentioned explicitly

In particular the setup is done that the **disks in close proximity do not run the test at the same time** to avoid the overheating.

Note that the "-d removable" in particular applies here so that if the disk is removed (or failed completely) the smartd will not be stuck.

The last "DEVICESCAN" is just to cover all the remaining drives.
{% endcapture %}

{% include notice level="info" %}

After any configuration change the daemon needs to be restarted:

```bash
systemctl restart smartd
```

### 3.3. Example reports

- example of "disk temperature too high" report:

```
This message was generated by the smartd daemon running on:

   host name:  fs-ares
   DNS domain: [Empty]

The following warning/error was logged by the smartd daemon:

Device: /dev/sdf [SAT], Temperature 45 Celsius reached critical limit of 45 Celsius (Min/Max 32/50)

Device info:
WDC WD80EZAZ-11TDBA0, S/N:xxxxxxxx, WWN:x-xxxxxx-xxxxxxxxx, FW:83.H0A83, 8.00 TB
```

### 3.4. Troubleshooting

- you can eventually check the syslog to see the smartd messages:

```bash
tail -f /var/log/syslog

# check just the smartd messages
grep "smartd" /var/log/syslog* | less
```

- as mentioned before, you can use the "-M test" to force sending the notification for testing that it works

## 4. RAID notifications

By default the MD-RAID and ZFS RAID deliver any notifications to the "root" user.

Therefore, I'd strongly encourage to **configure the postfix alias for root** if a notification to an external e-mail address is needed, as described here:
[E-mail delivery with postfix SMTP relay]({% post_url 2020-10-26-linux-postfix-smtp-relay %})  
In that case you'll have nothing else to do.

It is also **possible to configure the external e-mail address** if needed (but note that the postfix SMTP setup is needed anyway in most cases, so that the e-mails will not be rejected as spam at the receiving side, at least for the public providers like GMail, Yahoo, Hotmail etc.).

### 4.1. MD-RAID e-mail configuration

```bash
vim /etc/mdadm/mdadm.conf

# update to the preferred external address
# (keep "root" if postfix alias configured)
MAILADDR <your-email-address>
```

### 4.2. ZFS notifications configuration

```bash
vim /etc/zfs/zed.d/zed.rc

# update to the preferred external address
# (keep "root" if postfix alias configured)
ZED_EMAIL_ADDR="<your-email-address>"
```

### 4.3. Example reports

- example of ZFS resilver report:

```
ZFS has finished a resilver:

   eid: 56
 class: resilver_finish
  host: fs-ares
  time: 2020-10-25 00:32:35+0200
  pool: rpool
 state: ONLINE
  scan: resilvered 3.05G in 0 days 00:01:58 with 0 errors on Sun Oct 25 00:32:35 2020
config:

    NAME                                      STATE     READ WRITE CKSUM
    rpool                                     ONLINE       0     0     0
      mirror-0                                ONLINE       0     0     0
        nvme-TS512GMTE220S_F725141376-part4   ONLINE       0     0     0
        scsi-SATA_ST9750420AS_6WS276V6-part4  ONLINE       0     0     0

errors: No known data errors
```

## 5. Other monitoring tools

There are also other more "enterprisey" tools not covered in this article.
These tools often use HTTP interface to access the monitored data and can eventually be used to monitor multiple machines at once, also can usually monitor much more system indicators than just the disks.
On the other side, they usually require more complex setup, some of them are also paid.

Some examples:
- [Nagios](https://www.nagios.org/)
- [Monit](https://mmonit.com/)
- and many others

## Resources and references

- [Wikipedia: S.M.A.R.T.](https://en.wikipedia.org/wiki/S.M.A.R.T.)
- [Ubuntu: Running Smartmontools as a Daemon](https://help.ubuntu.com/community/Smartmontools#Advanced:_Running_as_Smartmontools_as_a_Daemon)
- [man: smartd.conf](https://linux.die.net/man/5/smartd.conf)

{% include abbrev domain="computers" %}
