---
title: "Minimal installation of Solaris 10 with Sun Studio"
toc: true
sidebar:
  nav: "sol-10"
hidden: true
categories:
  - Operating Systems
  - UNIX
---

## 2. System installation

### 2.1. Booting the installation media

Insert the installation media and start the machine.

If there is already an OS installed on the machine, you might need to change the **order of boot devices** in the BIOS, so that the CD/DVD device will be booted first before the hard drive.
However, if the hard drive is empty, leave the CD/DVD device as secondary boot device after the hard drive, so that the installer does not start again after the installation is complete.

When creating a virtual machine, the recommended size of the HDD to create is 20 GB (see later in the article, how to add additional disk for extra data).

When booted, the installer boot manager appears:

{% include figure image_path="/assets/images/posts/solaris-10/boot.png" alt="Solaris boot" caption="Solaris 10 Installer Boot Manager" %}

Here you can either start the installation, or press the 'e' key to edit the boot commands first.

**Disabling the USB (EHCI and UHCI) support**

If the Solaris 10 is used under QEMU (for example in [QNAP NAS Virtualization Station](https://www.qnap.com/solution/virtualization-station/en-us/), which uses the QEMU under the hood), it is recommended to **disable the USB EHCI and UHCI support**, as those are not virtualized well and Solaris 10 then spends a considerable amount of time trying to detect the devices (which is not able to do at the end anyway).

To disable, press the 'e' key upon the above boot screen, then press it again to edit the first line (starting with "kernel$") and add the following to the end of the line:

```
,disable-ehci=true,disable-uhci=true
```

Then press 'Enter' to return back, and the 'b' key to start the boot process.

### 2.2. Starting the installer

After the installer boots up, select the installation option 4 to start the installation in the "_Interactive Text_" mode:

{% include figure image_path="/assets/images/posts/solaris-10/install-1-select.png" alt="Solaris install" caption="Installation type selection" %}

The option 3 or 4 is required when the **[ZFS](https://en.wikipedia.org/wiki/ZFS)** is to be used, which is recommended (supports wide range of features, like snapshots etc.).

Next is setting of the keyboard layout - note that the choice is not confirmed by the 'Enter', but by the 'F2' key:

{% include figure image_path="/assets/images/posts/solaris-10/install-2-keyboard.png" alt="Select keyboard" caption="Choosing the keyboard layout" %}

Bunch of next dialogs follow, before setting up the network.

### 2.3. Initial network settings

The network connectivity setup:

{% include figure image_path="/assets/images/posts/solaris-10/install-3-network.png" alt="Network setup" caption="Network connectivity setup" %}

Normally the system should be using **[DHCP](https://de.wikipedia.org/wiki/Dynamic_Host_Configuration_Protocol)**, which assigns the IP address to the machine accordingly.

If the static IP address is needed, it can be setup there as well.
In that case, you'll need to choose the host name of the machine, the IP address and provide the gateway ("router") IP address.

In the case of _DHCP_ selection, the installer does not ask for the host name, and the system will try to get the host name over the _DHCP_ as well.
However most of the _DHCP_ servers do not provide the host names nowadays (would require a special setup on the _DHCP_ server), so the host name will be [set later]({% post_url 2017-06-11-minimal-install-solaris-10-with-sun-studio-3 %}#37-network-setup) after the basic installation is done.

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

For the minimal installation, select the option: "**Reduced Networking Core System Support**".  
This is to perform the _minimal text-based_ install.

{% include figure image_path="/assets/images/posts/solaris-10/install-4-minimal.png" alt="Software bundle" caption="Software bundle selection" %}

Do not press the `F2` yet, but press the `F4` key instead to **customize and add extra packages**, required for using the SSH, installing the Solaris Studio etc.

Alternatively you can select the "**Developer System Support**" (Sun Studio installer actually complains if this or the "_Entire Distribution_" option is not selected, but if the required packages are added manually, the Sun Studio will install and work properly).
But as you can see, this option install a lot more packages, including the full graphical desktop, which you might not want or need for the minimal system.

When selected the minimal install option and `F4` to customize, add the following packages (some just for convenience, some required for the Sun Studio or Java):

- **A Windows SMB/CIFS fileserver for UNIX** (the entire group):
  - Required for: _NetBios name resolution_ (from Windows)
  - If you do not need to access the machine via hostname from Windows, does not need to be installed
  - Can be also installed later if needed
- **Apache Standard C++ Library**:
  - <span style="color:red;">Required for: The Oracle Solaris Studio</span>
- **BIND DNS Name server and tool**:
  - Required for: _DNS name resolution_
  - Installs the "nslookup" tool for troubleshooting the networking issues
- **Basic IP commands (Usr)**:
  - Installs the "ping" command
- **CPU Performance Counter driver and utilities** (open the group by 'Enter' on the '>' before the group name)
  - **CPU Performance Counter libraries and utilities**:
    - <span style="color:red;">Required for: The Oracle Solaris Studio</span>
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
    - <span style="color:red;">Required for: The Oracle Solaris Studio</span>
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
  - <span style="color:red;">Required for: The Oracle Solaris Studio</span>
- **Motif RunTime Kit**
  - <span style="color:red;">Required for: The Java 8 SDK</span>
- **PICL Framework, Libraries, and Plugin Modules** (the entire group)
- **PICL Header Files (Usr)**
  - <span style="color:red;">Required for: The Oracle Solaris Studio</span>
- **Portable layout services for Complex Text Layout support**
  - <span style="color:red;">Required for: The Java 8 SDK</span>
- **Programming tools and libraries** (open the group)
  - **Math & Microtasking Libraries (Usr)**
  - **Math & Microtasking Library Headers & Lint Files (Usr)**
  - **Solaris Bundled tools**
  - **Solaris cpp**
    - <span style="color:red;">Required for: The Oracle Solaris Studio</span>
- **Secure Shell** (the entire group):
  - Installs the SSH server for remote login
- **SunOS Header Files**:
  - <span style="color:red;">Required for: The Oracle Solaris Studio</span>
- **rsync**:
  - Not essential, but can be handy

{% include figure image_path="/assets/images/posts/solaris-10/install-5-sw-selection.png" alt="Software selection" caption="Additional software selection" %}

After selecting all the additional software, press `F2` to continue.

The package manager will select some additional dependencies, press `F4` to preserve all the required dependencies:

{% include figure image_path="/assets/images/posts/solaris-10/install-6-dependencies.png" alt="Package dependencies" caption="Preserve package dependencies" %}

See here for the list of Solaris Studio requirements: [Oracle Solaris Studio 12.4 - Required System Software Packages](http://docs.oracle.com/cd/E37069_01/html/E37070/gnzpf.html#scrolltoc)

### 2.6. Install disk selection

In the next step, the disk for the installation can be selected:

{% include figure image_path="/assets/images/posts/solaris-10/install-7-disk.png" alt="Select disk" caption="Disk selection" %}

Here you can confirm the defaults and continue by `F2`, or press `F4` and make changes.

Next is to configure the **ZFS** settings (if the ZFS was selected before).
I'll usually leave the defaults:

{% include figure image_path="/assets/images/posts/solaris-10/install-8-zfs.png" alt="ZFS options" caption="ZFS settings" %}

Afterwards there are few other confirmation dialogs and the installation begins.
Wait for the installation to complete and restart.
If you changed the order of boot devices in the BIOS to boot from CD/DVD device first, you'll need to remove the installation media or change the boot order back to start from the HDD first (recommended), otherwise the installer will start again.

On the first boot, the machine should start into the text console (as we didn't install the graphical desktop with the minimal installation), where you can login into the root account with the password you set during the installation:

{% include figure image_path="/assets/images/posts/solaris-10/install-9-login.png" alt="Login screen" caption="The initial login screen" %}

The default shell is "_/bin/sh_", you can type "`bash`" to switch to the bash shell (if you installed it).

Note that the **hostname** is reported as "_unknown_", because the _DHCP_ server didn't configure it (but you can see that the IP address has been configured over the _DHCP_).
We will [set the host name manually]({% post_url 2017-06-11-minimal-install-solaris-10-with-sun-studio-3 %}#37-network-setup) later (alternatively the _DHCP_ server would have to be configured to know which host name it should push for the particular machine).

[Next: Post-install setup]({% post_url 2017-06-11-minimal-install-solaris-10-with-sun-studio-3 %}){: .btn .btn--info}
{: .align-right}

{% include abbrev domain="computers" %}
