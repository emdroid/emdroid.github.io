---
title: "Unlocking Linux encrypted root from network / USB"
header:
  teaser: /assets/images/posts/network_lock.png
toc: true
categories:
  - Operating Systems
  - Linux
tags:
  - root
  - encryption
  - network
  - usb
---

![Lock](/assets/images/posts/network_lock.png){: .align-left .img-thumbnail}

Unattended booting of Linux encrypted root via network and USB fallback.

<!-- more -->

## 1. Overview

In the previous ["Project Ares" articles]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}) I've shown how to setup the encrypted root partition with ZFS.

However for unlocking the root partition during the **unattended boot** the key had to be **present in the initrd image** on the boot partition, where it is freely accessible.
That is a security risk, because basically anyone with the physical access to the machine (or just the disk) can get the encryption key from there.

This article builds on top of that and shows 2 safer possibilities of unlocking the root partition:
1. **Unlocking from the network** - in particular using the Azure blob storage and limiting the access to specific IP address(es)
2. **Unlocking via USB flash drive** (the fallback option)

Note that the requirement to support the **unattended boot** still holds, for the NAS server I was building it was essential to not have to provide a password on every boot (especially because of the machine being "headless", i.e. not attached to any screen or keyboard).

It is technically still possible to solve this by running a SSH server during the bootup to wait for the password (e.g. by using the [dropbear server](https://linux.die.net/man/8/dropbear)), but that is still not great if other users should be able to start the machine as well.

In addition the unattended bootup is important to eventually **shut down and start up the machine regularly** (e.g. on a schedule during the night when nobody is using it).
And having to provide the password every day when the machine is scheduled to wake up is quite an inconvenience to say the least.

{% capture notice_contents %}
**<a name="ubuntu_note">Warning</a>**:

Note that the instructions are specific to the **Ubuntu distribution** and might not work for other distributions where I didn't test them.
{% endcapture %}

{% include notice level="warning" %}

## 2. Security considerations

The whole setup is constrained by the intention of running the boot **unattended** (without having to unlock manually), which brings some compromises into the play.

By default, the root partition key will be downloaded from the network (Internet), allowing to boot the machine without any manual intervention. Then there will be the fallback option of using an USB flash drive to boot in case the network connection is not available (or the Internet access is down), which will still be manual.

If that is sufficient from the security point of view depends on your threat model.
This setup will eventually protect the data in these cases:
- the machine or a disk is stolen
- the disk fails and needs to be replaced in warranty  
(and you are unable to securely erase it because of the failure)

Note in particular that the **root partition key is no longer stored unencrypted anywhere on the disk** (and in fact it doesn't need to be - and shouldn't be - stored on the machine at all).

The keys for any other eventual encrypted partitions ("_/home_", data etc.) can still be stored on the (encrypted) root partition and used from there (as the root partition itself is encrypted).
You could eventually do the same setup for the other partition as well, but that is not entirely necessary and will not be part of this article.

### 2.1. Default: Using remote key from the Internet

The key will be fetched from the Internet automatically by default.
There are a few options of doing that, some are more and some less secure (and some that are paid and some free).

In this case the **Azure Blob Storage** will be used, which offers quite interesting set of features for free:
1. **Can enforce HTTPS access**:
  - this increases the security because of encrypting the data exchanged over the Internet
2. **Allows to limit the access for specific IP addresses**:
  - you can limit the IP address(es) to access the key file to your network(s)
  - note that this is limited to the public IP address, that means if you don't have your own public IP and are part of a bigger ISP network, anyone from that network can still access the file
3. **Allows to use the SAS token**:
  - adds additional security by using a token that "only you know" (so even in the above case of having an aggregate IP only you know the token)
  - however note that the token still needs to be accessible inside the initrd image (unencrypted) so the boot process can see it to download the file
  - that means if the machine is stolen, it can eventually still be booted from anywhere inside the broader ISP network (if having the shared public IP address)

**Q: Why over the Internet and not locally?**

You could eventually think to put the key only somewhere **inside your local network**.
This has both **advantages and disadvantages** from the security point of view:
- the advantage is that the key isn't accessible anywhere outside of your network
- however, if the other network device you get the key from (e.g. your router or some other PC in the network) is stolen together with the machine, the attacker has access to both the machine and the key (indefinitely)
- so if the key is on the Internet blob storage instead, you might eventually be **able to delete the key** from there before the attacker is able to figure out how to retrieve the key

### 2.2. Fallback: Using USB flash drive

The article also shows the setup to unlock the root using the USB flash drive.
This will be the fallback option if the key retrieval from the network doesn't work for any reason (e.g. no network access or the Internet is down).

Note that this fallback option **will require a manual intervention**, so it is provided as a manual step to still be able to unlock the machine under circumstances.

To maintain the safety, it is recommended to **only use this fallback option when it is really needed** (i.e. _don't keep the USB drive with the key connected all the time_, as it would most likely be eventually taken together with the machine then, obviously).

The idea is to put the key on (any currently available) USB flash drive just for the particular one-time machine startup, and remove the USB drive and erase the key from it once not needed any more (or at least keep it separated somewhere safe when not using it).

## 3. Generating and preparing the key

The key can still be generated the same way as with the standard setup (as in the [Project Ares]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}#keys_generate) case).

However, you should skip copying the key to the root drive (it still could be copied, but it's not necessary and the keyfile doesn't appear anywhere on the target machine then).

In addition you should [skip adding the keyfile to the initramfs]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}#keys_initramfs).

That is for both the root and the eventual [swap key]({% post_url 2020-08-30-project-ares-part-5-ubuntu-on-zfs-raid %}#keys_swap) (if the hibernation is used).

### 3.1. Generating the keys

Just to recap, the encryption keys can be generated by:

**a) ZFS native encryption setup**:

```bash
# generate the key
mkdir -p /etc/crypt/zfs/init
dd if=/dev/urandom bs=32 skip=4 count=1 iflag=fullblock \
    | hexdump -ve '/1 "%02x"' \
    > /etc/crypt/zfs/root.key

# print the key for backing up on the primary PC
cat /etc/crypt/zfs/root.key
```

**b) LUKS encryption setup**:

```bash
# generate the key
mkdir -p /etc/crypt/init
dd if=/dev/urandom bs=512 skip=4 count=4 iflag=fullblock \
    | base64 -w 0 \
    > /etc/crypt/root.key

# print the key for backing up on the primary PC
cat /etc/crypt/root.key
```

**c) The swap LUKS encryption key** (for the eventual hibernation setup):

```bash
# generate the key
dd if=/dev/urandom bs=512 skip=4 count=4 iflag=fullblock \
    | base64 -w 0 \
    > /etc/crypt/swap.key

# print the key for backing up on the primary PC
cat /etc/crypt/swap.key
```

Note that the `"hexdump"` / `"base64"` is used so that the key will not contain "unprintable" characters which can cause issues when transferring and using them.

### 3.2. Saving the keys

You shouldn't forget to copy the files somewhere safe on your other (maybe primary) PC, ideally into an **encrypted location** (e.g. using VeraCrypt on Windows etc.).

You might need to use VM or Docker image to setup the USB drive under Windows, but make sure the keys are also stored safely there (e.g. the VM itself can be on an encrypted drive).
Although you might be also able to use the [dd for Windows](http://www.chrysocome.net/dd) or WSL (I didn't test those options yet).

To copy the key files from Linux you can either use the [WinSCP](https://winscp.net/eng/download.php) on Windows, the `"scp"` command on Linux or the clipboard after printing the key on the screen by e.g. `"cat /etc/crypt/zfs/root.key"`.

{% capture notice_contents %}
**<a name="key-newline-info">Warning</a>**

If copying the file via clipboard make sure the file is stored without a newline at the end - the key file **should not contain any newlines** which in particular create issues between Linux and Windows environments.

That is also the reason why the `"-w 0"` is used for the `"base64"` conversion, as that prevents adding the automatic newlines (that the `"base64"` does by default).
{% endcapture %}

{% include notice level="warning" %}

## 4. Setting up the Azure Blob Storage

The following blob storage setup in particular can be done with the free [Microsoft Azure](https://azure.microsoft.com/) account (without having to pay any fees).

### 4.1. Creating the blob storage

**a) Creating the virtual network**:

Here we'll create the virtual network for the setup:

- on the Home screen, select "**More Services**"
- select "**Networking**" in the "**Categories**" panel

- select "**Virtual networks**", click "**Add**":
  - _Subscription_: select your Azure subscription
  - _Resource group_: select the resource group  
  (or create a new one via "Create new")
  - _Name_: choose the name of your virtual network
  - _Region_: select the one closest to your location  
  (usually selected by default)
- continue to "**Review + create**":
  - will use defaults for everything else
- review and press "**Create**":

**b) Creating the storage account**:

- on the Home screen, select "**More Services**"
- in the "**Categories**" panel select the "**Storage**"

- select "**Storage Accounts**", click "Add":
  - _Subscription_: select your current Azure subscription
  - _Resource group_: select the resource group  
  (or create a new one via "Create new")
  - _Storage account name_: choose the name for your storage
  - _Location_: select the one closest to your location  
  (usually selected by default)
  - _Performance_: Standard  
  (the key is a very small amount of data, so doesn't need the Premium)
  - _Accound kind_: BlobStorage
  - _Replication_: Locally redundant (LRS)
  - _Blob access tier_: Hot
- continue to "**Networking**":
  - _Connectivity method_: Public endpoint (selected networks)
  - _Virtual network_: select the virtual network created before
- continue to "**Review + create**":
  - will use defaults for everything else
- review and press "**Create**"
- once the deployment is done, "**Go to resource**"

**c) Limiting the access to specific IP address(es)**:

Now we want to limit the access to your IP address:
- in the resource options, select "**Firewalls and virtual networks**" in the left panel
- in the "**Firewall**" section:
  - select the "_Add your client IP address_" option
  - you can eventually add any other IP addresses that should have access to the storage
- in the "**Exceptions**" section:
  - unselect the "_Allow trusted Microsoft services to access this storage account_" option
- click "**Save**" on top of the page

**d) Adding the storage container**:

- select the Storage account "**Containers**" option
- click on the "**+ Container**" to add a new container
  - _Name_: choose the name of the container for the keys
  - _Public access level_: Private (no anonymous access)
- press "**Create**"

**e) Uploading the key(s)**:

- open the container created before
- click on the "**Upload**" button to upload files:
  - _Files_: browse for the key file to be uploaded
- expand the "**Advanced**" options:
  - _Upload to folder_: eventually set a subfolder for your particular machine  
  (recommended - although you can also separate the machines by containers)
- press "**Upload**"

Repeat for all keys that are needed (root, swap for hibernation).

{% capture notice_contents %}
**<a name="upload_note">Note</a>**:

This only needs to be done for the keys that are **required for mounting the root partition** or any other partition at bootup (e.g. the hibernation swap).

But it is not necessary for volumes that can be mounted with the key stored on the root partition (like "_/home_", data drives etc.).
{% endcapture %}

{% include notice level="info" %}

### 4.2. Generating the SAS access token

The SAS token can be used to access the file "privately" without exposing the regular user account keys.

To generate a new SAS token:
- click on the uploaded key
- select the "**Generate SAS**" option:
  - _Permissions_: Read
  - _Expiry_: select the expiration date of the token
    - I'd recommend at least +2 years  
    (I've set mine to +20 years)
    - you'll basically need to create a new token and update it in the machine boot script after the current token expires
    - note that you cannot revoke the SAS token (so take that into accout when setting the expiration), but you can eventually remove the whole key file if needed
    - in particular there is still the network IP protection, so even if the attacker has the token, will still not be able to access the key unless in the same network
  - _Allowed IP addresses_: keep empty  
  (could also select your IP address, but this was already done on the Storage account level)
- press "**Generate SAS token and URL**"

- save the URL shown in the "**Blob SAS URL**" box to be used later to load the key:
  - it will look something like this:  
  "<tt>https://mystore.blob.core.windows.net/mycontainer/dir/root.key?sp=r&st=2020-09-20T19:55:11Z&se=2020-09-21T03:55:11Z&spr=https&sv=2019-12-12&sr=b&sig=0vQeTFxt1Oi%2BoHkeK%2F21GV7ECwbRcHPprS9ckXHt398%3D</tt>"
  - note that it is only shown when generated, and not retrievable any more after you navigate out of the page  
  (but you can always generate a new token)

This again needs to be done for each separate key (in case you want to use multiple keys, e.g. one for the root partition and one for the hibernation swap).

### 4.3. Testing the access

- try to **access the SAS URL** in the browser: should be able to access the file
- try to **access the file without SAS** (via the raw URL as shown in the "Overview" of the key file properties): should not be allowed
- eventually try to **access the file from a different network** (VPN etc.) using the SAS token URL: should not be allowed  
(as being accessed from a different IP address)

## 5. Setting up the USB fallback option

We will store the key on the USB drive when needed, using the **RAW access**.
That has the following advantages:
- will be accessing the USB drive directly in the RAW mode, so it **doesn't need to be mounted**  
(doesn't need to fiddle with the mount points etc.)
- the key will be "hidden" in the unused disk area outside the regular partition(s), so it **will not be visible** (like a file on the disk) - will be just a bunch of bytes "somewhere" on the drive

### 5.1. Enabling the USB drive alias

This is necessary so that the USB flash drive will appear accessible under specific "_/dev/_" alias path, in particular as "_/dev/usbkey%n_" where "_%n_" is a sequential number (in case multiple USB drives are connected).

This allows the script to know all the USB devices attached (would need to examine all disks otherwise).

There are two options, either enabling all USB drives or only a selected one.

**a) Enabling for all USB drives**:

- this UDEV rule enables the key to be retrieved from any USB drive:

```bash
# set the rule for all USB disks
echo 'SUBSYSTEMS=="usb", DRIVERS=="usb", SYMLINK+="usbkey%n"' \
    > /etc/udev/rules.d/99-custom-usb.rules

# reload the rule
udevadm control --reload-rules
```

**b) Enabling only a specific USB drive**:

- this enables just a particular USB flash drive to be used as the "unlocking" HW key
- note however that although this might be seen as "more secure" option, there isn't a big difference really so I prefer the other option (allowing any USB drive) and deleting the key from the drive when no longer needed

For determining the specific USB drive properties we'll need to:
- insert the USB drive
- use the `"udevadm"` command to list its properties, for example:

```bash
# show the USB drive properties, here for /dev/sdf
udevadm info -a -p $(udevadm info -q path -n /dev/sdf) | less

# search the particular attributes in the listing
# and use them for the rule, for example:
echo 'SUBSYSTEMS=="usb", DRIVERS=="usb", ATTRS{manufacturer}=="Corsair", ATTRS{product}=="Voyager SliderX1", ATTRS{serial}=="AAWKGXCFDFRRFBWG", SYMLINK+="usbkey%n"' \
    > /etc/udev/rules.d/99-custom-usb.rules

# reload the rule
udevadm control --reload-rules
```

**c) Testing the setup**:

- try to insert the USB drive (if already inserted, disconnect and re-connect again)

- check that the drive is available under the new alias:

```bash
ls -al /dev/usbkey*
```

### 5.2. Copying the key file to the USB drive

- the keyfile will be copied to the USB flash drive in the RAW mode, to the unused area before the first partition
- this way we don't need to "truly" mount the USB drive (i.e. mount the filesystem) as we can access the drive directly
- this generally only works on Linux (you can eventually use a VM or a Docker container to do this on Windows, perhaps also WSL)

**a) The ZFS native encryption setup**:

The copy command uses the [Project Ares]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}) setup, that is:
- the key is 32 B (256-bit) long (bs=32 count=1)
- the original key is dumped to "hex", thus converting back to binary using the `"xxd"` command
- writing to the 4th raw sector of the USB drive  
(sector size is 512 B, therefore seek = 512 / 32 * 4 = 64)

```bash
# write the key to the 4th sector of the disk
# (convert back from hex to binary)
cat /etc/crypt/zfs/root.key | \
    xxd -r -p | \
    dd of=/dev/usbkey bs=32 seek=64 count=1
```

**b) The LUKS root encryption setup**:

The copy command uses the original [Project Ares]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}) setup for LUKS, that is:
- the key is 2 kB long (bs=512 count=4)
- the original key is dumped to "Base64" (converting back to binary)
- writing to the 4th raw sector of the USB drive (seek=4)

```bash
# write the key to the 4th sector of the disk
# (converting from Base64 back to binary format)
cat /etc/crypt/init/root.key | \
    base64 -d | \
    dd of=/dev/usbkey bs=512 seek=4 count=4
```

**Additional keys**

If you have any additional keys (like for example one key for the root partition and another one for the hibernation swap), you can still store both keys, but the other keys **need to go to a different location** on the disk (so both the keys can be there side by side).

This can be done by changing the `"seek"` position of the `"dd"` command - for example with the extra swap file key (in the native ZFS setup, where the ZFS root key has a different format than the swap LUKS key):

```bash
# write the key to the 5th sector of the disk
# (converting from Base64 back to binary format)
cat /etc/crypt/init/swap.key | \
    base64 -d | \
    dd of=/dev/usbkey bs=512 seek=5 count=4
```

(in the case of the LUKS setup and a separate root/swap keys you could use `"seek=8"` to write to the 8th sector)

The alternative is to setup a **separate "key vault" partition** that is unlocked with a single master key on boot and which holds all the other encryption keys (I will eventually explain it in more detail in some follow up article).

### 5.3. Clearing the key

It is recommended to remove (clear) the key(s) from the USB drive when no longer needed (otherwise there is the risk of the USB drive being eventually taken together with the machine).

This will simply overwrite the RAW key sectors by random data to remove the key:

```bash
# overwrite the key by random data
dd if=/dev/urandom of=/dev/usbkey bs=512 seek=4 count=4 iflag=fullblock
```

If having multiple keys (for example the extra hibernation swap key), you need to adapt the command to erase all of them, for example by extending the `"count=8"` (to remove the keys starting at the 4-th and the 8-th sector).

## 6. Setting up the boot script

This section explains the boot script setup to load the encryption keys from the network or the USB fallback.

### 6.1. The initramfs setup

**a) The initramfs modules**:

- some extra modules are needed for setting up the network connection and the USB device mounting

```bash
  cat <<EOF >> /etc/initramfs-tools/modules
sha256
aes-x86_64
aes_generic
crypto_api
dm-crypt
dm-mod
netboot
scsi_dh
usbcore
usbhid
usb_storage
EOF
```

- eventually check the setup

```bash
less /etc/initramfs-tools/modules
```

**b) The initramfs setup hook**:

- this is needed for copying the additional tools and setting up the network options in the initramfs (like the DNS and resolver)

```bash
  cat <<EOF > /etc/initramfs-tools/hooks/setup-boot
#!/bin/sh

PREREQS=""

case \$1 in
    prereqs) echo "\${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

# the commands needed
copy_exec $(which curl)
copy_exec $(which base64)
copy_exec $(which hexdump)
copy_exec $(which ping)

# the DNS libraries
add_dns "\${DESTDIR}/"

# the resolver configuration
copy_file config "$(readlink -f /etc/resolv.conf)" "/etc/resolv.conf"
EOF
```

- adjust the permissions:

```bash
chmod +x /etc/initramfs-tools/hooks/setup-boot
```

(ping is just added so that it can be eventually used to check the network status if something fails)

### 6.2. Network configuration on boot

- the initramfs module to configure/initialize the network during the boot phase:

```bash
  cat <<EOF > /etc/initramfs-tools/scripts/init-premount/configure-network
#!/bin/sh
# Initialize the network
PREREQ=""
prereqs()
{
    echo "\$PREREQ"
}
case \$1 in
prereqs)
    prereqs
    exit 0
    ;;
esac

. /scripts/functions

configure_networking
EOF
```

- adjust the permissions:

```bash
chmod +x /etc/initramfs-tools/scripts/init-premount/configure-network
```

### 6.3. Initramfs script to retrieve the keys

- this is used to load the keys and pass them to the boot system
- the scripts will be using the `"curl"` command to retrieve the key(s) from Azure BlobStorage, in particular the following command will be used (test before using it):

```bash
curl --insecure -s '<sas_token_url>'

# for example (use your own SAS token URL):
curl --insecure -s 'https://mystore.blob.core.windows.net/mycontainer/dir/root.key?sp=r&st=2020-09-20T19:55:11Z&se=2020-09-21T03:55:11Z&spr=https&sv=2019-12-12&sr=b&sig=0vQeTFxt1Oi%2BoHkeK%2F21GV7ECwbRcHPprS9ckXHt398%3D'
```

You might notice the `"insecure"` parameter:
- this is to skip the HTTPS certificate validation which is not available at the bootup phase in this setup
- it might be eventually possible to setup the SSL certificates as well, but I hadn't any luck with that yet  
(it would basically require to copy the certificates and the SSL curl modules into the initramfs, but appears to be a bit tricky)

**a) ZFS native encryption**:

- this will get the root partition key from the network or USB and load it into the ZFS manager so that the root partition can be unlocked

```bash
# set the SAS token URL
# (replace the <sas_token_url> by your URL)
# !!! IMPORTANT: Use the apostrophes to avoid special char interpretations !!!
ROOT_KEY_SAS='<sas_token_url>'

# set the root pool name
ROOT_POOL=rpool/ROOT/ubuntu


# create the ZFS key load script
  cat <<EOF > /etc/initramfs-tools/scripts/init-premount/unlock-root.sh
#!/bin/sh
# Load the root partition key.

load_az() {
    # eventual errors should go to stderr
    curl --insecure -s '${ROOT_KEY_SAS}'
}

load_usb() {
    [ -b /dev/usbkey ] || return 1

    # if device exists then output the keyfile from the usb key 
    dd if=/dev/usbkey bs=32 skip=64 count=1 | hexdump -ve '/1 "%02x"'
}

get_key() {
    load_az || load_usb
}

export PATH=/sbin:/usr/sbin:/bin:/usr/bin

get_key | zfs load-key ${ROOT_POOL}
EOF
```

- adjust the permissions:

```bash
chmod +x /etc/initramfs-tools/scripts/init-premount/unlock-root.sh
```

**b) LUKS encrypted root**:

This is working by providing a script that will be called by LUKS when mounting the drive (which is different from how the above ZFS native variant works).

Here we will need multiple scripts:
- script to retrieve the key
- initramfs module to copy the retrieval script into the initramfs
- changing the setup for the LUKS partition to use the key script instead of a pure key
- the key no longer needs be copied into the initramfs

_The root key script_:

```bash
# set the SAS token URL
# (replace the <sas_token_url> by your URL)
# !!! IMPORTANT: Use the apostrophes to avoid special char interpretations !!!
ROOT_KEY_SAS='<sas_token_url>'


# create the root key script
mkdir -p /etc/crypt/scripts

  cat <<EOF > /etc/crypt/scripts/unlock-root.sh
#!/bin/sh
# Load the root partition key.

load_az() {
    # eventual errors should go to stderr
    $(which curl) --insecure -s '${ROOT_KEY_SAS}'
}

load_usb() {
    [ -b /dev/usbkey ] || return 1
    
    # if device exists then output the keyfile from the usb key 
    $(which dd) if=/dev/usbkey bs=512 skip=4 count=4 | $(which base64) -w 0
}

get_key() {
    load_az || load_usb
}

get_key
EOF
```

- adjust the permissions:

```bash
chmod +x /etc/crypt/scripts/unlock-root.sh
chmod -R go-rwx /etc/crypt/scripts
```

_The initramfs module to copy the key load script_:

```bash
# the intramfs module
  cat <<EOF > /etc/initramfs-tools/hooks/crypt-scripts
#!/bin/sh

PREREQS=""

case \$1 in
    prereqs) echo "\${PREREQS}"; exit 0;;
esac

. /usr/share/initramfs-tools/hook-functions

mkdir -p \${DESTDIR}/etc/crypt || true
cp -ar /etc/crypt/scripts \${DESTDIR}/etc/crypt/
EOF

# adjust the permissions
chmod +x /etc/initramfs-tools/hooks/crypt-scripts
```

_Updating the crypttab_ (for the [Project Ares]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}) setup):

```bash
vim /etc/crypttab

# change the line:
zroot /dev/md2 /etc/crypt/init/root.key luks,discard,initramfs
# into:
zroot /dev/md2 none luks,keyscript=/etc/crypt/scripts/unlock-root.sh,discard,initramfs
```

- that means, use the `"keyscript"` option instead of the direct key option  
(which is changed to `"none"`)

**c) Hibernation swap LUKS setup**:

This is similar to the LUKS encrypted root - need to have the separate swap key script, and then the initramfs module to copy the script to the initramfs.

Eventually it might be better to have a "key vault" partition instead, that can be used to store all keys and only have a single master key (might be explained in some further article).

_The swap key script_:

```bash
# set the SAS token URL
# (replace the <sas_token_url> by your URL)
# !!! IMPORTANT: Use the apostrophes to avoid special char interpretations !!!
ROOT_KEY_SAS='<sas_token_url>'


# create the swap key script
mkdir -p /etc/crypt/scripts

  cat <<EOF > /etc/crypt/scripts/unlock-swap.sh
#!/bin/sh
# Load the swap partition key.

load_az() {
    # eventual errors should go to stderr
    $(which curl) --insecure -s '${ROOT_KEY_SAS}'
}

load_usb() {
    [ -b /dev/usbkey ] || return 1
    
    # if device exists then output the keyfile from the usb key 
    $(which dd) if=/dev/usbkey bs=512 skip=4 count=4 | $(which base64) -w 0
}

get_key() {
    load_az || load_usb
}

get_key
EOF
```

- adjust the permissions:

```bash
chmod +x /etc/crypt/scripts/unlock-swap.sh
chmod -R go-rwx /etc/crypt/scripts
```

_The initramfs module to copy the key load script_:

- use the same as for the root key script  
(i.e. skip if already created for root)

_Updating the crypttab_ (for the [Project Ares]({% post_url 2020-08-25-project-ares-part-4-ubuntu-on-zfs-encrypted-root %}) setup):

```bash
vim /etc/crypttab

# change the line:
swap /dev/md0 /etc/crypt/init/swap.key luks,discard
# into:
swap /dev/md0 none luks,keyscript=/etc/crypt/scripts/unlock-swap.sh,discard
```

- similar to the root partition above, use the `"keyscript"` option instead of the direct key option

### 6.4. Rebuilding the initramfs

- after the setup is done, the initramfs image needs to be rebuilt:

```bash
# re-create the initramfs image
update-initramfs -c -k all
# ignore the eventual cryptsetup warnings - should still work

# check if the cryptkeys are present in the initramfs
lsinitramfs -l /boot/initrd.img-<tab-complete-the-img-path> | less
# check for:
# - the "etc/crypt/scripts/..." files
# - no "etc/crypt/init/*.key" files present
```

- now you can reboot and test whether the machine boots up properly

## 7. Troubleshooting

### 7.1. The boot stops in the initramfs prompt

- this means that the key load wasn't successful for some reason
- if booting via the primary option (Internet), try the fallback option (USB)

- you can also try to call the scripts manually just to check the output for any error messages, for example:

```bash
# the ZFS native root encryption variant
# (loads the key)
/scripts/init-premount/unlock-root.sh

# the LUKS encrypted root variant
# (prints the key)
/etc/crypt/scripts/unlock-root.sh

# the LUKS encrypted swap
# (prints the key)
/etc/crypt/scripts/unlock-swap.sh
```

- if neither option is booting, check the error messages and use the Live DVD/USB recovery

### 7.2. Live DVD/USB recovery

{% capture notice_contents %}
**Under construction - might be incomplete and not fully tested**
{% endcapture %}

{% include notice level="warning" %}

- boot from the Live DVD/USB

- install the necessary packages:

```bash
apt install -y mdadm zfsutils-linux
```

- download the key from the network (or copy from the backup):

```bash
# a) the native ZFS option
curl --insecure -s 'your-root-key-sas' > /etc/crypt/zfs/root.key

# b) the LUKS root option
curl --insecure -s 'your-root-key-sas' > /etc/crypt/root.key

# c) the swap LUKS key (if any)
curl --insecure -s 'your-root-key-sas' > /etc/crypt/swap.key
```

- load the encryption keys:

```bash
# a) the native ZFS option
# - nothing to be done here specifically
#   (key to be auto-loaded on ZFS mount)


# b) the LUKS root option (with mdraid mirror /dev/md2)
cryptsetup luksOpen -d /etc/crypt/root.key /dev/md2 zroot


# c) the LUKS swap (if any)
cryptsetup luksOpen -d /etc/crypt/swap.key /dev/md3 swap
```

- import/mount the ZFS pools  
(if not using ZFS, mount the root drive via the regular mount command):

```bash
mkdir -p /target

# create the target dir
mkdir /target

# import the pools
zpool export -a
zpool import -f -N -R /target rpool
zpool import -f -N -R /target bpool

# import the ZFS keys if any
zfs load-key -a

# mount the ZFS data sets
zfs mount -a

# check if mounted properly
ls -al /target/boot/grub
ls -al /target/home
```

- chroot to the target system to repair:

```bash
# prepare to enter the target system chroot
mount --rbind /dev     /target/dev
mount --rbind /dev/pts /target/dev/pts
mount --rbind /proc    /target/proc
mount --rbind /sys     /target/sys
mount --rbind /run     /target/run

# enter the chroot
chroot /target /usr/bin/env \
    bash --login

# import any other ZFS keys stored on the root partition
# (will load from the target root, e.g. "/etc/crypt/zfs/home.key")
zfs load-key -a
# mount the home
zfs mount -a

# validate the USER datasets mounted
mount | grep USER

# mount all additional disks from /etc/fstab
mount -a
```

- perform any fixes / updates / recovery

- finalize when done (for ZFS):

```bash
# exit the chroot
exit

# unmount and export the zpool
mount | grep -v zfs | tac | awk '/\/target/ {print $3}' | \
    xargs -i{} umount -lf {}

zpool export -a

# reboot and test
reboot
```

## Resources and references

The article is loosely based on the following resources:
- [Auto-mounting encrypted drives with a remote key on Linux](https://withblue.ink/2020/01/19/auto-mounting-encrypted-drives-with-a-remote-key-on-linux.html)
- [Debian Lenny + LUKS encrypted root + hidden USB keyfile](https://www.oxygenimpaired.com/debian-lenny-luks-encrypted-root-hidden-usb-keyfile)
- [Passwordless encryption of the Linux root partition on Debian 8 with an USB key](https://www.howtoforge.com/tutorial/passwordless-encryption-of-linux-root-partition/)
- [Enable Wireless networks in Debian Initramfs](http://www.marcfargas.com/posts/enable-wireless-debian-initramfs/)

{% include abbrev domain="computers" %}
