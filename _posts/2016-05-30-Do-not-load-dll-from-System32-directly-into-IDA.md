---
layout: post
title: "Do not load dll from System32 directly into IDA"
date: 2016-05-30
---


It's tempting do fire up IDA and load up a system binary (like ```C:/Windows/System32/ntdll.dll```) in order to check something quickly in the machine code. Don't.

<!--more-->

First thing you'll notice, IDA does not want to load it without Adminstrator privilege. That's perfectly normal since IDA usually save the ```.idb``` file in the same folder as the binary, which in this case is the folder for system-wide librairies. There error lies into thinking open IDA in admin mode will solve the issue.

![Stupid user !](/assets/ida_in_admin_mode.png) 


Apart from disabling drag & drop as well as shortcuts, opening IDA in admin mode has another side effects. Here is the initial analysis for ```ntdll!RtlUserThreadStart``` side by side for user mode on the left and admin mode on the right

![IDA you're drunk](/assets/ida_side_by_side_comparison.png)

For an unknown reason, IDA in admin forcefully decompile the x64 ntdll as x86 code (althought the loader correctly recognize the dll as PE64). Moreover, there is no corresponding opcodes between the two versions, indicating some offset computation went wrong.

From the log we can confirm the wrong loader has been used : 

* user mode :

		Autoanalysis subsystem has been initialized.
		Possible file format: MS-DOS executable (EXE) (${ida_dir}\loaders\dos64.l64)
		Possible file format: Portable executable for AMD64 (PE) (${ida_dir}\loaders\pe64.l64)
		Loading file 'D:\ntdll.dll' into database...
		Detected file format: Portable executable for AMD64 (PE)
		  0. Creating a new segment  (0000000180001000-00000001800FC000) ... ... OK
		  1. Creating a new segment  (00000001800FC000-00000001800FD000) ... ... OK
		  2. Creating a new segment  (00000001800FD000-00000001800FE000) ... ... OK
		  3. Creating a new segment  (00000001800FE000-000000018013F000) ... ... OK
		  4. Creating a new segment  (000000018013F000-0000000180148000) ... ... OK
		  5. Creating a new segment  (0000000180148000-0000000180155000) ... ... OK
		  6. Creating a new segment  (0000000180155000-0000000180159000) ... ... OK

* admin mode : 
	
		Autoanalysis subsystem has been initialized.
		Possible file format: MS-DOS executable (EXE) (${ida_dir}\loaders\dos64.l64)
		Possible file format: Portable executable for 80386 (PE) (${ida_dir}\loaders\pe64.l64)
		Loading file 'C:\Windows\System32\ntdll.dll' into database...
		Detected file format: Portable executable for 80386 (PE)
		  0. Creating a new segment  (000000004B281000-000000004B386000) ... ... OK
		  1. Creating a new segment  (000000004B386000-000000004B387000) ... ... OK
		  2. Creating a new segment  (000000004B387000-000000004B388000) ... ... OK
		  3. Creating a new segment  (000000004B388000-000000004B38C000) ... ... OK
		  4. Creating a new segment  (000000004B38C000-000000004B38F000) ... ... OK


Weirdly enough, this only happen when opening a 64-bit dll in a privileged folder (executables are OK, and any 64-bit binary in a user folder is OK too).


Smoking gun 
-----------

When opening the SysWow64 version of a dll (and creating ```id*``` files in ```C:\Windows\SysWow64```) you can't open the 64-bit version of the same dll.
There is probably some name-conflicts in the available paths and when opening ```C:\Windows\System32\ntdll.dll```, the loader (```pe64.l64``` is a 32-bit module) probably incorrectly load the SysWow version.
