---
title: "Switching off Windows Virtual Store"
header:
  teaser: /assets/images/thumb/registry.png
toc: true
categories:
  - Operating Systems
  - Windows
tags:
  - windows
  - uac
  - virtual-store
  - virtualization
  - git
---

![Virtual Store](/assets/images/thumb/registry.png){: .align-left .img-thumbnail}

**Windows File System and Registry Virtualization** (Virtual Store) is a technology to provide backward compatibility for legacy applications, which require unlimited access to system locations like "_Program Files_" or "_Windows_" folders, "_Local Machine_" registry, etc.

Unfortunately, this technology can significantly affect the performance of applications, therefore it can help to disable it in some cases, if we do not anticipate any legacy application requiring it.

<!-- more -->

## The costs of Windows Virtual Store

The write access to important system locations it is no longer allowed under Windows 7 and newer even for Administrator accounts, as such attempts are blocked by the UAC. The Windows Filesystem / Registry Virtualization helps to mitigate possible issues of legacy applications by redirecting those writes into the current user profile directory and/or user registry.

The redirection is however not for free and it can have significant performance consequences also for applications which do not actually need it. One such example is Git and GitBash shell emulation, which perform pretty poor with the Windows File / Registry Virtualization enabled - see e.g. here:

- [Git/Bash is extremely slow in Windows 7 x64](http://stackoverflow.com/a/22208863/1274747) - the post actually states that it should no longer be necessary with the newer Git versions, however according to my experience, disabling the Virtual Store still significantly improves the speed of Git and GitBash commands, especially if executed as part of Git commit hooks
- [Msysgit bash is horrendously slow in Windows 7](http://stackoverflow.com/q/2835775/1274747)
- experienced the same issue with the Windows GCC port (MinGW) running sub-optimal (not running full speed even with parallel make)

This is especially a bummer if you do not use any application which needs the Virtual Store.
Moreover, applications which do not _actually_ need the Virtual Store but do not have the security manifest embedded [^1] [^2], will still go through the compatibility check calls, rendering the application to run horrendously slow under circumstances (e.g. when called repeatedly in a loop).

Note that even for _reads,_ if the application does not have the manifest, even for any reads from system locations/registry, the Windows first needs to check whether the file/registry key exists in the Virtual Store location.

## Disabling the virtualization

There are two registry keys to disable the Virtual Store. Each of them should work on its own, but I generally use both to make sure everything is disabled:

1. **Disabling the Virtualization in the Policies** [^3]:
  - registry key:  
  _HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System_
  - create/update the DWORD value "_EnableVirtualization_", set to "_0_"
(disables the Virtual Store functionality)
2. **Disabling the Virtual Store Windows Driver** (LUAFV: LUA = UAC, FV - file virtualization):
  - registry key:  
  _HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\luafv_
  - set the value "_Start_" to "_4_" [^4]  
(sets the service to "_Disabled_", i.e. to not start automatically)

For convenience the contents of REG file for importing to the registry directly:  
(create a new file with the ".reg" extension, insert the contents and double-click the file to insert into the registry)

```
REGEDIT4

[HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows\CurrentVersion\Policies\System]
"EnableVirtualization"=dword:00000000

[HKEY_LOCAL_MACHINE\SYSTEM\CurrentControlSet\Services\luafv]
"Start"=dword:00000004
```

After finishing the setup the **restart of Windows** is necessary.

## Resources and references

[^1]: [&lt;trustInfo&gt; Element](https://msdn.microsoft.com/en-us/library/6ad1fshk.aspx)
[^2]: [Making Your Application UAC Aware](https://www.codeproject.com/Articles/17968/Making-Your-Application-UAC-Aware)
[^3]: [UAC Group Policy Settings and Registry Key Settings](https://technet.microsoft.com/en-us/library/dd835564(WS.10).aspx#BKMK_Virtualize)
[^4]: [Windows Service Start Key](https://technet.microsoft.com/en-us/library/cc959920.aspx)

{% include abbrev domain="computers" %}
