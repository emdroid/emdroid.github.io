---
title: "Auto-mounting CD/DVD drive in Ubuntu Server"
header:
  teaser: /assets/images/thumb/cd-drive.png
toc: true
categories:
  - Operating Systems
  - UNIX/Linux
tags:
  - linux
  - server
  - cdrom
  - dvd
  - automount
  - systemd
  - udev
---

{% include figure image_path="/assets/images/posts/dvd.jpg" alt="DVD" %}

CD drive is usually auto-mounted in the desktop Linux installations for the user currently logged in. However in a server installation without GUI (the so-called "*headless*" installation) the CD/DVD is not auto-mounted when inserted and special setup need to be done to allow that.

<!-- more -->

## 1. Motivation

Recently I was setting up a remote CD/DVD drive access, to be used by a media player (switched for a smaller one without a DVD-drive, and the DVD&zwj;s to be played were to be accessed remotely on a NAS-like server machine). Nowadays the DVDs are not used that much any more, however I still wanted to keep the option to play it if needed (of course there is the alternative to buy a new external DVD/BR drive, but didn't want to spend extra money when it will not be used all that often and already had a DVD drive available on the server box).

The following steps are for the Ubuntu Server (used the version 20.04 LTS), but might work for other distributions as well (besides the package manager and package names).

## 2. Setting up

### 2.1. Installing the packages

The "*udftools*" and "*autofs*" packages are needed:

```bash
sudo apt install -y udftools autofs
```

### 2.2. Adding the fstab entry

Open the "/etc/fstab" file (sudo needed):

```bash
sudo vim /etc/fstab
```

Add the following:

```
/dev/sr0        /media/cdrom    udf,iso9660 auto,exec,utf8,nofail,x-systemd.automount,x-systemd.device-timeout=2 0       0
```

Use the device name as the first parameter, it is "*/dev/sr0*" usually. If not sure, the "*lshw*" command can be used to find out:

```bash
sudo lshw -C disk
```

(look for the "*cdrom*" in the oputput)

As you can see, the DVD will be mounted using the "systemd" automount, which will take care about the automatic mounting/unmounting on insertion/removal.

In particular, there are guides mentioning the "*user,noauto*" options (the "standard" fstab setup), however that of course doesn't help with the auto-mounting - it allows the drive to be mounted by any user, but it still needs to be done manually (except when done by the GUI).

### 2.3. Test the DVD mounting

- try to insert a DVD
- test that it has been mounted (should show the CD/DVD contents):
  ```bash
  ls -al /media/cdrom
  ```
- try to eject the DVD
- test again it has been unmounted (should not show any constents)

### 2.4. Optional: Adding the Samba share

To share the device over samba (if using Windows sharing), you can simply add the CD-ROM entry to the "*smb.conf*":

```bash
sudo vim /etc/samba/smb.conf
```

Add the CD drive entry (added as read-only):

```
[dvd-rom]
        comment = DVD-ROM Drive
        path = /media/cdrom
        valid users = @users
        read only = yes
        browseable = yes
```

Restart samba to make the change effective:

```bash
sudo systemctl restart smbd
sudo systemctl status smbd
```

## 3. Troubleshooting

You can use the udev monitor to troubleshoot the device not being mounted:

```bash
sudo udevadm monitor
```

## 4. Alternatives

There are additional possibilities which I didn't explore yet:

- using **udev rules** [^1] - might be a bit more difficult to get it working, see the Ubuntu forum article (they eventually still settled with the *fstab*/*systemd* solution)
- **Network Block Device** (NBD) server

## Resources and references

[^1]: [Automount Blu-Ray drive on Ubuntu Server 16.10](https://ubuntuforums.org/showthread.php?t=2342902)

{% include abbrev domain="computers" %}
