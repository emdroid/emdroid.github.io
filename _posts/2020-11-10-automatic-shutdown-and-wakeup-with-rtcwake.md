---
title: "Automatic shutdown and wake-up with rtcwake"
header:
  teaser: /assets/images/posts/clock-pc.jpg.png
toc: true
categories:
  - Operating Systems
  - Linux
tags:
  - os
  - linux
  - power management
  - scheduling
  - rtcwake
  - cron
---

![NAS OS](/assets/images/posts/clock-pc.jpg){: .align-center .img-large}

How to make a Linux PC shutdown and wake-up automatically at specific times.

<!-- more -->

## 1. Motivation

In some situations it is useful to suspend and resume a computer **according to a schedule**.

This is for example convenient for home setups like the "[Project Ares]({% post_url 2020-07-13-project-ares-part-1-choosing-the-hardware %})" NAS box I was building recently, in order to save the electricity and shut down the computer during the night when it would normally not be used (or during the week days when at work).

There are more possibilities to achieve that, but the most common way is the [rtcwake](https://linux.die.net/man/8/rtcwake) utility that allows to put the computer into sleep/shutdown state and schedule the wake-up timer to a later point.

{% capture notice_contents %}
**<a name="ubuntu-note">Warning</a>**:

Note that the instructions here are based on the **Ubuntu distribution** and might not work for other distributions where I didn't test them.
{% endcapture %}

{% include notice level="warning" %}

## 2. Basic rtcwake usage

The following examples can be used to test whether the "rtcwake" works correctly on a particular system:

- enter the root console (otherwise you'll need to add "sudo" in front of each command):

```bash
sudo -i
```

- list the available modes:

```bash
rtcwake --list-modes

# output example:
freeze mem disk off no on disable show
```

- enter the "standby" mode and wake-up back in 2 minutes:

```bash
rtcwake -m mem -s 120
```

- sleep to disk ("hibernate") and wake up back in 2 minutes (see below for the hibernation details):

```bash
rtcwake -m disk -s 120
```

- shut down completely and wake up in 2 minutes:

```bash
rtcwake -m off -s 120
```

The modes are described in the [rtcwake manual](https://linux.die.net/man/8/rtcwake).
The important ones are:
- **standby**: First "sleep" state (S1). Fastest suspend and wake up, but doesn't survive the AC power failure.
- **mem**: Suspend to RAM (S3). Fast suspend and wake up, doesn't survive the AC power failure.
- **disk**: Suspend to disk (S4 - "hibernation"). Slower suspend and wake up, but able to survive the AC power failure.
- **off**: Power off (S5). Slowest (complete shutdown and boot up), survives the AC power failure.

### 2.1. Waking up at specific time

The "rtcwake" command also allows to setup the specific absolute time to wake up via the "-t" parameter.
This parameter takes the "time_t" value, i.e. the number of seconds since the "epoch" (1970/01/01 00:00:00 UTC).

This can be combined with the "date" command to specify the absolute time/date:

```bash
# shutdown and schedule the wake up at 6pm today
rtcwake -m off -t `date -d "18:00" +%s`

# shutdown and schedule the wake up at 10am tomorrow
rtcwake -m off -t `date -d "10:00 +1day" +%s`
```

This is convenient, as it can be used to schedule the wake-up at specific time without having to calculate the time differences in seconds.

Another convenient "date" options are for example:

```bash
date -d "today 14:00"
date -d "tomorrow 11:00"
date -d "15:00 next Fri"
# etc.
```

## 3. Hibernation and hybrid mode

In order to use the "suspend to disk" (S4) mode, the hibernation must be working first.

See here for the hibernation setup details: [Linux hibernation setup (ZFS, encryption)]({% post_url 2020-11-08-linux-hibernation-zfs-encryption %})

**Just to recap**:
- the swap needs to be enabled and larger than the size of the RAM  
  (so that the RAM contents can fit in there)
- needs to be available early during the boot-up phase  
  (so for example if encrypted, it requires the initramfs setup to be decrypted before resuming the machine)

As described there, the "pm-utils" can be used to test and manage the hibernation.

### 3.1. The "suspend-hybrid" mode

The Linux system (and pm-utils) supports the "suspend-hybrid" mode, which is particularly interesting.

It is a **combination of both** the "suspend" and "hibernate" modes, i.e. writing the RAM contents to the disk (swap) for the hibernation and then entering the suspend-to-RAM mode.

This means that the **wakeup is fast in the usual case** when the system status is **restored from the RAM** quickly.
However it can **survive the AC power loss**, because it is backed up by the hibernation (takes the action in the case of power being lost).

As you can notice, this hybrid mode is not supported by the "rtcwake" directly.
But it can still be used by **combining the "rtcwake" and the "pm-utils"**:
- using "rtcwake" with the "-m no" mode (just setting up the wake up timer, but not suspending the computer)
- then using the "pm-suspend-hybrid" to enter the hybrid state

For example, entering the "suspend-hybrid" mode and waking-up back in 2 mins:

```bash
rtcwake -m no -s 120 && pm-suspend-hybrid
```

## 4. Automatic power management scheduling

### 4.1. Using crontab to schedule the suspend / resume

The whole purpose of this exercise is to **schedule regular suspend / resume timers** to switch the machine off when nobody would normally use it, like for example during the night or when at work during the week.

This can be done by using the crontab to schedule the automatic rtcwake suspends.

It is necessary to do this under the admin account (i.e. using "sudo -i" or prepending "sudo"), as otherwise the commands would be running under the regular user, who might not have the sufficient permissions to do that.

```bash
crontab -e
```

There we can setup the rtcwake schedule, for example (you can change the mode from "disk" to any other mode you'd like to use):

```bash
# setup the power management schedule

# weekdays: suspend at 2am and wake up at 6pm
0  2 * * 1-5   /usr/sbin/rtcwake -m disk -t `date -d "18:00" +\%s` >> /var/log/suspend-resume.log
# weekdays: if still running, suspend at 10am and wake up at 6pm
0 10 * * 1-5   /usr/sbin/rtcwake -m disk -t `date -d "18:00" +\%s` >> /var/log/suspend-resume.log
# weekends: suspend at 3am and wake up at 9am
0  3 * * 6-7   /usr/sbin/rtcwake -m disk -t `date -d "9:00" +\%s` >> /var/log/suspend-resume.log
```

The same schedule using the "suspend-hybrid" mode (which is not supported directly by the rtcwake, thus using the pm-utils):

```bash
# setup the power management schedule

# weekdays: suspend at 2am and wake up at 6pm
0  2 * * 1-5   /usr/sbin/rtcwake -m no -t `date -d "18:00" +\%s` >> /var/log/suspend-resume.log && /usr/sbin/pm-suspend-hybrid
# weekdays: if still running, suspend at 10am and wake up at 6pm
0 10 * * 1-5   /usr/sbin/rtcwake -m no -t `date -d "18:00" +\%s` >> /var/log/suspend-resume.log && /usr/sbin/pm-suspend-hybrid
# weekends: suspend at 3am and wake up at 9am
0  3 * * 6-7   /usr/sbin/rtcwake -m no -t `date -d "9:00" +\%s` >> /var/log/suspend-resume.log && /usr/sbin/pm-suspend-hybrid
```

- to disable any schedule temporarily the particular line can just be commented out
- it is also possible to create a script that would check some environment setting (e.g. a file in /etc) to disable the schedule temporarily

### 4.2. Error checking and reporting

Note that the cron notifications are sent to the local (root) user by default.
If you want to receive any cron failure notifications to an external e-mail, it can be configured by using the "MAILTO" crontab setting:

```bash
crontab -e

# set the cron mail address
MAILTO=<your-email-address>
```
Any eventual errors can be also checked in the "_/var/log/syslog_" file, for example by:

```bash
tail -f /var/log/syslog
```

As you could notice, the normal logging is directed to the "_/var/log/suspend-resume.log_" file, which can then also be checked:

```bash
less /var/log/suspend-resume.log
```

## 5. Resuming after the power failure

There are 2 basic issues when the AC power is lost:

**I. Some of the suspend modes lose the data after the power failure**:
- generally the suspend modes that keep the state in RAM only (sleep, suspend to RAM) cannot be recovered after the power failure
- the modes that store the data to disk (hibernation, suspend-hybrid, shutdown) are safe to not lose any data in case of power loss
- so for longer shutdowns (e.g. overnight) it is recommended to use the hibernation (or complete shutdown) mode
- as described before, if you want a fast (but still safe) suspend for a prolonged period of time, it is recommended to use the **suspend-hybrid mode** (fast resume from RAM in the usual case, but able to restore from the disk in the case of power loss)

**II. The RTD wakeup timer is lost after the power loss**:

The other issue after losing the AC power is that the RTC timer will (usually) no longer be able to wake up the computer.
It does use the motherboard capabilities to some extent, but it doesn't setup the wake up alarm in the BIOS so that it only works as long as the part responsible for waking up is still under power.

### 5.1 Setting up the BIOS for resuming

There are 2 basic options to setup the BIOS to work-around the AC power loss issue (most newer BIOS implementations can use both options):
- start the computer immediately after the power is restored
- configure the APM timer to wake up regularly at a particular time

Both options require to perform changes in the BIOS, which is usually entered by pressing the "Del" or "F2" keys during the startup.

Note that in order to do that, a **display and keyboard are needed** (even though the server might be used without them otherwise).
The settings can be usually found somewhere under "Advanded Options", APM, ACPI, Wake Up / Power Management configuration etc.

In my case this is for example (UEFI BIOS):
- Advanced Mode (F7)
- Advanced tab
- APM Configuration

**a) Restore state after the power loss**:

The setting is usually something like **Restore Power On AC Loss**, in most cases these options are available:
- _Power Off_: stay off after the AC restore (usually the default)
- _Power On_: switch the computer on after the power is restored
- _Last State_: switch the computer on if it was on before, otherwise keep it off

> **Last State** option:
> - **recommended if available**
> - will ensure that the machine will **start again in the case the power was lost while the computer was running**
> - see the note below on restoring the RTC wake-up timer

> **Power On** option:
> - this way the computer will be **always started** when the power is lost and restored, thus ensuring it will not miss the schedule
> - the downside is that it will be eventually started at the time where it normally wouldn't
> - if waking up multiple computers like this, an **excess AC network spike** can happen (when all the computers are being switched on at once after the AC power is restored)
> - that might eventually trigger the circuit breakers or fuses thus shutting the AC power down again

**b) Fallback wake-up timer**:

Most BIOS implementations, especially the newer ones (including the UEFI) usually allow to set a **wakeup timer for a particular time of day**.
This can be used to ensure the machine is always woken up at last at this time.

For example, if you setup your schedule to wake up the machine at 6pm on the week days via the rtcwake, you can set up the BIOS wake up timer to 6:30pm in addition to ensure the machine will be booted up if the scheduled RTC timer is missed for whatever reason.

This in particular has the most sense **when using the "Last State"** AC power failure restore option, where the RTC timer could be lost eventually.
See also bellow for additional notes.

This can be usually found under "Wake Up On Time", "Resume By Alarm", "Power On By RTC" (on my UEFI server machine) etc.

There you can setup the wake-up day (date of month, 0 usually meaning "every day" - read the BIOS help) and the time (hour:min:sec) options.

Be careful in particular about your **UTC vs Local time** system setup (see later), as the UTC can lead to some confusion when setting up the timer.

### 5.2. Restoring the RTC timer

In my particular case (on my specific machine) I found out that with these options the scheduled **RTC timer is fully restored**:
- **Restore Power On AC Loss**: _Last State_
- **Power On By RTC**: _Enabled and time set_

This means that after the AC power restore the machine will not switch on immediately, but **correctly according to the scheduled time** (as it was planned in the system, i.e. not just on the time configured in the BIOS).

Note that this doesn't work (on my machine) when the BIOS RTC timer is not enabled and set (with just the "Last State" option) - **both options need to be set**.

This could be BIOS implementation defined behavior, so you'd need to test this with your PC:
- run the rtcwake schedule (disk or suspend-hybrid) with +5 mins to wake up
- in between those 5 mins, disconnect the AC power cable, wait for couple of seconds and plug it back
- wait to see whether the machine will wake up at the scheduled time
- recommended to suspend to disk or hybrid (suspending just to RAM not recommended - the machine might still come up but the RAM contents will be gone)

According to my experience this **does not work when using the "off" mode** (shutdown) - in that case the RTC timer is not restored and thus not woken up at the scheduled time, at least on my machine.

### 5.3. UTC vs local BIOS time

It is usually recommended for Linux to set the system clock to use the UTC BIOS time, except for dual-boot machines (Linux + Windows) where the local time is encouraged.

However when you'll be setting up the wake-up timer in the BIOS, I would recommend to **use the local time** because of the following reasons:
- **more convenient for setting up the timer** (don't need to re-calculate the current local time to UTC)
- **UTC doesn't play well with DST changes**:
  - you usually want to wake up the machine at the same local time of day
  - that means that you'll need to change the BIOS wake-up timer on every DST change back and forth in case the time is in UTC
  - unless your country doesn't do DST time switches (but still would need to recalculate the local time to UTC)
  - note that you could still have the DST issue in case the machine is scheduled to sleep during the DST change (the BIOS current time will not be updated then)

Checking the current UTC / local time status in Ubuntu:

```bash
timedatectl | grep local
```

Changing the current setup:

```bash
# use the BIOS local time
timedatectl --adjust-system-clock set-local-rtc 1

# use the BIOS UTC time
timedatectl --adjust-system-clock set-local-rtc 0
```

Note that the "--adjust-system-clock" option can be used to adjust the time immediately.

## 6. Summary

1. Using the rtcwake, cron and eventually pm-utils to schedule the suspend/resume.
2. Use "suspend-hybrid" (pm-utils) whenever possible:
   - is fast to restore, but survives AC power loss
   - works well with the "Last State" AC power loss option
3. BIOS configuration:
   - use the "Last State" AC power loss option
   - configure the "Wake-up Timer"
   - these settings might allow to restore the RTC timer after the AC power restored
4. Use local BIOS time:
   - easier to use for the BIOS timers
   - avoids DST switch issues (mostly)

## Resources and references

- [man: rtcwake](https://linux.die.net/man/8/rtcwake)
- [Ubuntu: Sleep & Wakeup Schedule](https://askubuntu.com/questions/1009684/sleep-wakeup-schedule-ubuntu-16-04-3-lts)
- [Ubuntu: Bios Clock UTC setup](https://askubuntu.com/questions/720445/ubuntu-15-10-assumes-bios-clock-is-set-to-utc-time-regardless-of-utc-no-in-etc)

{% include abbrev domain="computers" %}
