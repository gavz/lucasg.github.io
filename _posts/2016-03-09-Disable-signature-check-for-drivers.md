---
layout: post
title: "Enable non-signed drivers to be loaded and ran under Windows"
date: 2016-03-09
---

"Recents" Windows (from XP up to 10) force by default drivers to be signed (i.e. by Verisign) in order to be launched. This behavior, which has greatly reduced the attack surface for malware creators, can be a hindrance when you want to develop custom drivers. Here's how to disable it.


<!--more-->

<br>

## Windows 10

open an admin prompt and type :
```shutdown.exe /r /f /o /t 00 ``` <br>
in order to restart under advanced startup. After rebooting, click on "Troubleshooting", then "Advanced options"; "Startup Options" and finally click on "Restart".

After the second reboot, you have access to startup settings where you can disable driver signature enforcement by pressing F7.

That's it.

<br>

## Windows 7

The method under Win7 is a little hairier. Like before, open an admin prompt and type :


```bcdedit.exe -set loadoptions DDISABLE_INTEGRITY_CHECKS```
<br>
```bcdedit.exe -set TESTSIGNING ON```

And reboot.


If it does not work, find the registry key ```[HKEY_CURRENT_USER\Software\Policies\Microsoft\Windows NT\Driver Signing]``` and set ```BehaviorOnFailedVerify``` to `0`.

<br>

## Linux

Easy : type ```make menuconfig``` and look for the ```Enable loadable kernel module``` option to disable it.<br>
Then you just have to recompile your kernel. Yep. <br>

You can also generate a key and sign every kernel modules using your key.