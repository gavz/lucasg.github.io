---
layout: post
title: "AppV ISV virtual filesystem"
date: 2018-08-22
---

Office is a behemoth of a software suite : Excel itself load 150+ dll at startup, and dozens other dynamically via OLE calls. Among those dll, some are located in a special folder :

{% include image.html url='/assets/AppvIsvExcelMissingDll.PNG' description='Mso20Win32Client.dll is not found by Dependencies but correctly loaded by Excel.exe' %}


`Mso20Win32Client.dll` is located in a odd looking folder path : `C:\Program Files (x86)\Microsoft Office\root\VFS\ProgramFilesCommonX86\Microsoft Shared\OFFICE16\Mso20win32client.dll`; `VFS` probably meaning `Virtual FileSystem`. I've heard previously that certain MS applications were using some kind of a virtual filesystem. I think that's partly what's behind Project Centennial, the technology that convert Win32 applications into UWP applications for the Windows Store. I've looked deeper into in order to learn more about it.

<!--more-->

First thing first, I tried to look with ProcessHacker (and its handle search window) and Procmon which resources are used to store those VFS configuration, but without success. There are hardly any files or "interesting" registry keys touched during Excel startup :

![AppvIsvExcelRegistryRead](/assets/AppvIsvExcelRegistryRead.PNG)

`HKLM\SOFTWARE\Microsoft\AppVISV\C:\Program Files (x86)\Microsoft Office` registry key returns `8a115c93-df1b-47c9-9a9c-a185cf477896`, which looks like a GUID, but nothing else in the registry hive reference this GUID.

I decided to trace the import of `Mso20Win32Client.dll` via Windbg in order to see how the loading is actually done. `ntdll!ShowSnaps` is a static variable - not exported but available via pdb debug symbols - which activate NT loader debug outputs. It is also accessible via `gflags` but I always forget the correct command. Here are the results :

{% highlight text %}
Microsoft (R) Windows Debugger Version 10.0.16299.15 X86
Copyright (c) Microsoft Corporation. All rights reserved.

CommandLine: "C:\Program Files (x86)\Microsoft Office\root\Office16\EXCEL.EXE"

************* Path validation summary **************
Response                         Time (ms)     Location
Deferred                                       SRV*F:\Symbols*http://msdl.microsoft.com/download/symbols
Symbol search path is: SRV*F:\Symbols*http://msdl.microsoft.com/download/symbols
Executable search path is: 
ModLoad: 009a0000 02fb1000   Excel.exe
ModLoad: 77640000 777ce000   ntdll.dll
ModLoad: 756b0000 75780000   C:\WINDOWS\SysWOW64\KERNEL32.DLL
ModLoad: 771e0000 773bd000   C:\WINDOWS\SysWOW64\KERNELBASE.dll
ModLoad: 73f20000 73fbc000   C:\WINDOWS\SysWOW64\apphelp.dll
ModLoad: 770e0000 771da000   C:\WINDOWS\SysWOW64\ole32.dll
ModLoad: 76ba0000 76e03000   C:\WINDOWS\SysWOW64\combase.dll
ModLoad: 507b0000 50999000   C:\Program Files (x86)\Microsoft Office\root\Office16\AppVIsvSubsystems32.dll
ModLoad: 77520000 7763e000   C:\WINDOWS\SysWOW64\ucrtbase.dll
ModLoad: 765b0000 76670000   C:\WINDOWS\SysWOW64\RPCRT4.dll
...
ModLoad: 50600000 507aa000   C:\Program Files (x86)\Microsoft Office\root\Office16\c2r32.dll
ModLoad: 76720000 7679d000   C:\WINDOWS\SysWOW64\msvcp_win.dll
ModLoad: 773c0000 77438000   C:\WINDOWS\SysWOW64\ADVAPI32.dll
...
ModLoad: 706d0000 706f2000   C:\WINDOWS\SysWOW64\USERENV.dll
(33c0.4abc): Break instruction exception - code 80000003 (first chance)
eax=00000000 ebx=00000010 ecx=c81e0000 edx=00000000 esi=031eb000 edi=776465bc
eip=776edd79 esp=0335f8b8 ebp=0335f8e4 iopl=0         nv up ei pl zr na pe nc
cs=0023  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000246
ntdll!LdrpDoDebuggerBreak+0x2b:
776edd79 cc              int     3
0:000> x ntdll!*ShowSnaps*
77754790          ntdll!ShowSnaps = <no type information>
0:000> ed ntdll!ShowSnaps 9
0:000> g
...
33c0:4abc @ 326919546 - LdrGetDllHandleEx - ENTER: DLL name: mso20win32client.dll
33c0:4abc @ 326919546 - LdrpFindLoadedDllInternal - RETURN: Status: 0xc0000135
33c0:4abc @ 326919546 - LdrGetDllHandleEx - RETURN: Status: 0xc0000135
33c0:4abc @ 326919546 - LdrLoadDll - ENTER: DLL name: C:\Program Files (x86)\Common Files\Microsoft Shared\Office16\mso20win32client.dll
33c0:4abc @ 326919546 - LdrpLoadDllInternal - ENTER: DLL name: C:\Program Files (x86)\Common Files\Microsoft Shared\Office16\mso20win32client.dll
33c0:4abc @ 326919546 - LdrpDetectDetour - INFO: !!! Detour detected, disable parallel loading
33c0:4abc @ 326919546 - LdrpResolveDllName - ENTER: DLL name: C:\Program Files (x86)\Common Files\Microsoft Shared\Office16\mso20win32client.dll
33c0:4abc @ 326919546 - LdrpResolveDllName - RETURN: Status: 0x00000000
33c0:4abc @ 326919546 - LdrpMinimalMapModule - ENTER: DLL name: C:\Program Files (x86)\Common Files\Microsoft Shared\Office16\mso20win32client.dll
ModLoad: 501f0000 505f1000   C:\Program Files (x86)\Common Files\Microsoft Shared\Office16\mso20win32client.dll
33c0:4abc @ 326919546 - LdrpMinimalMapModule - RETURN: Status: 0x00000000
...
{% endhighlight text %}

Several interesting infos here : first, the loader has detected a "detour", which is the name of [a library Microsoft recently open-sourced](https://github.com/Microsoft/Detours) to hook natives API. The other is that the loader thinks it loads `C:\Program Files (x86)\Common Files\Microsoft Shared\Office16\mso20win32client.dll` whereas it actually loads `C:\Program Files (x86)\Microsoft Office\root\VFS\ProgramFilesCommonX86\Microsoft Shared\OFFICE16\Mso20win32client.dll` ! 

The solution to this phenomenon becomes pretty clear by looking at `NtOpenFile` (called during a dll load before memory mapping the shared library) :

{% highlight text %}
0:030> u NtOpenFile
ntdll!NtOpenFile:
776ad930 e9bb5b18d9      jmp     AppVIsvSubsystems32!RequestUnhookedFunctionList+0x70660 (508334f0)
776ad935 bae0686c77      mov     edx,offset ntdll!Wow64SystemServiceCall (776c68e0)
776ad93a ffd2            call    edx
776ad93c c21800          ret     18h
776ad93f 90              nop
ntdll!NtDelayExecution:
776ad940 b834000600      mov     eax,60034h
776ad945 bae0686c77      mov     edx,offset ntdll!Wow64SystemServiceCall (776c68e0)
776ad94a ffd2            call    edx

{% endhighlight text %}

`AppVIsvSubsystems32.dll` seems to hook quite a number of `ntdll` APIs. `AppVIsvSubsystems32.dll` is a dll located in `C:\Program Files\Common Files\microsoft shared\ClickToRun`, but described as being part of a tech stack called `Microsoft Application Virtualization (App-V)`. Below is the hook placed for `ntdll!NtOpenFile` :

![AppvIsvNtOpenFileHook](/assets/AppvIsvNtOpenFileHook.PNG)

Since I don't have any more informations, IDA was my only "friend" for now on. Unlike many Windows binaries developed by Microsoft, `AppVIsvSubsystems32.dll` symbols are not publicly available. Fortunately, there are debug strings and RAII peppered in the code base. Unfortunately, it also means it a C++ code base to reverse, with std calls; try/catch statements and lots of function pointer calls.

Among all the debug strings present, some stood up :

![AppvIsvSftVenvDebugString](/assets/AppvIsvSftVenvDebugString.PNG)

`AppVIsvSubsystems32.dll` seem to make RPC calls to an endpoint called `SFT-venv-server` in order to retrieve new environment variables. In order to identify which process holds the `SFT-venv-server` RPC endpoint, I used `RpcView` :


![AppvIsvSftVenvServer](/assets/AppvIsvSftVenvServer.PNG)

`SFT-venv-server` RPC endpoint is located within the `OfficeClickToRun.exe` process, and actually prefixed by 
`AppV-ISV-8a115c93-df1b-47c9-9a9c-a185cf477896` which is the GUID read from the registry at startup. `RpcView` also list all the RPC interfaces bound to the various endpoints within `OfficeClickToRun.exe`, but none of them are documented. I've identified interface `edce686d-acae-4a2a-8945-24489443c35e` as the one used to update Excel's process ENV variables. Here the decompilation returned by `RpcView` :

#### edce686d-acae-4a2a-8945-24489443c35e 

<table>
  <thead>
    <tr>
      <th style="text-align: left">Endpoint</th>
      <th style="text-align: left">Interface</th>
    </tr>
  </thead>
  <tbody>
    <tr>
      	<td style="text-align: left; font-weight: bold">Client</td>
      	<td style="text-align: left">
      		<figure class="highlight">
      		<pre><code class="highlighter-rouge">[                                                   
	uuid(edce686d-acae-4a2a-8945-24489443c35e),         
	version(1.0),                                       
]                                                   
interface DecompileItInterface                      
{                                                   
                                                
	typedef struct Struct_8_t                       
	{                                               
	        long    StructMember0;                  
	        short   StructMember1;                  
	        short   StructMember2;                  
	        byte    StructMember3[8];               
	}Struct_8_t;                                    
	                                                
	long Proc0(                                         
	    [in]long arg_1,                             
	    [in]struct Struct_8_t arg_2,                
	    [in]struct Struct_8_t arg_6,                
	    [out]/* simple_ref */small *arg_10,         
	    [out]/* simple_ref */long *arg_11,          
	    [out]/* simple_ref */long *arg_12,          
	    [out][ref][size_is(*arg_11)]byte **arg_13,  
	    [out][ref][size_is(*arg_12)]byte **arg_14); 
}   
      		</code></pre>
      		</figure>
      	</td>
 	</tr>
    <tr>
      	<td style="text-align: left; font-weight: bold">Server</td>
      	<td style="text-align: left">
      		<figure class="highlight">
      		<pre><code class="highlighter-rouge">[
	uuid(edce686d-acae-4a2a-8945-24489443c35e),
	version(1.0),
]
interface DefaultIfName
{

    typedef struct Struct_12_t
    {
        long    StructMember0;
        short   StructMember1;
        short   StructMember2;
        byte    StructMember3[8];
    }Struct_12_t;

long Proc0(
     [in]long arg_1,
     [in]struct Struct_12_t* arg_2,
     [in]struct Struct_12_t* arg_3,
     [out]small *arg_4,
     [out]long *arg_5,
     [out]long *arg_6,
     [out][ref] /* [DBG] FC_BOGUS_ARRAY */ [size_is(*arg_5)] /*  */ [string] wchar_t*** arg_7,
     [out][ref] /* [DBG] FC_BOGUS_ARRAY */ [size_is(*arg_6)] /*  */ [string] wchar_t*** arg_8); 
}    																								  
      		</code></pre>
      	</figure>
      	</td>
 	</tr>
  </tbody>
</table>            

`Struct_8_t` / `Struct_12_t` is probably an `uuid` / `GUID`. It fits exactly with IDA's description of a GUID structure : 

![IdaGUID](/assets/IdaGUID.PNG)

Now that I know the RPC procedure prototype, I know how to read client-side rpc calls :

{% highlight C %}

struct edce686d_acae_4a2a_8945_24489443c35e_Proc0_STACK 
{
  int (__cdecl *rpc_client_call_fn)(int, int);
  int rpc_stack;
  int ___pad;
  int arg_1;
  GUID arg_2;
  GUID arg_6;
  int arg_10;
  int arg_11;
  int arg_12;
  int arg_13;
  int arg_14;
};

// AppVIsvSubsystems32.dll + 0x44F90
int __thiscall __do_rpc_call_to_venv_sft_server(edce686d_acae_4a2a_8945_24489443c35e_Proc0_STACK *Context)
{
  __m128i v1; // xmm0
  int v2; // ST04_4
  int v4; // [esp-34h] [ebp-34h]
  int v5; // [esp-24h] [ebp-24h]
  int v6; // [esp-14h] [ebp-14h]
  int v7; // [esp-10h] [ebp-10h]
  int v8; // [esp-Ch] [ebp-Ch]
  int v9; // [esp-8h] [ebp-8h]
  int v10; // [esp-4h] [ebp-4h]

  v10 = Context->arg_14;
  v1 = _mm_loadu_si128((const __m128i *)&Context->arg_6);
  v9 = Context->arg_13;
  v8 = Context->arg_12;
  v7 = Context->arg_11;
  v6 = Context->arg_10;
  _mm_storeu_si128((__m128i *)&v5, v1);
  v2 = Context->arg_1;
  _mm_storeu_si128((__m128i *)&v4, _mm_loadu_si128((const __m128i *)&Context->arg_2));
  return Context->rpc_client_call_fn(Context->rpc_stack, v2);
}

{% endhighlight C %}


Using the debugger, I've managed to extract the correct input parameters and replicate the RPC call in Python using the really [useful Python library `PythonForWindows`](https://github.com/hakril/PythonForWindows) :


{% highlight python %}

import struct
import time
import sys

import windows.rpc
import windows.generated_def.windef as windef
from windows.rpc import ndr

class NdrGUID(ndr.NdrStructure):
	MEMBERS = [
		ndr.NdrLong,
		ndr.NdrShort,
		ndr.NdrShort,
		ndr.NdrFixedArray(ndr.NdrByte, 8),
    ]

class NdrSftVenvServerParameters(ndr.NdrParameters):
    MEMBERS = [
		ndr.NdrLong,
		NdrGUID,
		NdrGUID,
    ]


class NdrSftVenvServerResponse(ndr.NdrParameters):
    MEMBERS = [
		ndr.NdrShort,
		ndr.NdrLong,
		ndr.NdrLong,
		ndr.NdrUniquePTR(ndr.NdrUniquePTR(ndr.NdrUniqueWString)),
		ndr.NdrUniquePTR(ndr.NdrUniquePTR(ndr.NdrUniqueWString)),
	]

client = windows.rpc.RPCClient(r"\\RPC Control\\AppV-ISV-8a115c93-df1b-47c9-9a9c-a185cf477896vfs_subsystem")
iid = client.bind("edce686d-acae-4a2a-8945-24489443c35e")

input_args = [
	12528, # EXCEL.EXE process PID
	[
		0x9ac08e99,
		0x230b,
		0x47e8,
		(0x97, 0x21, 0x45, 0x77, 0xb7, 0xf1, 0x24, 0xea,)
	], #  9ac08e99-230b-47e8-9721-4577b7f124ea
	[
		0x1a8308c7,
		0x90d1,
		0x4200,
		(0xb1, 0x6e, 0x64, 0x6f, 0x16, 0x3a, 0x08, 0xe8,)
	], #  1a8308c7-90d1-4200-b16e-646f163a08e8
]

resp = client.call(iid, 0, NdrSftVenvServerParameters.pack(input_args))
stream = ndr.NdrStream(resp)
args = NdrSftVenvServerResponse.unpack(stream)
return_value = ndr.NdrLong.unpack(stream)

print("input_args: %r" % input_args)
print("response: %r" % args)
print("Return value = {0:#x}".format(return_value))

>>> input_args: [12528, [2596310681L, 8971, 18408, (151, 33, 69, 119, 183, 241, 36, 234)], [444795079, 37073, 16896, (177, 110, 100, 111, 22, 58, 8, 232)]]
>>> response: [0, 1, 0, u'PATH=%PATH%;C:\\Program Files (x86)\\Microsoft Office\\root\\Client\x00', None]
>>> Return value = 0x0
{% endhighlight python %}

#### edce686d-acae-4a2a-8945-24489443c35e 


Following the RPC call `edce686d-acae-4a2a-8945-24489443c35e:Proc0`  which updates the `PATH` environment variable, `AppVIsvSubsystems32.dll` calls `8c7fbdb0-8513-44f9-a8b1-1a3b49322bf4:Proc0` which will return all systems folders redirections. Here is the interface's decompilation result (and annotated) :

{% highlight C %}
[
  uuid(8c7fbdb0-8513-44f9-a8b1-1a3b49322bf4),
  version(1.0),
]
interface DecompileItInterface
{
  
  // GUID isn't actually a well known Ndr type
  typedef struct _Guid
  {
    long    Data1;
    short   Data2;
    short   Data3;
    byte    Data4[8];

  } Guid;

  // kinda like a UNICODE_STRING struct
  typedef struct _WideString
  {
    [range(0,32767)] long   Length;
    [ptr][size_is(Length)]wchar_t *  Buffer;

  }   WideString;


  // Redirection description entry
  typedef struct _Redirection
  {
    long        StructMember0;
    long        StructMember1;
    
    WideString      OrigDrive;
    WideString      OrigWin32Path;
    WideString      OrigDOSPath;
    
    WideString      RedirDrive;
    WideString      RedirWin32Path;
    WideString      RedirDOSPath;

  } Redirection;

  // ??
  typedef struct Struct_130_t
  {
    long      StructMember0;
    WideString  StructMember1;
    WideString  StructMember2;

  }Struct_130_t;

  // 
  typedef struct _RSODiffContext
  {
    Guid        PackageID;
    Guid        VersionID;
    long      __Flags;
    long      __Policy;
    WideString  RootFolderPath;
    long      RedirectionsCount;
    [ptr] [size_is(RedirectionsCount)] Redirection*    RedirectionsArray;

    // Not used
    long      StructMember7; // = 0 
    [ptr] [size_is(StructMember7)] struct Struct_130_t*   StructMember8; // = NULL 

  } RSODiffContext;

long Proc0(
    [in] long ProcessID,
    [in] Guid PackageID,
    [in] Guid VersionID,
    [out] long *ResponseCount,
    [out] [ref] [size_is( , *ResponseCount)] RSODiffContext** ResponseArray
  );
}
{% endhighlight C %}


We get the following redirections : 

| Original Path     |   Redirection    |
| :------------- | :-------------   |
| C:\ProgramData | C:\Program Files (x86)\Microsoft Office\root\VFS\Common AppData |
| C:\ProgramData\Microsoft\Windows\Start Menu\Programs | C:\Program Files (x86)\Microsoft Office\root\VFS\Common Programs |
| C:\Windows\Fonts | C:\Program Files (x86)\Microsoft Office\root\VFS\Fonts |
| C:\Program Files\Common Files | C:\Program Files (x86)\Microsoft Office\root\VFS\ProgramFilesCommonX64 |
| C:\Program Files (x86)\Common Files | C:\Program Files (x86)\Microsoft Office\root\VFS\ProgramFilesCommonX86 |
| C:\Program Files | C:\Program Files (x86)\Microsoft Office\root\VFS\ProgramFilesX64 |
| C:\Program Files (x86) | C:\Program Files (x86)\Microsoft Office\root\VFS\ProgramFilesX86 |
| C:\Windows\System32 | C:\Program Files (x86)\Microsoft Office\root\VFS\System |
| C:\Windows\SysWOW64 | C:\Program Files (x86)\Microsoft Office\root\VFS\SystemX86 |
| C:\Windows | C:\Program Files (x86)\Microsoft Office\root\VFS\Windows |

When loading a shared library, if the resolved filepath is located in these particular folder, `AppVIsvSubsystems32`'s hook will try to swap the system dll by the VFS one. But where this VFS configuration is located ? Once again, Procmon is a big help on this :

![AppvIsvPackageFiles](/assets/AppvIsvPackageFiles.PNG)


At startup time, the `ClickToRun.exe` process read files from ```C:\ProgramData\Microsoft\ClickToRun\MachineData\Catalog\Packages\{9AC08E99-230B-47E8-9721-4577B7F124EA}\{1A8308C7-90D1-4200-B16E-646F163A08E8}\```, in which there are 2 XML manifest files describing the entirety of an `AppV` configuration for Office (shortcut, file extensions handlers, COM interfaces, ENV variables, etc.). For example, one entry updating the `PATH` ENV variable :

```
  <appv:Extension Category="AppV.EnvironmentVariables">
    <appv:EnvironmentVariables>
      <appv:Include>
        <appv:Variable Name="PATH" Value="%PATH%;[{AppVPackageRoot}]\Client"/>
      </appv:Include>
    </appv:EnvironmentVariables>
  </appv:Extension>
```

Now that I know exactly how this virtual filesystem dll redirection works, here I've emulated in Dependencies :

{% highlight C# %}
// add warning for appv isv applications 
if (String.Compare(DllImport.Name, "AppvIsvSubsystems32.dll", StringComparison.OrdinalIgnoreCase) == 0 ||
    String.Compare(DllImport.Name, "AppvIsvSubsystems64.dll", StringComparison.OrdinalIgnoreCase) == 0)
{
    if (!this._DisplayWarning)
    {
        MessageBoxResult result = MessageBox.Show(
        "This binary use the App-V containerization technology which fiddle with search directories and PATH env in ways Dependencies can't handle.\n\nFollowing results are probably not quite exact.",
        "App-V ISV disclaimer"
        );

        this._DisplayWarning = true; // prevent the same warning window to popup several times
    }

}
{% endhighlight C# %}

This dll redirection is way too complicated for me to attempt to emulate it : to do it properly I would have to read a registry key, locate the RPC endpoint associated, spawn the executable to analyse (most RPC procedures here works using PIDs, not executable paths) and make two or three RPC calls in order to retrieve the folder redirections. Instead, I choose to display a warning popup.

#### RPC Interfaces 

Now we've seen how folder visualization work, we can take a look at all the RPC interfaces `ClickToRun.exe` expose to the user.

`Proc1` of `d99d2add-4f00-41b8-bf99-bb03127f015f` is actually an interesting one since it takes a process `PID` and a binary name, and at first glance seem to load the binary into the remote process. However there are no injection done here (at least none visible via reverse engineering). I think it's more an orchestration thing where the rpc server signal an "enlightened" remote process to load the binary, but it's up to the target process to do the actual loading.

Here below are all the interfaces and procedures available via `ClickToRun.exe`'s process:

| Interface     |   Explanation    |
| :------------- | :-------------   |
| **44e10347-37a0-494c-871c-fb90f7145742** | **CVirtualCOM** |
|   ├ Proc0 | send pid, return GUID (packageId, VersionId) associated |
|   ├ Proc1 | CVirtualCOM::RegisterServer |
|   ├ Proc2 | CVirtualCOM::ReleaseServer |
|   ├ Proc3 | CVirtualCOM SID stuff |
|   ├ Proc4 |  |
|   ├ Proc5 | CVirtualCOM::SgServerMap::ReleaseInstance |
|   ├ Proc6 | CVirtualCOM::WaitForClsidRegistration |
|   ├ Proc7 | high level API ? |
|   ├ Proc8 |  |
|   └ Proc9 | Create registry COM registration |
| **469d3a0e-e164-422e-a662-9cbe0621407e** | **OfficeClickToRun.exe miscellaneous** |
|   ├ Proc0 | QueueTaskItem (ApiServer) |
|   ├ Proc1 | |
|   ├ Proc2 | Triggering an update: {'Parameters': |
|   ├ Proc3 | TaskType something |
|   ├ Proc4 | PromptUser ? |
|   ├ Proc5 | InvokeProcessKiller ? |
|   ├ Proc6 | EnsureOrchestrationForAppLaunch |
|   ├ Proc7 | InstallProofOfPurchase |
|   ├ Proc8 | UninstallProofOfPurchase |
|   ├ Proc9 | HandleScheduledHeartbeat |
|   ├ Proc10 | Querying OSPP for current licenses |
|   ├ Proc11 | RaiseTaskErrorEvent |
|   ├ Proc12 | RaiseTaskDialogEvent |
|   ├ Proc13 | RaiseTaskToastEvent |
|   ├ Proc14 |  TaskIntegrateRepair::IsRepairForAppValidNow |
|   ├ Proc15 | TaskUpdateDetection::BeginUpdatesDiscoveryPeriod |
|   ├ Proc16 |  |
|   ├ Proc17 | TaskIntegrateRepair_DoRepairForApp |
|   ├ Proc18 | retourne le PID du process ClickToRun.exe |
|   ├ Proc19 | RaiseTaskErrorEvent |
|   ├ Proc20 | DiffRSOD |
|   ├ Proc21 | PublishRSOD |
|   ├ Proc22 | ScenarioController::ShowErrorUI |
|   ├ Proc23 | ModifyOfficeProducts" |
|   ├ Proc24 | CancelUpdate |
|   ├ Proc25 | Activate |
|   ├ Proc26 | SetC2RProperty |
|   ├ Proc27 | GetServiceVersion |
|   ├ Proc28 | Deleting AFO Task |
|   ├ Proc29 |  |
|   ├ Proc30 | Triggering an update |
|   ├ Proc31 | InstallProofOfPurchase |
|   ├ Proc32 | UninstallProofOfPurchas |
|   ├ Proc33 | ::HandleScheduledHeartbeat (ApiServer) |
|   ├ Proc34 | |
|   ├ Proc35 | |
|   ├ Proc36 | TaskIntegrateRepair::DoRepairForApp |
|   ├ Proc37 | Activate |
|   ├ Proc38 | Checking virtualized file location for calling module |
|   ├ Proc39 | set some settings |
|   ├ Proc40 | |
|   ├ Proc41 | InvokeProcessKillerEx |
|   ├ Proc42 | ApplyPolicy: Fetch async started |
|   ├ Proc43 | SetPolicyOverride |
|   └ Proc44 | CollectFileDiagnostics |
| **4b183cf6-affd-4872-9da2-7564b683d027** | **AppVIsvSubsystemController.dll** |
|   ├ Proc0       |    ??    |
|   ├ Proc1       |    ??    |
|   ├ Proc2       |    ??    |
|   ├ Proc3       |    ??    |
|   ├ Proc4       |    ??    |
|   └ Proc5       |    ??    |
| **6d809348-7e6c-41b9-91bc-630fe5503d66** | **VObjects ??** |
|   └ Proc0       |   return "exclusions" for virtualizing "objects" (events, etc.)  ? |
| **772431d9-ef99-4f35-99f6-12c8f5d55232** | **Streaming server ?** |
|   ├ Proc0 | convert path into 'C:\\Program Files (x86)\\Microsoft Office' + relative folder |
|   ├ Proc1 | StreamFault ? |
|   ├ Proc2 | LoadRange ? |
|   ├ Proc3 | LoadMemory ? |
|   ├ Proc4 | GetFileDiskRanges |
|   ├ Proc5 | GetFileMemRanges |
|   ├ Proc6 | EnsureDir |
|   ├ Proc7 | EnsureFile |
|   ├ Proc8 | LoadFile |
|   ├ Proc9 | GetPipelineStats |
|   ├ Proc10|  SuperPipeline::GetTotalProgress |
|   ├ Proc11|  StartFeatureBlock |
|   ├ Proc12|  IsFBComplete |
|   ├ Proc13|  GetFBFiles |
|   ├ Proc14|  GetFBError |
|   ├ Proc15|  LoadAll |
|   ├ Proc16|  CancelLoadAll |
|   ├ Proc17|  GetFilesToUpdate |
|   ├ Proc18|  GetFBProgress |
|   ├ Proc19|  GetFBSize |
|   ├ Proc20|  GetStreamedMegabytes  |
|   ├ Proc21|  GetTotalMegabytes |
|   ├ Proc22|  SetMinimumPriorityLevel |
|   └ Proc23|  GetNumberOfFilesToUpdate ? |
| **8c7fbdb0-8513-44f9-a8b1-1a3b49322bf4**     |   **vfs subsystem**    |
| 	└ Proc0     	|   Get folders redirections   |
| **8d17061c-534a-4f1b-bd77-f615421cf379** | **Virtual registry ??** |
| 	├ Proc0     	|   vreg_server_GetConfiguration ?    |
| 	├ Proc1     	|   ??    |
| 	├ Proc2     	|   ??    |
| 	└ Proc3     	|   ??    |
| **d614caa0-0d0e-4b3f-a285-ce1c560b3d7d**     |   **vregistry ?**    |
| 	├ Proc0     	|   get root folder (HLKM and install dir)   |
| 	└ Proc1     	|   set root folder ?    |
| **d99d2add-4f00-41b8-bf99-bb03127f015f** | ** Virtual Process/Modules ??  **|
| 	├ Proc0     	|   AppV__Client__Virtualization__IsvNotificationListener__add_process    |
| 	└ Proc1     	|   VirtualModuleLoading ?   |
| **edce686d-acae-4a2a-8945-24489443c35e**     |   **vfs subsystem**   |
|   └ Proc0       |   get new ENV variables    |



## References

* [http://www.rump.beer/2017/slides/from_alpc_to_uac_bypass.pdf](http://www.rump.beer/2017/slides/from_alpc_to_uac_bypass.pdf)
* [http://hakril.github.io/PythonForWindows/build/html/index.html](http://hakril.github.io/PythonForWindows/build/html/index.html)
* [NDR transfert syntax](http://pubs.opengroup.org/onlinepubs/9629399/chap14.htm#tagfcjh_30)
* [`_MIDL_STUB_DESC` structure definition](https://msdn.microsoft.com/fr-fr/library/windows/desktop/aa374178(v=vs.85).aspx)