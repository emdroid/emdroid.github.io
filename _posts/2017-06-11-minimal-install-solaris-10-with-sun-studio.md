---
title: "Minimal installation of Solaris 10 with Sun Studio"
header:
  teaser: /assets/images/thumb/sun-solaris.png
toc: true
sidebar:
  nav: "sol-10"
categories:
  - Operating Systems
  - UNIX
tags:
  - solaris
  - sun-studio
  - vm
  - minimal-install
  - csw
  - open-csw
  - jenkins
---

![Solaris](/assets/images/thumb/sun-solaris.png){: .align-left .img-thumbnail}

Minimal installation of the **Solaris 10** operating system together with the **Sun Studio** (Oracle Solaris Studio) and **Java 8 SDK** for C++ and Java development, to serve for example as a _Jenkins build node_.

Also covers basic integration with different environments like other Linux servers, or basic _WINS_ setup so that the installed machine is accessible from Windows via the host name.

<!-- more -->

## Solaris 10 vs 11

To test C++ applications with the Solaris system, the Solaris 10 is the "easier" option than the more recent Solaris 11 version, because both the OS and the Solaris Studio can be downloaded for free.
In Solaris 11, a registration certification is needed to download and install the Solaris Studio.

The Solaris 10 and the Solaris Studio can be used free of charge under the **OTN License**, for the purposes of developing, testing, prototyping and demonstrating applications, even in commercial environments (for details, read the _OTN License agreement_ that must be accepted before downloading the installation media).

## 1. Prerequisites

The necessary prerequisites for installing the Solaris 10 operating system and the additional packages include the following:

- [Machine or VM](#11-machine-or-vm-to-install "Machine or VM to install")
- [Solaris 10](#12-solaris-10-installation-media "Solaris 10 installation media")
- [Solaris (Sun) Studio](#13-sun-studio-installation-package "Sun Studio installation package")
- [Java 8 SDK](#14-java-sdk-installation-package "Java SDK installation package")

### 1.1. Machine or VM to install

To install the system, you'll need indeed either a physical machine to install to, or it can be installed into a virtual machine ([VMware Playe_](https://www.vmware.com/go/downloadplayer), [Virtual Box](https://www.virtualbox.org/), [QEMU](http://www.qemu.org/), etc.).

When using a **[virtual machine](https://en.wikipedia.org/wiki/Virtual_machine)**, the target architecture would usually be **x86** (_64-bit_) - installing the _SPARC_ version would require an architecture _[emulator](https://en.wikipedia.org/wiki/Emulator)_, not a virtual machine (which can only virtualize the same architecture the host has).
The _QEMU_ can emulate various SPARC architectures to some extent, but the support of higher version of Solaris is [reportedly not particularly good](https://unix.stackexchange.com/questions/199827/booting-solaris-10-or-11-for-sparc-in-qemu-system-sparc64).

I currently have no experience if the _x86_ version can be installed on a regular desktop PC machine (and not a special Oracle/Sun provided hardware), but I suppose it would work in the principle (although it might not fully support all the available devices, also there might be issues regarding newer chipsets / CPU&zwj;s / network adapters etc.).

{% capture notice_contents %}
**<a name="vm-remark">VM remark</a>**

When installing into the **VMware** virtual machine, add / create the hard disks as "_IDE_", not the "_SCSI_" offered by default. When the HDD is added as SCSI, there might be issues with other device detection (CD/DVD IDE device etc.).
{% endcapture %}

{% include notice level="info" %}

### 1.2. Solaris 10 installation media

The installation media is available for download from Oracle:

- **[Oracle Solaris 10 Downloads](http://www.oracle.com/technetwork/server-storage/solaris10/downloads/index.html)**

According to the machine HW you plan to install the system to, either the _SPARC_ or the _x86_ version can be downloaded.
As I currently only installed the OS into a _VMware_ virtual machine on the x86 architecture, the article covers installation of the **x86** version of the OS, which may differ from the _SPARC_ version installation.

Note that there is **no separate 64-bit / 32-bit** installation media, as both packages are provided on a single media and the 64-bit packages are used by default.

When installing onto a physical machine, you'll need to _burn the Solaris 10 ISO image_ on a physical DVD media (or install from USB if supported).

If installing into a virtual machine, the downloaded ISO image can be usually "_inserted_" directly into the virtual CD/DVD drive (most virtualization solutions support that, for example both _VMware_ and _VirtualBox_ can do that).

### 1.3. Sun Studio installation package

The installation package of the Sun Studio is also available for download from the Oracle site:

- **[Oracle Solaris Studio 12.4 Downloads](http://www.oracle.com/technetwork/server-storage/developerstudio/downloads/solaris-studio-12-4-3045436.html)**

Download the appropriate installation package (_Solaris 10_, _x86_ or _SPARC_, also available as RPM for Linux).

### 1.4. Java SDK installation package

Installing the JDK (or JRE) is not absolutely necessary, as the Solaris 10 system installation already contains "_some Java_". Nevertheless the Java versions provided by the OS are outdated (1.6 the newest), therefore it is highly recommended to install a recent Java version (for example, the _Jenkins_ slave package will now only run under Java 8, as Java 7 support [has been dropped](https://jenkins.io/blog/2017/04/10/jenkins-has-upgraded-to-java-8/)).

The Java 8 downloads are available from Oracle as well:

- **[Java SE Development Kit 8 Downloads](http://www.oracle.com/technetwork/java/javase/downloads/jdk8-downloads-2133151.html)**
- **[Java SE Runtime Environment 8 Downloads](http://www.oracle.com/technetwork/java/javase/downloads/jre8-downloads-2133155.html)**

The SDK is recommended (it also contains the JRE).
Again, select the appropriate OS and architecture version.

To install the Java SDK **system-wide**, download the "_*.tar.Z_" version of the installation package (the _SVR4_ package). The "_*.tar.gz_" package is for installing the SDK privately for a particular user - see here for further details: [JDK 8 Installation on the Oracle Solaris Operating System](https://docs.oracle.com/javase/8/docs/technotes/guides/install/solaris_jdk.html#A1097833).

[Next: System installation]({% post_url 2017-06-11-minimal-install-solaris-10-with-sun-studio-2 %}){: .btn .btn--info}
{: .align-right}

{% include abbrev domain="computers" %}
