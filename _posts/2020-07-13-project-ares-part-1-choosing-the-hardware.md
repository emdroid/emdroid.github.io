---
title: "Project Ares: Part I – Choosing the hardware"
header:
  teaser: /assets/images/posts/nas-ares/1/cs381.jpg
toc: true
sidebar:
  nav: "fs-ares"
categories:
  - Hardware
  - Storage
tags:
  - hardware
  - nas
  - server
  - vm
---

![NAS Box](/assets/images/posts/nas-ares/1/cs381.jpg){: .align-center .img-large}

Sharing the experience of building a home NAS/VM server.

In the first part we'll have a look on the hardware used to build the machine.

<!-- more -->

## 1. Requirements

The intention is to build a multi-purpose machine to be deployed as a data/backup storage (NAS), virtual machine server, and eventually other server-like functionality.

The decision to build a new NAS/VM machine came after the current solution (the [QNAP TS-453 Pro](https://www.qnap.com/en-us/product/ts-453%20pro)) started being insufficient.

### 1.1. The current status

The current **QNAP TS-453 Pro** NAS has the following parameters:

- Intel Celeron 4-core CPU
- 16 GB RAM (was upgraded from the initial 2 GB)
- 4 x 4 TB HDD (RAID-5)

The QNAP NAS by itself is not bad at all, in fact is a very good data storage solution.
However once I started running VM&zwj;s there (which is supported via Qemu), I quickly realized that the CPU in particular is not quite up for such a task, and also the VM disk performance is sub-optimal.

In addition I started getting close to the limit of the total 12 TB available space.
Therefore begun to look for bigger disks first, but eventually decided to build a whole new machine.

### 1.2. The new machine requirements

Considering the new machine intended usage (NAS + VM server), I came up with the following requirements:

- enough storage for data, externally accessible bays (at least 4, but ideally 6-8)
- ability to use RAID for data redundancy, data safety via encryption
- enough performance for running virtual machines (at least 4 C / 8 HT CPU, but the more the better)
- silent (at least when idle) and as low power consumption as possible, especially when idle  
  (as the NAS machines are generally switched on most of the time)

## 2. Choosing the hardware

This part describes the selection of hardware, reasons for choosing the particular items and the alternatives.

### 2.1. The case: [Silverstone SST-CS381](https://www.silverstonetek.com/product.php?pid=861)

![SST-CS381]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/cs381.jpg "Silverstone SST-CS381"){: .align-left .img-medium}

The main requirement for the case was ideally 6-8 externally accessible HDD bays (so that switching a disk is easy without having to take the whole thing apart).

In addition of course a good enough airflow while having reasonable dimensions (don't have enough room in my apartment to install an air-conditioned 19" rack cabinet, however "cool" it may be).

Looking through several options, I eventually settled with the Silverstone SST-CS381 case:
- 8 external hot-swappable bays for 3.5" or 2.5" HDD&zwj;s  
  (2 separate cages, support of SAS-12G or SATA-6G drives)
- accommodates micro-ATX motherboards
- 2 x 12 cm rear ventilator included + additional 2 x 12 cm side fans possible
- 4 full-profile expansion slots
- eventually fits 12 x 24 cm water cooler radiator

There are some more similar cases, but this one was in particular also the most easily available at the time (they are usually somewhat difficult to get, especially in the Europe).

**Alternatives considered**

*[SUPER MICRO CSE-721TQ-250B](https://www.supermicro.com/en/products/chassis/tower/721/SC721TQ-250B)*:

- has a CD/DVD drive position
- somewhat cheaper
- 4 external bay positions
- ITX-only
- only 1 low-profile expansion slot
- 250 W power supply built in
- would be very appropriate for a smaller 4-bay NAS

*[Eolize SVD-NC11-4](https://www.etonix-media.com/en/eolize-svd-nc11-4-mini-itx-gehause-fur-perfekte-nas-server.html)*:

- also relatively cheap
- 4 external HDD positions
- ITX-only
- 200 W power supply built-in
- reportedly loud and suboptimal airflow

*[INTER-TECH IPC SC-4004](https://www.inter-tech.de/en/products/ipc/storage-cases/sc-4004)*:

- cheapest of the mentioned options
- 4 external positions
- ITX-only
- no power supply included

### 2.2. The power supply: [SilverStone SST-SX500-LG](https://www.silverstonetek.com/product.php?pid=527)

![SST-SX500-LG]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/sx500-lg.jpg "SilverStone SST-SX500-LG"){: .align-left .img-medium}

The SST-CS381 case doesn't have the power supply included, therefore it was required to get one.

I went through various options and selected the SilverStone SST-SX500-LG at the end:

- 500 W max
- high 80+ Gold efficiency
- silent 12 cm fan (the SFX power supplies usually employ a smaller 8-9 cm fan,
  which is then usually louder)
- modular cable management

The 500 W maximal power output may be too much, but it is actually a bit difficult to get a *silent* power supply with less output nowadays, and considering I was looking into a more powerful machine (for VM) that implies a heavier CPU, and the possibility to have maybe up to 12 HDDs, it is still somewhat appropriate.
Of course looking for even more powerful supply would be an overkill and a waste of money (except when we'd like to add a high-end GPU for CUDA calculations or something similar).

The modular cable management is not a must-have (and some money can be saved by not having it).
But it is quite handy at the end, as only the used cables can be connected and the cabling kept clean.

**Alternatives considered**

*[SilverStone SST-ST30SF v. 2.0](https://www.silverstonetek.com/product.php?pid=458)*:

- 300 W max (should still be more than enough)
- 80+ Bronze efficiency
- no modular cable management
- 92 mm fan

*[be quiet! BN238](https://www.bequiet.com/en/powersupply/1555)*:

- 500 W max
- 80+ Gold efficiency
- modular cable management
- 12 cm fan
- was the first choice initially, but wasn't available for delivery at the time

*[Fractal Design Ion SFX-L 500 W](https://www.fractal-design.com/products/power-supplies/ion/ion-sfx-500w-gold/black/)*:

- 500 W max
- 80+ Gold efficiency
- modular cable management
- 12 cm fan
- semi-passive*) - the fan is turned off when the power consumption is low
- slightly more expensive
- wasn't available from Amazon at the time

*) For the expected maximum power consumption around 200 W the Fractal Design would likely operate in the passive mode, i.e. completely silent.
On the other side, the power supplies with the silent 12 cm fan can in fact help with the case airflow (when the power supply is turned with the fan facing inwards).
More about that in the Part II.

### 2.3. The CPU: [Intel Core i7-10700T](https://www.intel.com/content/www/us/en/products/processors/core/i7-processors/i7-10700t.html)

![i7-10700T]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/core-i7.jpg "Intel Core i7-10700T"){: .align-left .img-medium}

As mentioned in the Requirements section, the new system should be capable of running virtual machines, therefore the more powerful CPU the better.

At the end I decided to go with the low-TDP version of the new 10-th Comet Lake generation Intel Core i7-10700T (8 C / 16 HT / TDP 35 W). In particular the 35 W version has lower consumption (this also implies easier cooling) than the "standard" 65 W Intel Core i7-10700, at the expense of somewhat lower performance (core frequency).
However for running multiple VM&zwj;s the core frequency isn't as much as important as the number of cores.

At the time the *T low-TDP versions of the CPU&zwj;s were quite hard to get, but I was eventually able to get my hands on it.

If the NAS would only be used as a storage machine and not running VM&zwj;s, a 6-core i5-10400T/10500T or even a 4-core i3-10300T would be much cheaper and still powerful enough option to handle the SW RAID workload etc.

Performance comparison: [i7-10700T vs i7-10700](https://cpu.userbenchmark.com/Compare/Intel-Core-i7-10700T-vs-Intel-Core-i7-10700/m1218230vs4077)


**Alternatives considered**

*[Intel Core i5-10500T](https://www.intel.com/content/www/us/en/products/processors/core/i5-processors/i5-10500t.html)*:

- 10-th gen Comet Lake 6 C / 12 HT 14 nm CPU
- low-power TDP 35 W
- significantly cheaper than the i7-10700T, would still be powerful enough for a less expensive machine build
- performance comparison: [i5-10500T vs i7-10700T](https://cpu.userbenchmark.com/Compare/Intel-Core-i5-10500T-vs-Intel-Core-i7-10700T/m1204939vsm1218230)


*[Intel Core i3-10300T](https://www.intel.com/content/www/us/e/products/processors/core/i3-processors/i3-10300t.html)*:

- 10-th gen Comet Lake 4 C / 8 HT 14 nm CPU
- low-power TDP 35 W
- cheap option, would still be more than enough for pure-storage NAS builds without much VM usage

*[AMD Ryzen 5 (/PRO) 3400GE](https://www.amd.com/en/products/apu/amd-ryzen-5-pro-3400ge)*:

- AMD 4 C / 8 HT 12 nm CPU
- low-power TDP 35 W
- difficult to get (mostly the regular 65 W TDP versions are offered)
- performance comparison: [Ryzen 5 3400G vs Core i3-10300](https://cpu.userbenchmark.com/Compare/AMD-Ryzen-5-3400G-vs-Intel-Core-i3-10300/m825156vs4074)  
  (the regular versions comparison, comparison of low-TDP models currently not available)  
  => i3 is slightly cheaper and has approx. 19 % better in overall performance

### 2.4. The RAM: [Corsair Vengeance LPX 32GB (2x16GB) DDR-4 3200 MHz](https://www.corsair.com/eu/en/Categories/Products/Memory/VENGEANCE-LPX/p/CMK32GX4M2B3200C16)

![Corsair Vengeance]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/corsair-vengeance.png "Corsair Vengeance LPX DDR-4"){: .align-left .img-medium}

The memory details are generally not that important for the Intel i3/5/7 architecture (the frequency/latency seems to be much more important for the AMD platform), therefore just chose the memory set that was readily available at the time for a reasonable price.
The DDR-4 3200 MHz should be fast enough, the capacity of 32 GB should also be relevant for the intended usage of launching VM&zwj;s (eventually it could be extended by another 2 x 16 GB modules for the total of 64 GB RAM or even more).

For the "regular" (non-VM) NAS usage 16 GB or even 8 GB should be enough.
Although if the ZFS file-system is to be used (currently being promoted a lot for storage systems), it is quite memory heavy as well.

Another alternatives I'd eventually consider are e.g. the Kingston HyperX, Crucial Ballistix or similar memory modules.

### 2.5. The motherboard: [ASUS Prime Z490M-Plus](https://www.asus.com/Motherboards/PRIME-Z490M-PLUS/)

![ASUS Z490M-Plus]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/asus-prime-z490m-plus.jpg "ASUS Prime Z490M-Plus"){: .align-left .img-medium}

For the new Intel Comet Lake processors the new LGA-1200 socket is required.

There are various chipset options:

- H410 (lowest end)
- B460
- H470
- Q470
- Z490 (highest end)
- see e.g. here for comparison:  
  [https://en.wikipedia.org/wiki/LGA_1200](https://en.wikipedia.org/wiki/LGA_1200)


I decided to go with a Z490 motherboard, as it offers the most features.
It is the only chipset that allows the Intel CPU overclocking, but that wasn't the main concern (if anything, I'd be more likely to underclock the CPU to lower the power consumption and the heating).
But the Z490 chipset also has the most available PCI-Express lines, which is particularly important when a lot of drives (and eventually HBA or similar disk controller cards) are to be used.

In addition the motherboard needs to be micro-ATX (i.e. not the full ATX) format.

The selection of the motherboard manufacturer was rather arbitrary, just chose the "cheapest" and most readily available one (the LGA-1200 motherboards were still not that available at the time of building the machine).
I would as well be completely fine with Gigabyte (which I personally have a good experience with) or ASRock motherboards.

The motherboard offers in particular:

- 4 x DIMM DDR-4 slot
- 2 x PCIe x16, 2 x PCIe x1 slots
- 2 x m.2 PCIe NVME slots (one of them can also be used as SATA m.2)
- 5 x SATA-600 connectors
- 1000 Mbps network connector

### 2.6. The system SSD: [Transcend TS512GMTE220S 512 GB PCIe Gen3 x4 NVME](https://www.transcend-info.com/Products/No-991)

![Transcend SSD]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/transcend-nvme.jpg "Transcend 512 GB NVME SSD"){: .align-left .img-medium}

This is not completely essential, as I initially considered using a small 2.5" disk that I already had as a system disk.
But as the motherboard offers the fast PCIe NVME m.2 SSD slots, I eventually decided to get one for the system to improve the boot-up time, and use the other HDD ad a secondary RAID-1 mirror to backup the system.

I eventually chose the Transcend SSD version with the DRAM cache (as the version without the DRAM is indeed cheaper, but reportedly much slower).
There is also the 256 GB version which would suffice, but it is not all that much cheaper (the difference was 77 EUR for the 512 GB version vs. 52 EUR for the 256 GB version), so I bought the bigger one (in addition the secondary drive I have and plan to use as the mirror is also a 512 GB drive).

Note that there are 2 different m.2 standards (actually 3 more precisely, but 2 of them are the most used) - the older SATA-600 NGFF standard (B-key), which is slower (up to 600 MB/s), and the new fast PCIe NVME standard (M-key) that is much faster (up to 6 GB/s, with the Transcend drive the reported speeds are around 3400 MB/s read and 2000 MB/s write).

**Alternatives considered**

*[Corsair MP510 Force 480GB](https://www.corsair.com/eu/en/Categories/Products/Storage/M-2-SSDs/Force-Series-MP510/p/CSSD-F480GBMP510)*:

- similar speeds to the Transcend, more "known" manufacturer
- reported Linux incompatibilities (issues with TRIM), see e.g. [here](https://forum.corsair.com/v2/showthread.php?p=1023476) or [here](https://forum.corsair.com/v3/showthread.php?t=189430) for details, also [TCG Opal issues](https://www.reddit.com/r/linuxhardware/comments/aqvy7i/linux_friendly_m2_ssd/)

*[WD Black SN750 NVMe 500 GB](https://shop.westerndigital.com/en-us/products/internal-drives/wd-black-sn750-nvme-ssd)*:

- the speed is very good, but relatively expensive
- also reported Linux incompatibilities (system hungs in both Fedora 28 / Ubuntu 18.04, disk not shown or disappearing in the Linux installer - see e.g. [here](https://www.amazon.de/Black-High-Performance-interne-Gaming-WDS500G3X0C-00SJG0/product-reviews/B07MH2P5ZD/ref=cm_cr_arp_d_viewpnt_rgt?ie=UTF8&reviewerType=all_reviews&filterByStar=critical&pageNumber=1))

*[Crucial P2 500GB PCIe](https://www.crucial.com/ssd/p2/ct500p2ssd8)*:

- not much experience with the drive at the time
- wasn't available for delivery at the time

Generally there seem to be quite a few Linux issues reported for the "established" disk manufacturers SSD&zwj;s (Corsair, WD, ...) with particular features not working properly (TRIM, Opal encryption, ...) or providing poor performance.
Many of those devices are apparently officially supported for Windows only and may not even be tested under Linux at all - which can eventually lead to issues when trying to use these SSD&zwj;s for Linux machines.

### 2.6. The CPU Cooler: [Noctua NH-L9x65](https://noctua.at/en/nh-l9x65)

![NH-L9x65]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/nh-l9x65.jpg "Noctua NH-L9x65"){: .align-left .img-medium}

The CPU heatsink space of the SST-CB381 case is quite constrained by the middle HDD cage - according to the specs the CPU cooler height limit is 59 mm.
Therefore a low-profile CPU cooler needs to be used.

I initially built in a small Arctic active cooler (see below), but eventually switched to the [Noctua NH-L9x65](https://noctua.at/en/nh-l9x65).
It wouldn't fit normally, but I removed the ventilator (using it as a "passive" cooler) to make the machine more silent.
As the cooler is relatively big for a low-profile cooler (the passive part of the cooler is nearly a perfect fit for the space available), it can handle the 35 W TDP CPU in this "passive mode" (providing there is enough airflow otherwise, but that is also necessary for cooling the hard drives so it didn't require any extra fans).

Check the next part ("Putting it together") for the tests and temperature measurements.

Initially I was also thinking about using a water cooling system, as the SilverStone case can accommodate it eventually, however that also has some issues:

- the radiators are relatively big (so they take a lot of space in the case)
- there is no ventilator on the CPU, but there is still a pump instead, which sometimes also makes a noise (although usually insignificant)
- there is the danger of eventual leakage, should something go wrong with the cooling system (and you generally don't want fluids dropping onto the motherboard)

Generally speaking the water cooling might be beneficial if using a CPU with a (much) higher TDP, like the high-performance 125 W+ models, especially considering the low-profile coolers might not be able to cope with those at all.
But as the 35 W model is used here, the "safer" (and cheaper) air cooling should be sufficient.

**Alternatives considered**

*[Arctic Alpine 12 LP](https://www.arctic.ac/eu_en/alpine-12-lp.html)*:

- cheap low profile cooler
- very good cooling performance up to the 65 W TDP models
- active cooler (with a fan) but relatively quiet
- was the first build option, later replaced by the Noctua that can cool semi-passively
- generally a very good and money-saving option, at the expense of slightly more noise

*[ARCTIC Liquid Freezer II 240](https://www2.arctic.ac/liquidfreezer2/)*:

- water cooling system
- was the initial option I was looking into, unfortunately it didn't fit the case  
  (because of the pipes going upwards - the CPU block itself would fit, but the pipes are blocked by the HDD cage)
- therefore I had to return it and I eventually decided to just stay with the air cooling

*[SilverStone Tundra SST-TD02-Slim-V2](https://www.silverstonetek.com/product.php?pid=597)*:

- another water cooling system
- should reportedly fit the SilverStone case properly

### 2.7. The Disk Controller: [10Gtek LSI-2008-8I HBA](https://www.10gtek.com/products/6Gb-s-Internal-PCI-Express-SAS-SATA-HBA-RAID-319.html)

![10Gtek HBA]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/10gtek-hba.jpg "10Gtek LSI-2008-8I HBA"){: .align-left .img-medium}

As mentioned before, the SST-CS381 case supports up to 12 disks, 8 of them in the 2 HDD cages that are connected via the SFF-8643 mini-SAS connector (each connection supports up to 4 drives).

However the motherboard only has total of 5 SATA connectors, so it would support connecting one of the cages only (via the SATA-to-SFF conversion cable, more about that later).

Therefore additional disk controller card is needed in order to support both cages.
There are 2 basic options:

- **SATA controller**: Provides additional "regular" SATA ports, to be connected by the SAS-SATA reverse cable
- **HBA SAS controller**: Provides SAS ports that can be connected to the HDD cage by a SAS cable

Either of them can come with or without RAID support (the cheaper ones have a SW RAID which offloads the tasks to the CPU) or HW RAID (those are generally very expensive).

I decided to use the SAS HBA card because of its advantages:

- supports both SATA and SAS drives (although I didn't plan to actually use SAS drives)
- supports the SAS side channel (which is generally not supported by the SATA option)
- the cables are easier to manage (just a single cable compared to the SAS-to-SATA adapter that converts the single SAS port to 4 SATA connectors), i.e. the cable management is easier (also, the SATA conversion cable is directional and the reverse version is needed which is quite hard to get)
- on the downside, the HBA controllers are significantly more expensive than the SATA controllers (approx. 2-3x more), the cables are also somewhat more expensive

As for the HW RAID options, I generally prefer to stick with the SW RAID anyway (not even using the HBA capabilities, just using the Linux SW RAID), as the HW RAID (besides the controllers being very expensive) also has some significant disadvantages:

- it may be difficult to recover the data should something go wrong with the adapter (might need the exact same adapter to replace or at least with the same onboard chip) - with the Linux SW RAID you can simply connect the disks to a different machine in most cases
- requires a special driver to be able to install and boot up the OS
- disables/circumvents the file system error detection/correction (in particular when ZFS is used)
- some controllers don't allow to reach the HDD&zwj;s for retrieving the status information like SMART (especially in the RAID configuration), eventually might also block SSD TRIM (leading to greater SSD wear leveling thus decreasing the SSD lifetime)

Therefore the chosen HBA appears to be close to perfect for the use-case.

**Alternatives considered**

*[MZHOU PCIe SATA Card 8 Port](https://www.amazon.com/dp/B082D6XSZN)*:

- provides 8 extra SATA ports through PCI-Express x1 (vs. the PCIe x8 for the 10Gtek HBA)
- for the SilverStone case, a reverse SST-8643 to SATA cable is required  
  (the most cables are SATA-to-SAS cables that will not work; the reverse cables are much harder to get)
- because of the use of 4 SATA ports for a single SFF-8643 conversion, the cable management is suboptimal
- however much cheaper than the HBA cards

#### 2.7.a) The Disk Controller Cables: [10Gtek Mini-SAS SFF-8643 to SFF-8087 Internal Cable](https://www.amazon.com/dp/B01AOS4RFQ)

![10Gtek HBA Cable]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/hba-cable.jpg "10Gtek Mini-SAS SFF-8643 to SFF-8087 Internal Cable"){: .align-left .img-medium}

The 10Gtek HBA has the SFF-8087 connectors, therefore the SFF-8087 to SFF-8643 cables are needed.
They are bidirectional, i.e. work both ways (contrary to the SATA conversion cables).

For the 2 cages 2 cables are needed. I chose the cables from the same manufacturer (10Gtek) that are good quality according to the Amazon reviews (which I myself must configm, worked like a charm).
These are sold in pairs, therefore only one package is needed; ordered the length of 0.8 m which turned to be the exactly right one for the SilverStone case (0.5 m would be a bit too short).

**Alternatives considered**

*[SilverStone CPS05-RE](https://www.silverstonetek.com/product.php?pid=701)*:

- SFF-8643 to 4x SATA (host) reverse conversion cable
- would be needed (2 x) if the SATA PCIe controller card is used
- compared to the HBA cable, 4 SATA connectors are used instead of a single SFF-8087
- note that this special reverse cable would be needed - it is directional and the most of available cables are SFF-8643 (host) to SATA (target) which is the opposite of what would be needed here

### 2.8. The Network Adapter: [10Gtek I350-T4 Quad RJ-45 Gigabit NIC](https://www.amazon.com/dp/B01H6NE4X2)

![I350-T4 NIC]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/10gtek-nic.jpg "10Gtek I350-T4 Quad RJ-45 Gigabit NIC"){: .align-left .img-medium}

Not strictly necessary, as the motherboard already contains one RJ-45 network connector.
However when working with the old QNAP NAS box I found multiple network connections quite handy, as they can be separated for the data connections, virtual machines etc., and eventually using link aggregation over multiple ports for faster data transfer.

Highlights:

- 4 x 1 Gbps RJ-45 ports
- using the Intel I350-T4 server-grade chip
- PCIe x4 link
- both full-height and low-profile bracket provided
- link aggregation support

Generally the plan is to use 2 of the ports in the link aggregation mode for data transfers (as is used at the old QNAP NAS) and the other 2 (+ eventually the motherboard one) for other connection types, dedicated to virtual machines etc.

**Alternatives considered**

*Using just the motherboard LAN connector*:

- 1 Gbps shared connection for all network transfers (data + VM&zwj;s)
- no link aggregation (obviously, as there is no other link to use)
- no extra money spent on the NIC
- there are no plans for heavyweight (multi-user) usage, so although somewhat slower under circumstances, still wouldn't be too bad
- would totally suffice for a cheaper data-only NAS

*[Intel I350-T2 NIC](https://www8.hp.com/us/en/accessories/product-details/10935418)*:

- 2 x 1 Gbps ports
- PCIe x4 link
- link aggregation support
- cold be used as cheaper option to have 2 link-aggregated ports used for data transfers + using the motherboard connector for the rest of the tasks (VMs etc.)

### 2.9. System ventilators: Noctua NF-P12 redux PWM

![Noctua NF-P12]({{ site.url }}{{ site.baseurl }}/assets/images/posts/nas-ares/1/nf-p12-redux.jpg "Noctua NF-P12 redux PWM"){: .align-left .img-medium}

The SilverStone SST-CB381 case includes 2 x 12 cm rear fans already, and 2 additional fan positions on the left side.
The included ventilators are not particularly noisy (are quite good in the fact), but I eventually decided to switch them for the very silent Noctua 12 cm PWM versions, as the motherboard supports up to 1 (CPU) + 3 (system) PWM ventilators.

I ended up using 3 of these:

- 2 x [Noctua NF-P12 redux-1300 PWM](https://noctua.at/en/nf-p12-redux-1300-pwm): 300-1300 RPM
- 1 x [Noctua NF-P12 redux-1700 PWM](https://noctua.at/en/nf-p12-redux-1700-pwm): 450-1700 RPM
- see the next part ("Putting it together") for details

To explain in short, there are 2 basic types of ventilators:

- **DC** ("Direct Current"): 3-pin connectors, controlled by the voltage directly (the third pin is the monitoring one)
- **PWM** ("Pulse Width Modulation"): 4-pin connectors; the 3 of them are shared with the DC version, except the power pins always get the full voltage (the third is still for monitoring) and the 4th pin is used to control switching the fan motor on an off rapidly, thus steering the rotation speed

The PWM ventilators are typically capable of going to much lower minimum speeds compared to DC, typically to 20% or lower, whereas the DC fans are usually unable to go below the 40-60% range.
This is because if the DC voltage is too low, it is unable to turn the ventilator at all - whereas the PWM always gets the full voltage, therefore it is always able to turn (and those full-voltage bursts are just switched on/off depending on the required speed).

The PWM capability to go lower speeds is then a significant advantage for keeping the computer more quiet when idle.
The SilverStone included ventilators are indeed the DC versions, therefore they are a bit louder when the machine is idle (which will still be most of the time), therefore it makes sense to replace them by the PWM versions.

## 3. Price calculation

The following table contains the total costs of hardware at the time (without disks, which are not included in here):

| Item | Model | Price [EUR] |
| :--- | :--- | ---: |
| Case | [SilverStone SST-CS381](https://www.amazon.de/dp/B07TYVXQKY/) | 299.00 EUR |
| Power Supply | [SilverStone SST-SX500-LG v 2.0 - SFX Serie, 500W](https://www.amazon.de/dp/B01GCSCMW2/) | 85.00 EUR |
| CPU | [Intel Core i7-10700T](https://www.computeruniverse.net/de/intel-core-i7-10700t-tray-sockel-1200) | 354.85 EUR |
| RAM | [Corsair Vengeance LPX 32GB (2x16GB) DDR4 3200MHz C16](https://www.amazon.de/dp/B016ORTNI2/) | 134.90 EUR |
| Motherboard | [ASUS Prime Z490M-Plus](https://www.amazon.de/dp/B08827G69N/) | 172.99 EUR |
| CPU Cooler | [Noctua NH-L9x65](https://www.amazon.de/dp/B00VB3Y89E/) | 46.00 EUR |
| System SSD | [Transcend TS512GMTE220S 512GB PCIe Gen3 x4 NVME](https://www.amazon.de/dp/B07MTWMR18/) | 87.90 EUR |
| HBA Card | [10Gtek® Internal SAS/SATA RAID Controller](https://www.amazon.de/dp/B01M2AC40Y/) | 124.99 EUR |
| HBA Cables | [10Gtek Mini-SAS SFF-8643 to SFF-8087 Cable 0.8 m](https://www.amazon.de/dp/B01DCZC0UW/) | 35.99 EUR |
| Network Card | [10Gtek® Gigabit PCIE I350-T4 NIC](https://www.amazon.de/dp/B01H6NE4X2/) | 101.99 EUR |
| Ventilators | 2 x [Noctua NF-P12 redux-1300 PWM](https://www.amazon.de/dp/B07CG2PGVG/) | 24.83 EUR |
|| 1 x [Noctua NF-P12 redux-1700 PWM](https://www.amazon.de/dp/B07CG2PGY6/) | 12.42 EUR |
| **Total** || **1 480.86 EUR** |

## 4. Alternatives

There are off-the-shelf NAS boxes provided by manufacturers like QNAP, Synology, Netgear, Terra Master etc.

However only the QNAP seems to offer similar configurations (i5/i7, 32 GB RAM, 6+ bays), the other manufacturers usually just put a Celeron-level CPU into the machine.
And the QNAP configurations are pretty costly compared to the above build, as you can see in the following table (prices at the time of building the machine, also without disks):

| Item | Price [EUR] |
| :--- | ---: |
| [QNAP TVS-1282-i7-32G](https://www.amazon.de/dp/B01HO8JVPK/) | 2 567.86 EUR |
| [QNAP TVS-1282-i5-16G](https://www.amazon.de/dp/B01HJUSY9M/) | 2 501.23 EUR |
| [QNAP TVS-1282-i7-64G](https://www.amazon.de/dp/B01M0BNABV/) | 2 868.28 EUR |
| [QNAP TVS-872XT-i5-16G](https://www.amazon.de/dp/B07JH48H21/) | 2 296.06 EUR |

The most close is the QNAP TVS-1282-i7-32G, which we can compare to the the custom build:

{% capture compare_cpu_label %}
CPU
{% endcapture %}

{% capture compare_cpu_nas %}
[Intel Core i7-10700T](https://ark.intel.com/content/www/us/en/ark/products/199314/intel-core-i7-10700t-processor-16m-cache-up-to-4-50-ghz.html):
- 10-th gen Comet Lake
- 8 cores / 16 threads
- 2.0 GHz base / 4.5 GHz turbo
- 16 MB cache
- 35 W / 25 W TDP
{% endcapture %}

{% capture compare_cpu_qnap %}
[Intel Core i7-6700](https://ark.intel.com/content/www/us/en/ark/products/88196/intel-core-i7-6700-processor-8m-cache-up-to-4-00-ghz.html):
- 6-th gen Skylake
- 4 cores / 8 threads
- 3.4 GHz base / 4.0 GHz turbo
- 8 MB cache
- 65 W TDP
{% endcapture %}

{% capture compare_cpu_value %}
**[Benchmark](https://cpu.userbenchmark.com/Compare/Intel-Core-i7-6700-vs-Intel-Core-i7-10700T/3515vsm1218230): <span style="color:green;">+ 18 %</span>**

**Consumption: <span style="color:green;">- 46 %</span>**
{% endcapture %}

{% capture compare_ram_label %}
RAM
{% endcapture %}

{% capture compare_ram_nas %}
32 GB DDR-4:
- 2 x 16 GB
- <span style="color:green;">2 free slots</span> (can extend)
{% endcapture %}

{% capture compare_ram_qnap %}
32 GB DDR-4:
- 4 x 8 GB
- <span style="color:red;">no free slots</span> (can't extend)
{% endcapture %}

{% capture compare_disk_label %}
Disk positions
{% endcapture %}

{% capture compare_disk_nas %}
External:
- 8 x 3.5" / 2.5" HDD / SSD
{% endcapture %}

{% capture compare_disk_qnap %}
External:
- 8 x 3.5" / 2.5" HDD / SSD
- <span style="color:green;">4 x 2.5" SSD</span>
{% endcapture %}

{% capture compare_size_label %}
Enclosure size
{% endcapture %}

{% capture compare_size_nas %}
<span style="color:red;">400 (W)</span> x <span style="color:green;">225 (H)</span> x <span style="color:green;">316 (D)</span> mm
{% endcapture %}

{% capture compare_size_qnap %}
<span style="color:green;">370 (W)</span> x <span style="color:red;">235 (H)</span> x <span style="color:red;">320 (D)</span> mm
{% endcapture %}

{% capture compare_os_label %}
OS / Maintenance
{% endcapture %}

{% capture compare_os_nas %}
*OS must be installed and maintained*:
- must be maintained manually
- can still go with e.g. FreeNAS for a nice graphical interface
- advanced configuration possible (encryption, selection of file system etc.)
- more flexibility for advanced users
{% endcapture %}

{% capture compare_os_qnap %}
*OS pre-installed*:
- easier maintenance
- complete graphical interface
- plugin system
- better suited for beginners or people not wanting to spend too much time setting up the system
{% endcapture %}

{% capture compare_price_label %}
Price
{% endcapture %}

{% capture compare_price_nas %}
<span style="color:green;">1 480.86 EUR</span>
{% endcapture %}

{% capture compare_price_qnap %}
<span style="color:red;">2 567.86 EUR</span>
{% endcapture %}

{% capture compare_price_value %}
**Price difference: <span style="color:green;">- 42 %</span>**
{% endcapture %}

<table>
  <thead>
    <tr>
      <th width="20%">Item</th>
      <th width="40%">Home NAS build</th>
      <th width="40%">QNAP TVS-1282-i7-32G</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      <td markdown="1" rowspan="2"    >{{ compare_cpu_label }}</td>
      <td markdown="1" class="box-top">{{ compare_cpu_nas   }}</td>
      <td markdown="1" class="box-top">{{ compare_cpu_qnap  }}</td>
    </tr>
    <tr>
      <td markdown="1" colspan="2" class="box-top">{{ compare_cpu_value }}</td>
    </tr>
    <tr>
      <td markdown="1"                >{{ compare_ram_label }}</td>
      <td markdown="1" class="box-top">{{ compare_ram_nas   }}</td>
      <td markdown="1" class="box-top">{{ compare_ram_qnap  }}</td>
    </tr>
    <tr>
      <td markdown="1"                >{{ compare_disk_label }}</td>
      <td markdown="1" class="box-top">{{ compare_disk_nas   }}</td>
      <td markdown="1" class="box-top">{{ compare_disk_qnap  }}</td>
    </tr>
    <tr>
      <td markdown="1"                >{{ compare_size_label }}</td>
      <td markdown="1" class="box-top">{{ compare_size_nas   }}</td>
      <td markdown="1" class="box-top">{{ compare_size_qnap  }}</td>
    </tr>
    <tr>
      <td markdown="1"                >{{ compare_os_label }}</td>
      <td markdown="1" class="box-top">{{ compare_os_nas   }}</td>
      <td markdown="1" class="box-top">{{ compare_os_qnap  }}</td>
    </tr>
    <tr>
      <td markdown="1" rowspan="2"   >{{ compare_price_label }}</td>
      <td markdown="1" align="center">{{ compare_price_nas   }}</td>
      <td markdown="1" align="center">{{ compare_price_qnap  }}</td>
    </tr>
    <tr>
      <td markdown="1" colspan="2" align="center">{{ compare_price_value }}</td>
    </tr>
  </tbody>
</table>

As you can see, even the QNAP i5 16 GB version is approx. 1 000 EUR more expensive than the custom NAS build (with 32 GB).

## 5. Next steps

In the next part, we'll have a look onto putting the machine together, with pictures and some measurements.

[Next: Putting it together]({% post_url 2020-07-15-project-ares-part-2-putting-it-together %}){: .btn .btn--info}
{: .align-right}

{% include abbrev domain="computers" %}
