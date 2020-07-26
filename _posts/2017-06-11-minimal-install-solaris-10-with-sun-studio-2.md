---
title: "Minimal installation of Solaris 10 with Sun Studio"
header:
  teaser: /assets/images/thumb/sun-solaris.png
toc: true
hidden: true
categories:
  - Operating Systems
  - UNIX/Linux
tags:
  - solaris
  - sun-studio
  - vm
  - minimal-install
  - csw
  - open-csw
  - jenkins
---

![Speed](/assets/images/thumb/sun-solaris.png){: .align-left .img-thumbnail}

Minimal installation of the **Solaris 10** operating system together with the **Sun Studio** (Oracle Solaris Studio) and **Java 8 SDK** for C++ and Java development, to serve for example as a _Jenkins build node_.

Also covers basic integration with different environments like other Linux servers, or basic _WINS_ setup so that the installed machine is accessible from Windows via the host name.

<!-- more -->

## Solaris 10 vs 11

To test C++ applications with the Solaris system, the Solaris 10 is the "easier" option than the more recent Solaris 11 version, because both the OS and the Solaris Studio can be downloaded for free.
In Solaris 11, a registration certification is needed to download and install the Solaris Studio.

The Solaris 10 and the Solaris Studio can be used free of charge under the **OTN License**, for the purposes of developing, testing, prototyping and demonstrating applications, even in commercial environments (for details, read the _OTN License agreement_ that must be accepted before downloading the installation media).

## 1. Prerequisites

The necessary prerequisites for installing the Solaris 10 operating system and the additional packages include the following:

1. [Machine or VM](#11-machine-or-vm-to-install "Machine or VM to install")
2. [Solaris 10](#12-solaris-10-installation-media "Solaris 10 installation media")
3. [Solaris (Sun) Studio](#13-sun-studio-installation-package "Sun Studio installation package")
4. [Java 8 SDK](#14-java-sdk-installation-package "Java SDK installation package")

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

## 2. System installation

1. [Booting the installation media](#installation-boot "Booting the installation media")
2. [Starting the installer](#installation-start "Starting the installer")
3. [Initial network settings](#installation-network "Initial network settings")
4. [System settings](#installation-settings "System settings")
5. [Software selection](#installation-software "Software selection")
6. [Disk settings](#installation-disk "Install disk selection")

### 2.1. Booting the installation media

Insert the installation media and start the machine.

If there is already an OS installed on the machine, you might need to change the_ order of boot devices_ in the BIOS, so that the CD/DVD device will be booted first before the hard drive. However, if the hard drive is empty, leave the CD/DVD device as secondary boot device after the hard drive, so that the installer does not start again after the installation is complete.

When creating a virtual machine, the recommended size of the HDD to create is 20 GB (see later in the article, how to add additional disk for extra data).

When booted, the installer boot manager appears:

[caption id="attachment_358" align="aligncenter" width="700"]<img class="size-full wp-image-358" src="https://devsector.files.wordpress.com/2017/06/boot.png" alt="" width="700" height="389" /> Solaris 10 Installer Boot Manager[/caption]

Here you can either start the installation, or press the 'e' key to edit the boot commands first.
<h4>**1.1. Disabling the USB (EHCI and UHCI) support**</h4>
If the Solaris 10 is used under QEMU (for example in [_QNAP NAS Virtualization Station_](https://www.qnap.com/solution/virtualization-station/en-us/), which uses the QEMU under the hood), it is recommended to **_disable the USB EHCI and UHCI support_**, as those are not virtualized well and Solaris 10 then spends a considerable amount of time trying to detect the devices (which is not able to do at the end anyway).

To disable, press the 'e' key upon the above boot screen, then press it again to edit the first line (starting with "kernel$") and add the following to the end of the line:
<pre style="color: #bbbbbb; background-color: #000000;">**,disable-ehci=true,disable-uhci=true**</pre>
Then press 'Enter' to return back, and the 'b' key to start the boot process.

### 2.2. Starting the installer

After the installer boots up, select the installation option 4 to start the installation in the "_Interactive Text_" mode:

[caption id="attachment_361" align="aligncenter" width="700"]<img class="size-full wp-image-361" src="https://devsector.files.wordpress.com/2017/06/install-1-select.png" alt="" width="700" height="389" /> Selection of the installation type[/caption]

The option 3 or 4 is required when the [**ZFS**](https://en.wikipedia.org/wiki/ZFS) is to be used, which is recommended (supports wide range of features, like snapshots etc.).

Next is setting of the keyboard layout - note that the choice is not confirmed by the 'Enter', but by the 'F2' key:

[caption id="attachment_362" align="aligncenter" width="700"]<img class="size-full wp-image-362" src="https://devsector.files.wordpress.com/2017/06/install-2-keyboard.png" alt="" width="700" height="389" /> Choosing the keyboard layout[/caption]

Bunch of next dialogs follow, before setting up the network.

### 2.3. Initial network settings

The network connectivity setup:

[caption id="attachment_363" align="aligncenter" width="700"]<img class="size-full wp-image-363" src="https://devsector.files.wordpress.com/2017/06/install-3-network.png" alt="" width="700" height="389" /> Network connectivity setup[/caption]

Normally the system should be using [**DHCP**](https://de.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol), which assigns the IP address to the machine accordingly.

If the static IP address is needed, it can be setup there as well. In that case, you'll need to choose the host name of the machine, the IP address and provide the gateway ("router") IP address.

In the case of _DHCP_ selection, the installer does not ask for the host name, and the system will try to get the host name over the _DHCP_ as well. However, most of the _DHCP_ servers nowadays do not provide the host names (would require a special setup on the _DHCP_ server), so the host name will be [set later](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/3/#post-network-hostname) after the basic installation is done.

Recommended network settings:

- **Networked**: _Yes_
- **Use DHCP**: _Yes_
- **Enable IPv6**: _No_

Further network settings, which I used in my case:

- **Configure Kerberos**: _No_
- **Name service**: _None_
(or _DNS_, but that is only required when using the static IP address - with DHCP, DNS servers will be retrieved from the DHCP server even with setting this to _None_)
- _Use the NFSv4 domain derived by the system_

### 2.4. System settings

Follow some settings of the system:

- **Time Zone**: Select appropriately
- **Root password**: The main administrator user password
- **Remote services**: _No_ (only SSH will be allowed initially)
- **Register**: _No_
- **Proxy server**: Set appropriately

&nbsp;

- **Installation type**: _F2_ (Standard)
- **iSCSI target**: Select appropriately
(note that iSCSI drive requires additional settings)
- _Automatically eject CD/DVD_
- _Auto Reboot_
- **Media**: _CD/DVD_
- **Regions**: Select appropriately
(can be skipped to use default US setting; select _North America_/_U.S.A. (UTF-8)_ to allow setting locale to UTF-8)
- **Locale**: Select appropriately (_POSIX C_ the default)
- **Additional Products**: _None_
- **Filesystem**: _ZFS_ (or _UFS_ according to preference)

### 2.5. Software selection

For the minimal installation, select the option: _**Reduced Networking Core System Support**_.
This is to perform the _minimal text-based_ install.

[caption id="attachment_364" align="aligncenter" width="700"]<img class="size-full wp-image-364" src="https://devsector.files.wordpress.com/2017/06/install-4-minimal.png" alt="" width="700" height="389" /> Software bundle selection[/caption]

Do not press the '_F2_' yet, but press the '_F4_' key instead to _**customize and add extra packages**_, required for using the SSH, installing the Solaris Studio etc.

Alternatively you can select the "_**Developer System Support**_" (Sun Studio installer actually complains if this or the "_Entire Distribution_" option is not selected, but if the required packages are added manually, the Sun Studio will install and work properly). But as you can see, this option install a lot more packages, including the full graphical desktop, which you might not want or need for the minimal system.

When selected the minimal install option and '_F4_' to customize, add the following packages (some just for convenience, some required for the Sun Studio or Java):

- **A Windows SMB/CIFS fileserver for UNIX** (the entire group):

- Required for _NetBios name resolution_ (from Windows)
- If you do not need to access the machine via hostname from Windows, does not need to be installed
- Can be also installed later if needed


- **Apache Standard C++ Library**:

- <span style="color: #ff0000;">Required for the Oracle Solaris Studio</span>


- **BIND DNS Name server and tool**:

- Required for _DNS name resolution_
- Installs the "nslookup" tool for troubleshooting the networking issues


- **Basic IP commands (Usr)**:

- Installs the "ping" command


- **CPU Performance Counter driver and utilities **(open the group by 'Enter' on the '&gt;' before the group name)

- **CPU Performance Counter libraries and utilities**:

- <span style="color: #ff0000;">Required for the Oracle Solaris Studio</span>




- **Documentation tools**:

- Installs the "man" command


- **Freeware Other Utilities** (open the group)

- **The GNU pager (less)**:

- Installs the "less" command




- **Freeware Shells** (open the group)

- **GNU Bourne-Again shell (bash)**:

- I prefer to use this shell (see below how to set the bash shell as default)
- zsh can be selected here as well, if you prefer or plan to use it




- **GNOME Runtime** (open the group)

- **The Python interpreter, libraries and utilities**
- **The Python interpreter, libraries and utilities - development files**

- <span style="color: #ff0000;">Required for the Oracle Solaris Studio</span>




- **GNU binutils, C compiler, m4 and make** (open the group)

- **GCC Runtime Libraries**
- **binutils - GNU binutils**
- **gcc - The GNU C compiler**
- **gmake - GNU m4**
- **gmake - GNU make**

- Not required, only if you plan to use the _GCC compiler_ as well




- **GNU wget** (the entire group):

- Installs the "wget" command


- not selecting any additional Java packages:

- Java 8 will be installed later from the downloaded package


- **Lint Libraries (usr)**:

- <span style="color: #ff0000;">Required for the Oracle Solaris Studio</span>


- **Motif RunTime Kit**

- <span style="color: #ff0000;">Required for the Java 8 SDK</span>


- **PICL Framework, Libraries, and Plugin Modules** (the entire group)
- **PICL Header Files (Usr)**

- <span style="color: #ff0000;">Required for the Oracle Solaris Studio</span>


- **Portable layout services for Complex Text Layout support**

- <span style="color: #ff0000;">Required for the Java 8 SDK</span>


- **Programming tools and libraries** (open the group)

- **Math &amp; Microtasking Libraries (Usr)**
- **Math &amp; Microtasking Library Headers &amp; Lint Files (Usr)**
- **Solaris Bundled tools**
- **Solaris cpp**

- <span style="color: #ff0000;">Required for the Oracle Solaris Studio</span>




- **Secure Shell** (the entire group):

- Installs the SSH server for remote login


- **SunOS Header Files**:

- <span style="color: #ff0000;">Required for the Oracle Solaris Studio</span>


- **rsync**:

- Not essential, but can be handy



[caption id="attachment_369" align="aligncenter" width="700"]<img class="size-full wp-image-369" src="https://devsector.files.wordpress.com/2017/06/install-5-sw-selection-1.png" alt="" width="700" height="389" /> Additional software selection[/caption]

After selecting all the additional software, press '_F2_' to continue.

The package manager will select some additional dependencies, press '_F4_' to preserve all the required dependencies:

[caption id="attachment_370" align="aligncenter" width="700"]<img class="size-full wp-image-370" src="https://devsector.files.wordpress.com/2017/06/install-6-dependencies.png" alt="" width="700" height="389" /> Preserve package dependencies[/caption]

See here for the list of Solaris Studio requirements: [Oracle Solaris Studio 12.4 - Required System Software Packages](http://docs.oracle.com/cd/E37069_01/html/E37070/gnzpf.html#scrolltoc)

### 2.6. Install disk selection

In the next step, the disk for the installation can be selected:

[caption id="attachment_371" align="aligncenter" width="700"]<img class="size-full wp-image-371" src="https://devsector.files.wordpress.com/2017/06/install-7-disk.png" alt="" width="700" height="389" /> Disk selection[/caption]

Here you can confirm the defaults and continue by '_F2_', or press '_F4_' and make changes.

Next is to configure the _ZFS_ settings (if _ZFS_ was selected before). I normally leave the defaults alone:

[caption id="attachment_372" align="aligncenter" width="700"]<img class="size-full wp-image-372" src="https://devsector.files.wordpress.com/2017/06/install-8-zfs.png" alt="" width="700" height="389" /> ZFS settings[/caption]

Afterwards there are few other confirmation dialogs and the installation begins. Wait for the installation to complete and restart. If you changed the order of boot devices in the BIOS to boot from CD/DVD device first, you'll need to remove the installation media or change the boot order back to start from the HDD first (recommended), otherwise the installer will start again.

On the first boot, the machine should start into the text console (as we didn't install the graphical desktop with the minimal installation), where you can login into the root account with the password you set during the installation:

[caption id="attachment_375" align="aligncenter" width="700"]<img class="size-full wp-image-375" src="https://devsector.files.wordpress.com/2017/06/install-9-done1.png" alt="" width="700" height="389" /> Initial login[/caption]

The default shell is "/bin/sh", you can type "bash" to switch to the bash shell (if you installed it).

Note that the _**hostname** _is reported as "_unknown_", because the _DHCP_ server didn't configure it (but you can see that the IP address has been configured over _DHCP_). We will [set the host name manually](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/3/#post-network-hostname) later (alternatively the _DHCP_ server would have to be configured to know which host name it should push for the particular machine).
<p style="text-align: right;">[**Next Page: _Post-install setup_**](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/3/)</p>
<!--nextpage-->

## 3. Post-install setup

1. [Adding user](#post-user "Adding a regular user")
2. [SSH access](#post-ssh "Setting up the SSH access")
3. [Setting paths](#post-paths "Setting the paths to the GCC compiler and tools")
4. [Shutdown/reboot](#post-shutdown "Shutting down/rebooting the system")
5. [Mounting CD/DVD](#mount-cdrom "Mounting the CD/DVD device")
6. [Boot manager](#post-bootmgr "Editing the boot manager menu")
7. [Network setup](#post-network "Setting up the network") (DNS, hostname, WINS)

### 1. Adding a regular user

It is recommended to add a regular user for the normal work (for security reasons). Also, the remote SSH login of the "root" superuser is disabled by default (the "root" user access over SSH can be enabled, but it is disabled for security reasons, so I generally prefer to keep it disabled and create an additional "regular" user).

A new user can be added via the "useradd" command, and the password set via the "passwd":
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">useradd</span> -m -d /export/home/my_username -s /usr/bin/bash my_username</strong>
UX: useradd: my_username name too long
<strong style="color: #ffffff;"># <span style="color: #ffff00;">passwd</span> my_username</strong>
New Password:
Re-enter new Password:
passwd: password successfully changed for my_username
</pre>
The "-m -d" is to create the home directory and its location. Note the the standard _**location of home directories** _in Solaris is not <tt style="color: #4682b4; background-color: #fff8dc;">"/home"</tt> as usual in Linux, but <tt style="color: #4682b4; background-color: #fff8dc;">"/export/home"</tt> (see here: [Difference between "/export/home" and "/home"](https://unix.stackexchange.com/questions/11685/difference-between-export-home-and-home)).

The user is now created with the Bash as the default shell (you can choose any other shell or not use the "-s" parameter to keep the default "/bin/sh" shell).

The warning about "_username too long_" can be normally ignored (only some special services might require the name to be shorter than 8 chars, but I didn't hit any issues with longer usernames so far).

To change the default shell for the root user as well, you can do:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">usermod</span> -s /usr/bin/bash root</strong>
UX: usermod: root is currently logged in, some changes may not take effect until
 next login.
</pre>
To perform administrative tasks when logged as a regular user, the "su" command needs to be used to switch to the "root" account (Solaris does not provide the "sudo" command):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">-bash-3.2$ <span style="color: #ffff00;">su</span> -</strong>
Password:
Oracle Corporation      SunOS 5.10      Generic Patch   January 2005
<strong style="color: #ffffff;">-bash-3.2# </strong>
</pre>
Note the different prompt strings for the regular user (ending with the "$" character) and the administrator account (ending with "#"). This is the default, but the prompt strings can be changed.

### 2. Setting up the SSH access

The SSH server should already be running, and you can use the previously created regular user to log in via SSH or Putty (see also here: [SSH password-less login](https://devsector.wordpress.com/2017/06/03/ssh-passwordless-login/)). If the DHCP was selected during installation, you will not be able to connect via the hostname, only by the IP address (the setup of the host name is [explained later](#post-network-hostname)).

The IP address to connect to can be shown by the "ifconfig" command:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">ifconfig</span> -a</strong>
lo0: flags=2001000849&lt;UP,LOOPBACK,RUNNING,MULTICAST,IPv4,VIRTUAL&gt; mtu 8232 index 1
        inet 127.0.0.1 netmask ff000000
e1000g0: flags=1004843&lt;UP,BROADCAST,RUNNING,MULTICAST,DHCP,IPv4&gt; mtu 1500 index 2
        inet 192.168.0.100 netmask ffffff00 broadcast 192.168.0.255
</pre>
The SSH access will be also used for transferring the downloaded additional packages (Sun Studio, Java) to the machine.

### 3. Setting the paths to the GCC compiler and tools

The GCC compiler and tools are located under "/usr/sfw", which is not in the path by default. The path can be added in the <tt style="color: #4682b4; background-color: #fff8dc;">"/etc/profile"</tt>, but it is better to add it to the <tt style="color: #4682b4; background-color: #fff8dc;">"/etc/default/login"</tt> file, to be also available for the _**non-interactive shells **_- this is in particular essential to work within Jenkins build slave (which runs under non-interactive shell). This needs to be done under the "root" user ("su -"):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u+w /etc/default/login</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /etc/default/login</strong>
</pre>
(need to change the permissions first, because the file is read-only by default)

Find the PATH setting (commented out by default) and change it to (or add):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">PATH=/usr/bin:/bin:/usr/sbin<span style="color: #00ff00;">:/usr/sfw/bin:/usr/sfw/i386-sun-solaris2.10/bin</span></strong>
</pre>
(the path "/usr/sbin" is mostly relevant to the "root" superuser, as the regular user can only run few commands present there, for example "ping")

At this point, you can add the paths for the Sun Studio and the [CSW packages](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/5/#extras-csw) as well, to not have to repeat the setting later:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">PATH=/usr/bin:/bin:/usr/sbin<span style="color: #00ff00;">:/opt/csw/bin:/opt/solarisstudio12.4/bin:/usr/sfw/bin:/usr/sfw/i386-sun-solaris2.10/bin</span></strong>
</pre>
When done, it is recommended to change the permissions back to read-only:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u-w /etc/default/login</strong>
</pre>
For the settings to take effect, logout and login back (or restart the machine). Confirm the installation by checking the "gcc" (GNU C compiler) and "g++" (GNU C++ compiler) commands - under the regular user:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">$ <span style="color: #ffff00;">which</span> gcc</strong>
/usr/sfw/bin/gcc
<strong style="color: #ffffff;">$ <span style="color: #ffff00;">which</span> g++</strong>
/usr/sfw/bin/g++
<strong style="color: #ffffff;">$ <span style="color: #ffff00;">gcc</span> --version</strong>
gcc (GCC) 3.4.3 (csl-sol210-3_4-branch+sol_rpath)
...
<strong style="color: #ffffff;">$ <span style="color: #ffff00;">g++</span> --version</strong>
g++ (GCC) 3.4.3 (csl-sol210-3_4-branch+sol_rpath)
...
</pre>
Note that the GCC compiler shipped with the system is of a relatively old version, so you might want to [install a newer one](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/5/#extras-devtools-gcc) (that can be done via the CSW repository).

### 4. Shutting down/rebooting the system

Similar to Linux, the Solaris 10 can be shutdown/rebooted by the "shutdown" or "init" commands (both can only be run under the "root" administrator account, located under "/usr/sbin").

To shutdown:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">shutdown</span> -y -i5</strong>    <span style="color: #008000;"># shutdown with grace period (-g0 for immediate shutdown)</span>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">init</span> 5</strong>             <span style="color: #008000;"># alternative</span>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">poweroff</span></strong>           <span style="color: #008000;"># alternative
</span></pre>
To restart:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">shutdown</span> -y -i6</strong>    <span style="color: #008000;"># restart with grace period (-g0 for immediate restart)</span>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">init</span> 6</strong>             <span style="color: #008000;"># alternative</span>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">reboot</span></strong>             <span style="color: #008000;"># alternative
</span></pre>
The "shutdown" command is the most graceful option, especially if multiple users might be using the machine - it allows a grace period, and sends broadcast messages to the connected users, that the system is about do go down or reboot.

### 5. Mounting the CD/DVD device

The CD/DVD device needs to be mounted if you e.g. want to install additional packages.

The device names are different to Linux, so you need to know the scheme to be able to mount them. All needs to be performed under the "root" user, i.e. if logged under a regular user, elevate the rights via "su -". The "standard" CD/DVD device name is usually "c1t0d0".

To mount the CD/DVD device (using "mkdir" to create the mount directory if it does not exist yet):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">mkdir</span> /cdrom</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">mount</span> -r -F hsfs /dev/dsk/c1t0d0s0 /cdrom</strong>
</pre>
Note the "s0" added at the end - "c1t0d0" is the entire physical device, the "s_n_" is the _n-th_ partition on the device (zero-based index).

If the device above is not found, you'll need to list the devices to find out the correct device name. This can be done with the "iostat" command:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">iostat</span> -En</strong>
<span style="color: #ff0000;">**c0d0** </span>Soft Errors: 0 Hard Errors: 0 Transport Errors: 0
Model: VMware Virtual Revision: Serial No: 000000000000000 Size: 21.47GB &lt;21474754560 bytes&gt;
Media Error: 0 Device Not Ready: 0 No Device: 0 Recoverable: 0
Illegal Request: 0
**<span style="color: #ff0000;">c1t0d0</span> **Soft Errors: 0 Hard Errors: 0 Transport Errors: 0
Vendor: NECVMWar Product: VMware IDE CDR10 Revision: 1.00 Serial No:
Size: 0.00GB &lt;0 bytes&gt;
Media Error: 0 Device Not Ready: 0 No Device: 0 Recoverable: 0
Illegal Request: 6 Predictive Failure Analysis: 0
</pre>
Here the CD device name is the "c1t0d0" ("c0d0" is the first hard drive).

See here for further details: [How to mount the CD-ROM on Solaris 10?](https://unix.stackexchange.com/questions/78791/how-to-mount-the-cd-rom-on-solaris-10)

### 6. Editing the boot manager menu

To edit the boot menu, use the "bootadm" command to find out the path of the active configuration:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">bootadm</span> list-menu</strong>
The location for the active GRUB menu is: <span style="color: #ff0000;">**/rpool/boot/grub/menu.lst**</span>
default 0
timeout 10
0 Oracle Solaris 10 1/13 s10x_u11wos_24a X86
1 Solaris failsafe
</pre>
Then you can edit the boot menu:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /rpool/boot/grub/menu.lst</strong>
</pre>
Here you can do for example:
<h4>**6.1. Changing the boot timeout**</h4>
Update the value of the "_**timeout**_" setting.
<h4>**6.2. Disabling the USB (EHCI, UHCI)**</h4>
As already [mentioned before](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/2/#installation-boot-nousb), under QEMU the USB device drivers can cause issues, you might also want to disable the USB devices in case you do not plan to use them.

To disable, find the current active boot option and add the parameters to disable the EHCI/UHCI:
<pre style="color: #bbbbbb; background-color: #000000;">title Oracle Solaris 10 1/13 s10x_u11wos_24a X86
findroot (pool_rpool,0,a)
<strong style="color: #ffffff;">kernel$ /platform/i86pc/multiboot -B $ZFS-BOOTFS<span style="color: #00ff00;">,disable-ehci=true,disable-uhci=true</span></strong>
module /platform/i86pc/boot_archive
</pre>
The USB device drivers will be disabled after the next reboot.

### 7. Setting up the network

<h4>**7.1. Setting up the hostname (DHCP)**</h4>
If the system has been configured to _**use DHCP**_, no computer name has been defined explicitly and therefore it shows as "_unknown_". The system tries to receive the hostname via DHCP, but most DHCP servers do not supply the host names nowadays (and the DHCP server would need to be configured with hostnames for each particular machine anyway). This problem does not exist if the static IP has been defined and you provided an explicit computer name, nevertheless it is easy to fix.

First, disable the retrieval of the hostname over DHCP (so the system will not try to retrieve the name any more):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /etc/default/dhcpagent</strong>
</pre>
Find the value "_**PARAM_REQUEST_LIST**_" there, and remove the value "12" from it:
<pre style="color: #bbbbbb; background-color: #000000;"><span style="color: #008000;"># Before:</span>
PARAM_REQUEST_LIST=1,3,6,**<span style="color: #ff0000;">12,</span>**15,28,43

<span style="color: #008000;"># After:</span>
PARAM_REQUEST_LIST=1,3,6,15,28,43
</pre>
Do the same for the "_**.v6.PARAM_REQUEST_LIST**_" if you enabled IPv6.

Next set the hostname in the <tt style="color: #4682b4; background-color: #fff8dc;">"/etc/nodename"</tt> file:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /etc/nodename</strong>
</pre>
Insert the host name there.

Last location to set the hostname is for resolution from other machines over a particular network interface (note that each network interface can be set with a different hostname, although it is not recommended). To set the hostname for network interface "e1000g0":
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /etc/hostname.e1000g0</strong>
</pre>
Insert the following there ("test-sol10" is the example host name):
<pre style="color: #bbbbbb; background-color: #000000;">inet test-sol10
</pre>
You could also notice the error message regarding the "loghost" (which should point to the local machine):
<pre style="color: #bbbbbb; background-color: #000000;">syslogd: line 24: WARNING: loghost could not be resolved
</pre>
This can be fixed by updating the <tt style="color: #4682b4; background-color: #fff8dc;">"/etc/hosts"</tt> file:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u+w /etc/inet/hosts</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /etc/inet/hosts</strong>
</pre>
Add the "loghost" at the end of the "localhost" line:
<pre style="color: #bbbbbb; background-color: #000000;">127.0.0.1       localhost <span style="color: #00ff00;">loghost
</span></pre>
As you might notice, the file was originally write-protected (needed to "chmod" to be able to edit it even under the "root" user), so the permission to write should be removed again:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u-w /etc/inet/hosts</strong>
</pre>
Restart the machine to apply the new settings. The new host name should now be shown, there also should not be the "loghost" warning any more:

[caption id="attachment_380" align="aligncenter" width="700"]<img class="size-full wp-image-380" src="https://devsector.files.wordpress.com/2017/06/sol10-hostname.png" alt="" width="700" height="389" /> Hostname set successfully[/caption]

The "hostname" command should now also show the new host name:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">$ <span style="color: #ffff00;">hostname</span></strong>
test-sol10
</pre>
Now the Solaris 10 machine should be also reachable from other UNIX/Linus machines via the hostname. Nevertheless, the host name will still not be resolved from Windows machines, because Windows uses different protocol ([_NetBios_](https://en.wikipedia.org/wiki/NetBIOS)).
<h4>**7.2. Setting up NetBios name for Windows clients (Samba/WINS)**</h4>
As mentioned, Windows uses different means than Linux to resolve the local network machines via hostnames (the older [_NetBios_](https://en.wikipedia.org/wiki/NetBIOS) protocol). Therefore to resolve the Solaris 10 machine from Windows, the system must be set up specifically via Samba/WINS.

We will not actually configure and launch the Samba server (although this can be done as well, if you want to access drives in Solaris via regular Windows/Samba shares, but it is not covered by the article). Still for the WINS resolution to work, the Samba server must be installed and partially configured on the Solaris machine.

If you do not need to access the machine from Windows via hostname, then this step can be skiped.

If you did not install the Samba packages during the installation, you can do that manually as well. First, you need to insert and [mount the installation media](#mount-cdrom). Then the required packages can be installed by the "pkgadd" command (under the "root" user, obviously):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgadd</span> -d /cdrom/Solaris_10/Product SUNWsmbar SUNWsmbac SUNWlibpopt SUNWsmbau SUNWsfman</strong>
</pre>
Once the packages are there, we will just copy the default configuration and enable the WINS service:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">cp</span> **/etc/samba**/smb.conf-example **/etc/samba/**smb.conf
</strong><strong style="color: #ffffff;"># <span style="color: #ffff00;">svcadm</span> enable wins</strong>
</pre>
The status can be tested by:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">svcs</span> -a | <span style="color: #ffff00;">grep</span> wins</strong>
<span style="color: #00ff00;">online</span>         18:52:56 svc:/network/wins:<span style="color: #00ff00;">default
</span></pre>
Now the WINS service should already be running and the Solaris machine resolvable from Windows via name. If it does not work yet, try to restart the machine and/or wait for some time first.
<p style="text-align: right;">[**Next Page: _Installing extra packages_**](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/4/)</p>
<!--nextpage-->

## Installing extra packages

1. [Java 8 SDK](#packages-jdk "Installing the Java 8 SDK")
2. [Sun Studio](#packages-studio "Installing the Solaris (Sun) Studio")

### 1. Installing the Java 8 SDK

To install the recent Java SDK or JRE, we downloaded the package before. Now it needs to be transferred to the machine.

First, create the target directory on the Solaris machine (which can be completely removed afterwards, as there will be some funky unpacking business going on):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">mkdir</span> /jdk-install</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> 777 /jdk-install</strong>
</pre>
If the Java SDK package was downloaded under Linux, it can be transferred via the "scp" command:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">[<span style="color: #00ff00;">username@client</span>]$ <span style="color: #ffff00;">scp</span> ~/Download/jdk-8u131-solaris-x64.tar.Z <span style="color: #ff9900;">username@sol10-machine</span>:/jdk-install/</strong>
</pre>
Under Windows the [**WinSCP**](https://winscp.net/) client can be used to copy the package. You can import the machine settings from PuTTY. But make sure to _**change the default file protocol** _to "**SCP**" ("_SFTP_" would require a different service which is not running on the Solaris machine):

[caption id="attachment_381" align="aligncenter" width="626"]<img class="size-full wp-image-381" src="https://devsector.files.wordpress.com/2017/06/winscp-scp.png" alt="" width="626" height="423" /> WinSCP protocol selection[/caption]

Transfer the file to the target directory (locate it first in the right panel):

[caption id="attachment_382" align="aligncenter" width="700"][<img class="size-large wp-image-382" src="https://devsector.files.wordpress.com/2017/06/winscp-transfer.png?w=700" alt="" width="700" height="451" />](https://devsector.files.wordpress.com/2017/06/winscp-transfer.png) WinSCP transfer file[/caption]

Before installing the Java SDK, the _**pre-requisites** _need to be installed from the Solaris 10 installation media (_if not already installed during the system installation as recommended_):

- [mount the installation media](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/3/#mount-cdrom)

<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgadd</span> -d /cdrom/Solaris_10/Product SUNWctpls SUNWmfrun</strong>
</pre>
To install the Java SDK on Solaris (under the "root" user):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">cd</span> /jdk-install</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">zcat</span> jdk-8u131-solaris-x64.tar.Z | <span style="color: #ffff00;">tar</span> xf -</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgadd</span> -d . SUNWj8rt SUNWj8dev SUNWj8cfg SUNWj8man</strong>
</pre>
Confirm the installation by checking the Java version:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">java</span> -version</strong>
java version "1.8.0_131"
Java(TM) SE Runtime Environment (build 1.8.0_131-b11)
Java HotSpot(TM) 64-Bit Server VM (build 25.131-b11, mixed mode)
</pre>
If everything works, the installation directory can be removed:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">cd</span> /</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">rm</span> -rf /jdk-install</strong>
</pre>

### 2. Installing the Solaris (Sun) Studio

_**Oracle Solaris Studio** _(originally _Sun Sudio_) is the "native" C/C++ compiler suite used with the Solaris. The system is also supported by GCC, however you might want to make sure your applications build with the native compiler as well.

First step is to transfer the installation package to the machine. This it pretty the same as in the case of the [Java SDK](#packages-jdk).

We will create the target directory first:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">mkdir</span> /studio-install</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> 777 /studio-install</strong>
</pre>
Then transfer the installation file - under Linux by using "scp":
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">[<span style="color: #00ff00;">username@client</span>]$ <span style="color: #ffff00;">scp</span> ~/Download/SolarisStudio12.4-solaris-x86-pkg.tar.bz2 <span style="color: #ff9900;">username@sol10-machine</span>:/studio-install/</strong>
</pre>
Under Windows the [**WinSCP**](https://winscp.net/) client can be used again.

Before installing the Sun Studio, the _**pre-requisites** _need to be installed from the Solaris 10 installation media (_if not already installed during the system installation as recommended_):

- [mount the installation media](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/3/#mount-cdrom)

<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgadd</span> -d /cdrom/Solaris_10/Product **SUNWhea SUNWarc SUNWcpcu SUNWcpp SUNWcsl SUNWlibC SUNWtoo SUNWlibmr SUNWlibm SUNWlibms SUNWpiclh SUNWpiclr SUNWpiclu SUNWsprot SUNWlibstdcxx4 SUNWrcmdc SUNWrcmdr SUNWrcmds SUNWtftp SUNWtnetc SUNWscpr SUNWscpu SUNWPython SUNWPython-devel**</strong>
</pre>
The list of pre-requisites is available here: [Required System Software Packages](http://docs.oracle.com/cd/E37069_01/html/E37070/gnzpf.html)

To install the Solaris Studio (under the "root" user):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">cd</span> /studio-install</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">bzip2</span> -dc SolarisStudio12.4-solaris-x86-pkg.tar.bz2 | <span style="color: #ffff00;">tar</span> xf -</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">cd</span> SolarisStudio12.4-solaris-x86-pkg</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">./solarisstudio.sh</span> --non-interactive --tempdir /studio-install/tmp --install-components c-and-cpp-compilers,code-analyzer-tool,dbx-debugger,dbxtool,dmake,fortran-compiler,oic,performance-and-thread-analysis-tools,performance-library</strong>
</pre>
As we did not install the graphic desktop and do not plan to use the Solaris Studio IDE, we will not install it.

See here for the list of available Solaris Studio components and the installation options:

- [Command-Line Options for the Installer, Uninstaller, and install_patches Utility for Oracle Solaris 10 and Linux Platforms](https://docs.oracle.com/cd/E37069_01/html/E37072/gozps.html#OSSIGgiqse)
- [Components and Package Names in Oracle Solaris Studio](https://docs.oracle.com/cd/E37069_01/html/E37072/gozpl.html)

Note the use of the "--tempdir" parameter - the default location is "/tmp", however as it is a relatively small partition by default, it might not have enough space required for the Solaris Studio installation.

Because we installed the system by using the minimal install option and not the recommended "_Developer System Support_" or "_Entire Distribution_" options, the installer will complain about that. This can be ignored now, as we explicitly added/installed all the prerequisites of the Sun Studio:
<pre style="color: #bbbbbb; background-color: #000000;">[2017-06-05 02:35:37.315]: WARNING - Required patches should be installed. The following patches need to be installed for Oracle Solaris Studio to function correctly: 120754-13, 119964-31, 147437-02, 119961-13. You can use the install_patches.sh utility in your download directory to install the patches.
<strong style="color: #ffffff;">[2017-06-05 02:35:37.315]: WARNING - Unsupported Solaris meta cluster is detected. To be able to use the compilers, one of the following Solaris meta clusters must be installed: Entire Solaris Software Group, Entire Solaris Software Group Plus OEM Support or Developer Solaris Software Group.</strong>
</pre>
Next step is to install the required patches (as the other warning message suggests):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">./install_patches.sh</span></strong>
</pre>
Next step is to add the Solaris Studio paths to the environment (alternative is to use the "--create-symlinks" for the installer, but for me it didn't work and the non-interactive installer choked on that option).

First we will add the Solaris Studio man path. This can be done in the <tt style="color: #4682b4; background-color: #fff8dc;">"/etc/profile"</tt> file, by adding the following setings (in green):
<pre style="color: #bbbbbb; background-color: #000000;">#ident  "@(#)profile    1.19    01/03/13 SMI"   /* SVr4.0 1.3   */

# The profile that all logins get before using their own .profile.

trap ""  2 3
<span style="color: #00ff00;">**MANPATH=$MANPATH:/opt/solarisstudio12.4/man**</span>
export LOGNAME PATH <span style="color: #00ff00;">**MANPATH**</span>

...
</pre>
The path can be also added here, but it is better to add it to the <tt style="color: #4682b4; background-color: #fff8dc;">"/etc/default/login"</tt> file, to be also available in the _**non-interactive shells** _- this is in particular essential for Solaris Studio to work within Jenkins build slave:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u+w /etc/default/login</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /etc/default/login</strong>
</pre>
(need to change the permissions first, because the file is read-only by default)

Find the PATH setting (might be commented out) and change it to (or add):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">PATH=/usr/bin:/bin:/usr/sbin<span style="color: #00ff00;">:/opt/solarisstudio12.4/bin</span>:/usr/sfw/bin:/usr/sfw/i386-sun-solaris2.10/bin</strong>
</pre>
(note adding the path before "/usr/sfw/bin" to take precedence)

When done, change the rights back to read-only:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u-w /etc/default/login</strong>
</pre>
For the settings to take effect, logout and login back (or restart the machine). Confirm the installation by checking the "cc" (Sun C compiler) and "CC" (Sun C++ compiler) commands - under the regular user:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">$ <span style="color: #ffff00;">which</span> cc</strong>
/opt/solarisstudio12.4/bin/cc
<strong style="color: #ffffff;">$ <span style="color: #ffff00;">which</span> CC</strong>
/opt/solarisstudio12.4/bin/CC
<strong style="color: #ffffff;">$ <span style="color: #ffff00;">cc</span> -V</strong>
cc: Sun C 5.13 SunOS_i386 2014/10/20
<strong style="color: #ffffff;">$ <span style="color: #ffff00;">CC</span> -V</strong>
CC: Sun C++ 5.13 SunOS_i386 2014/10/20
</pre>
The "man" path can be tested by:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">$ <span style="color: #ffff00;">man</span> codean</strong>
Reformatting page. Please wait... done.

User Commands                                           codean(1)

NAME
     codean - Command Line Interface of Code Analyzer
...
</pre>
If everything works, the installation directory can be removed (under the "root" user):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">cd</span> /</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">rm</span> -rf /studio-install</strong>
</pre>
<p style="text-align: right;">[**Next Page: _Extras_**](https://devsector.wordpress.com/2017/06/11/minimal-install-solaris-10-with-sun-studio/5/)</p>
<!--nextpage-->

## Extras

- [CSW repository](#extras-csw "Installing and using the CSW repository")
- [Secondary disk](#extras-disk "Adding a secondary disk")
- [Jenkins node](#extras-jenkins "Jenkins node setup")
- [Development tools](#extras-devtools "Installing additional development tools") (Git, CMake, GCC 4/5)

### Installing and using the CSW repository

The [Open Community SoftWare](https://www.opencsw.org/) (OpenCSW) project is an additional repository of software packages for Solaris. It provides newer package versions and/or packages which do not exist in Solaris originally (Git, CMake, etc.).

The repository is added as an additional package (needs to be done under the superuser - "su -"):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgadd</span> -d http://get.opencsw.org/now</strong>

## Downloading...

..............25%..............50%..............75%..............100%

## Download Complete


The following packages are available:
  1  CSWpkgutil     pkgutil - Installs Solaris packages easily
                    (all) 2.6.7,REV=2014.10.16

Select package(s) you wish to process (or 'all' to process
all packages). (default: all) [?,??,q]:
</pre>
The package installs the CSW package management utility ("pkgutil"), which works separately from the original Solaris package manager.

The packages are installed in a separate directory ("/opt/csw"), therefore the paths need to be updated. The user path (it you already didn't do that after the system installation):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u+w /etc/default/login</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /etc/default/login</strong>
</pre>
Find the PATH setting and insert into it:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">PATH=/usr/bin:/bin:/usr/sbin<span style="color: #00ff00;">:/opt/csw/bin</span>:/opt/solarisstudio12.4/bin:/usr/sfw/bin:/usr/sfw/i386-sun-solaris2.10/bin</strong>
</pre>
The superuser ("root") paths are kept separately in the <tt style="color: #4682b4; background-color: #fff8dc;">"/etc/default/su"</tt> file, so the CSW path should be added there as well:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u+w /etc/default/su</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">vi</span> /etc/default/su</strong>
</pre>
The setting is kept in the "SUPATH" variable:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">SUPATH=/usr/sbin:/usr/bin<span style="color: #00ff00;">:/opt/csw/sbin</span></strong>
</pre>
Note that I normally do not add the Sun Studio/SFW paths for the root (as those are better to only run under the regular user), but the CSW path should be added to have direct access to the CSW "pkgutil". Note also that the order of the dirs for the superuser is different.

When done, it is recommended to change the permissions back to read-only:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u-w /etc/default/login</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> u-w /etc/default/su</strong>
</pre>
Logout and login back (or restart the machine) for the settings to take effect.

Next initialize the CSW database with the "-U" parameter:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgutil</span> -U</strong>
=&gt; Fetching new catalog and descriptions (http://mirror.opencsw.org/opencsw/testing/i386/5.10) if available ...
==&gt; 3986 packages loaded from /var/opt/csw/pkgutil/catalog.mirror.opencsw.org_opencsw_testing_i386_5.10
</pre>
Then the status of packages can be checked, for example:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgutil</span> -a vim</strong>
common               package              catalog                        size
gvim                 CSWgvim              8.0.238,REV=2017.01.30       1.2 MB
vim                  CSWvim               8.0.238,REV=2017.01.30       1.1 MB
vimrt                CSWvimrt             8.0.238,REV=2017.01.30       9.0 MB
</pre>
And a particular package installed:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgutil</span> -y -i vim</strong>
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
=&gt; Fetching CSWcommon-1.5,REV=2010.12.11 (1/11) ...
=&gt; Fetching CSWterminfo-rxvt-unicode-9.20,REV=2014.10.31 (2/11) ...
...
Installation of &lt;CSWvim&gt; was successful.
</pre>
See also here for additional details: [OpenCSW - Getting started](https://www.opencsw.org/manual/for-administrators/getting-started.html)

### Adding a secondary disk

<h4>**1. Preparing the disk partition**</h4>
A secondary drive can be formatted/partition created by using the "format" command:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">format</span></strong>
Searching for disks...
Inquiry failed for this logical diskInquiry failed for this logical diskdone

AVAILABLE DISK SELECTIONS:
       0. c0d0 &lt;▒x▒▒▒▒▒▒▒▒▒@▒▒▒ cyl 2607 alt 2 hd 255 sec 63&gt;
          /pci@0,0/pci-ide@7,1/ide@0/cmdk@0,0
       1. c0d1 &lt;▒x▒▒▒▒▒▒▒▒▒@▒▒▒ cyl 13052 alt 2 hd 255 sec 63&gt;
          /pci@0,0/pci-ide@7,1/ide@0/cmdk@1,0
Specify disk (enter its number):
</pre>
If the secondary disk was newly added after the installation, it might not show in the list. In that case, a "devfsadm" cleanup is necessary:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">devfsadm</span> -C -c disk -v</strong>
devfsadm[429]: verbose: symlink /dev/dsk/c0d1s0 -&gt; ../../devices/pci@0,0/pci-ide@7,1/ide@0/cmdk@1,0:a
devfsadm[429]: verbose: symlink /dev/dsk/c0d1s1 -&gt; ../../devices/pci@0,0/pci-ide@7,1/ide@0/cmdk@1,0:b
...
</pre>
See here for further troubleshooting details: [How to make Solaris rescan disk info after hotswap?](https://serverfault.com/questions/556929/how-to-make-solaris-rescan-disk-info-after-hotswap)

In the "format" utility, select the particular disk (in the above case, disk number 1):
<pre style="color: #bbbbbb; background-color: #000000;">Specify disk (enter its number): <span style="color: #ffff00;">**1**</span>
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
<strong style="color: #ffffff;">format&gt;</strong>
</pre>
Run the "fdisk" command. If the disk is new, fdisk will ask to create a new Solaris partition (enter 'y' to create the default partition):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">format&gt; <span style="color: #ffff00;">fdisk</span></strong>
No fdisk table exists. The default partition for the disk is:

  a 100% "SOLARIS System" partition

Type "y" to accept the default partition,  otherwise type "n" to edit the
 partition table.
<strong style="color: #ffffff;"><span style="color: #ffff00;">y</span></strong>
<strong style="color: #ffffff;">format&gt;</strong>
</pre>
Insert 'p' to enter the "partition" menu:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">format&gt; <span style="color: #ffff00;">p</span></strong>

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
<strong style="color: #ffffff;">partition&gt;</strong>
</pre>
The partition list can be printed by the "print" command:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">partition&gt; <span style="color: #ffff00;">print</span></strong>
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

<strong style="color: #ffffff;">partition&gt;</strong>
</pre>
Enter "0" to edit the first partition:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">partition&gt; <span style="color: #ffff00;">0</span></strong>
Part      Tag    Flag     Cylinders         Size            Blocks
  0 unassigned    wm       0                0         (0/0/0)             0

Enter partition id tag[unassigned]:
</pre>

- The _**partition id tag** _and **_flags_ **can be left as is (just press 'Enter').
- The _**starting cylinder** _will have the value of "3" (see in the print output above - there are two partitions "boot" and "alternates" taking together 3 cylinders, therefore we will use 3 so that the new partition is put after those two).
- The **_partition size_ **needs to be calculated as the total number of cylinders (according to the previous "print" output - "13050" in the above case) minus the star cylinder (3), so it will be 13047c (the 'c' at the end denotes the number specifying the cylinder count).

<pre style="color: #bbbbbb; background-color: #000000;">Enter partition id tag[unassigned]:
Enter partition permission flags[wm]:
Enter new starting cyl[0]: <strong style="color: #ffff00;">3</strong>
Enter partition size[0b, 0c, 3e, 0.00mb, 0.00gb]: <strong style="color: #ffff00;">13047c</strong>
<strong style="color: #ffffff;">partition&gt;</strong>
</pre>
The status can be checked by "print" again:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">partition&gt; <span style="color: #ffff00;">print</span></strong>
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

<strong style="color: #ffffff;">partition&gt;</strong>
</pre>
The partition map and label is written back to the disk with the "label" command:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">partition&gt; <span style="color: #ffff00;">label</span></strong>
Ready to label disk, continue? <strong style="color: #ffff00;">y</strong>

<strong style="color: #ffffff;">partition&gt;</strong>
</pre>
Exit by using the "q" command (twice - exit the "partition" and "format"):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">partition&gt; <span style="color: #ffff00;">q</span></strong>
<strong style="color: #ffffff;">format&gt; <span style="color: #ffff00;">q</span></strong>
<strong style="color: #ffffff;">#</strong>
</pre>
<h4>**2. Formatting the partition to ZFS and adding to the disk pool**</h4>
This is done by using the "zpool" command:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">zpool</span> create -f srv c0d1s0</strong>
</pre>
In the above, the "c0d1s0" is the first partition ("s0") of the secondary disk ("d1") on the first controller ("c0"). The "-f" parameter specifies the mount point of the new pool ("/srv" in this case).

The successful creation of the pool can be checked by the "df" command:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">df</span> -h</strong>
Filesystem             size   used  avail capacity  Mounted on
rpool/ROOT/s10x_u11wos_24a
                        20G   3.7G    14G    22%    /
/devices                 0K     0K     0K     0%    /devices
...
<strong style="color: #00ff00;">srv                     98G    31K    98G     1%    /srv</strong>
</pre>
In case of a mistake, the pool can be removed by:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">zpool</span> destroy srv</strong>
</pre>
(where "srv" is the name of the pool created before)

### Jenkins node setup

Creating the special user for the Jenkins node (Jenkis slave can run under any user, but it is generally recommended to have a separate user running the Jenkins slaves):
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">useradd</span> -m -d /srv/jenkins -s /usr/bin/bash jenkins</strong>
<strong style="color: #ffffff;"># <span style="color: #ffff00;">passwd</span> jenkins</strong>
New Password:
Re-enter new Password:
passwd: password successfully changed for jenkins
</pre>
Note the setting of the home directory for the Jenkins user, which points to the "/srv/jenkins" on a separate data drive - the Jenkins slave workspace data will be kept there.

The most convenient option is to make the Jenkins node a SSH slave, which is maintained by the Jenkins master automatically. For it to work, the Jenkins master SSH key needs to be inserted to the Jenkins node user <tt style="color: #4682b4; background-color: #fff8dc;">"~/.ssh/authorized_keys"</tt> file.

On the Jenkins master:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;">[<span style="color: #00ff00;">jenkins@master</span>]$ <span style="color: #ffff00;">cat</span> <span style="color: #ffffff;">~/.ssh/id_rsa.pub</span> | <span style="color: #ffff00;">ssh</span> <span style="color: #ff9900;">jenkins@test-sol1</span> '<span style="color: #ffff00;">mkdir</span> -p .ssh; <span style="color: #ffff00;">cat</span> &gt;&gt; .ssh/authorized_keys'</strong></pre>
Resetting the file permissions on the Solaris node:
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">chmod</span> 700 /srv/jenkins/.ssh
# <span style="color: #ffff00;">chmod</span> 600 /srv/jenkins/.ssh/authorized_keys</strong></strong></pre>
With this setup the Solaris machine should be prepared to be used as SSH Jenkins slave.

### Installing additional development tools

When using the Solaris machine for C/C++ development (e.g. as a Jenkins node), some additional tools are usually needed (_**Git**_, _**CMake**_, _**SVN** _etc.). Many of those are provided by the [OpenCSW repository](#extras-csw).
<h4>**Installing _git_**:</h4>
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgutil</span> -y -i git</strong>
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
Installation of &lt;CSWgit&gt; was successful.
<strong style="color: #ffffff;"># <span style="color: #ffff00;">which</span> git</strong>
/opt/csw/bin/git
<strong style="color: #ffffff;"># <span style="color: #ffff00;">git</span> --version</strong>
git version 2.3.1
</pre>
<h4>**Installing _CMake_**:</h4>
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgutil</span> -y -i cmake</strong>
Solving needed dependencies ...
Solving dependency order ...
17 CURRENT packages:
        CSWcacertificates-20160830,REV=2016.08.30
        CSWcas-migrateconf-1.50,REV=2015.01.17
...
Installation of &lt;CSWcmake&gt; was successful.
<strong style="color: #ffffff;"># <span style="color: #ffff00;">which</span> cmake</strong>
/opt/csw/bin/cmake
<strong style="color: #ffffff;"># <span style="color: #ffff00;">cmake</span> --version</strong>
cmake version 3.4.3

CMake suite maintained and supported by Kitware (kitware.com/cmake).
</pre>
<h4>**Installing _Subversion (SVN)_**:</h4>
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgutil</span> -y -i subversion</strong>
Solving needed dependencies ...
Solving dependency order ...
17 CURRENT packages:
        CSWbdb48-4.8.30,REV=2010.12.06_rev=p0
        CSWcommon-1.5,REV=2010.12.11
        CSWggettext-data-0.19.8,REV=2016.09.08
...
Installation of &lt;CSWsubversion&gt; was successful.
<strong style="color: #ffffff;"># <span style="color: #ffff00;">which</span> svn</strong>
/opt/csw/bin/svn
<strong style="color: #ffffff;"># <span style="color: #ffff00;">svn</span> --version</strong>
svn, version 1.9.4 (r1740329)
   compiled Sep 19 2016, 14:50:04 on i386-pc-solaris2.10
</pre>
<h4>**Installing _GCC 5_**:</h4>
<pre style="color: #bbbbbb; background-color: #000000;"><strong style="color: #ffffff;"># <span style="color: #ffff00;">pkgutil</span> -y -i gcc5g++</strong>
Solving needed dependencies ...
Solving dependency order ...
11 CURRENT packages:
        CSWcas-texinfo-1.50,REV=2015.01.17
        CSWcommon-1.5,REV=2010.12.11
...
Installation of &lt;CSWgcc5g++&gt; was successful.
<strong style="color: #ffffff;"># <span style="color: #ffff00;">which</span> g++</strong>
/opt/csw/bin/g++
<strong style="color: #ffffff;"># <span style="color: #ffff00;">g++</span> --version</strong>
g++ (GCC) 5.2.0
Copyright (C) 2015 Free Software Foundation, Inc.
This is free software; see the source for copying conditions.  There is NO
warranty; not even for MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
</pre>
(similar for GCC v.4)

## Resources and references

{% include abbrev domain="computers" %}
