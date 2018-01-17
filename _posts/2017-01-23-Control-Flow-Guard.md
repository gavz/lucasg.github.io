---
layout: post
title: "How Control Flow Integrity is implemented in Windows 10"
date: 2017-02-05
---


This blog post resume a month of work analyzing how Windows has implemented Control Flow Integrity in Win10 build 14393 and 14986.

A lot have been said about CFG, the Windows's version of control flow integrity. It has been released first in Windows 8.1 Update 3 (`KB3000850`) in 2014 and had been improved upon since. Coming recently with the Windows 10 Anniversary Update (i.e. build `14393`) several new features like `longjmp` hardening.

<!--more-->

## Introduction


The rationale behind `CFG` is to ensure dynamically that indirect pointer calls jumps to a valid target. Microsoft teams takes advantage of the fact they control the `PE` executable file format, the compiler and the NT `PE` loader which acts together to provide runtime check of an `icall`'s target address.


`msvc`, the compiler behind Visual Studio, has a link directive `/cfguard` which, when enabled, will wrap all the indirect calls in a given binary by a call to `_guard_check_icall` (and on build 14986 by `_guard_dispatch_icall`). It will also generate a whitelist of `CFG` approved targets within the given binary. This list is stored in the `Load Configuration` data directory in the `PE` header.


The NT loader will, on a module load (see `ntdll!LdrSystemDllInitBlock` ), parse the `Load Configuration` entry to look for `CFG` aware capabilities and, if enabled, will generate a `CFG` bitmap storing all the valid targets address from the `CFG` whitelist in the module.

The wrapper function `_guard_check_icall` is actually a placeholder at compile-time and will be patched by the NT loader on module loading to point to `LdrpValidateUserCallTarget` to do the actual check.


## CFG entries in PE

Currently on build `14393`, the load configuration data directory is the following structure :

{% highlight C %}
typedef struct _IMAGE_LOAD_CONFIG_DIRECTORY64 {
    DWORD      Size;
    DWORD      TimeDateStamp;
    WORD       MajorVersion;
    WORD       MinorVersion;
    DWORD      GlobalFlagsClear;
    DWORD      GlobalFlagsSet;
    DWORD      CriticalSectionDefaultTimeout;
    ULONGLONG  DeCommitFreeBlockThreshold;
    ULONGLONG  DeCommitTotalFreeThreshold;
    ULONGLONG  LockPrefixTable;                // VA
    ULONGLONG  MaximumAllocationSize;
    ULONGLONG  VirtualMemoryThreshold;
    ULONGLONG  ProcessAffinityMask;
    DWORD      ProcessHeapFlags;
    WORD       CSDVersion;
    WORD       DependentLoadFlags;
    ULONGLONG  EditList;                       // VA
    ULONGLONG  SecurityCookie;                 // VA
    ULONGLONG  SEHandlerTable;                 // VA
    ULONGLONG  SEHandlerCount;
    ULONGLONG  GuardCFCheckFunctionPointer;    // VA
    ULONGLONG  GuardCFDispatchFunctionPointer; // VA
    ULONGLONG  GuardCFFunctionTable;           // VA
    ULONGLONG  GuardCFFunctionCount;
    DWORD      GuardFlags;
    IMAGE_LOAD_CONFIG_CODE_INTEGRITY CodeIntegrity;
    ULONGLONG  GuardAddressTakenIatEntryTable; // VA
    ULONGLONG  GuardAddressTakenIatEntryCount;
    ULONGLONG  GuardLongJumpTargetTable;       // VA
    ULONGLONG  GuardLongJumpTargetCount;
    ULONGLONG  DynamicValueRelocTable;         // VA
    ULONGLONG  HybridMetadataPointer;          // VA
} IMAGE_LOAD_CONFIG_DIRECTORY64, *PIMAGE_LOAD_CONFIG_DIRECTORY64;
{% endhighlight C %}


There is a `GuardFlags` members specifying which `CFG` mitigations are met : 

{% highlight C %}
#define IMAGE_GUARD_CF_INSTRUMENTED                    0x00000100 // Module performs control flow integrity checks using system-supplied support
#define IMAGE_GUARD_CFW_INSTRUMENTED                   0x00000200 // Module performs control flow and write integrity checks
#define IMAGE_GUARD_CF_FUNCTION_TABLE_PRESENT          0x00000400 // Module contains valid control flow target metadata
#define IMAGE_GUARD_SECURITY_COOKIE_UNUSED             0x00000800 // Module does not make use of the /GS security cookie
#define IMAGE_GUARD_PROTECT_DELAYLOAD_IAT              0x00001000 // Module supports read only delay load IAT
#define IMAGE_GUARD_DELAYLOAD_IAT_IN_ITS_OWN_SECTION   0x00002000 // Delayload import table in its own .didat section (with nothing else in it) that can be freely reprotected
#define IMAGE_GUARD_CF_EXPORT_SUPPRESSION_INFO_PRESENT 0x00004000 // Module contains suppressed export information
#define IMAGE_GUARD_CF_ENABLE_EXPORT_SUPPRESSION       0x00008000 // Module enables suppression of exports
#define IMAGE_GUARD_CF_LONGJUMP_TABLE_PRESENT          0x00010000 // Module contains longjmp target information
{% endhighlight C %}

They are three optional tables for `CFG` protection:

- `GuardCFFunctionTable`
- `GuardAddressTakenIatEntryTable`
- `GuardLongJumpTargetTable`

Each table is an virtual address to an array of RVA entries concatenated with an optional flag "header" based on which `GuardFlags` flag is set. In theory the flags header can be up to 8 bytes, but it probably won't exceed a byte.

	+============+-------++============+-------++============+-------++============+-------+
	|     RVA    | flags ||     RVA    | flags ||     RVA    | flags ||     RVA    | flags |
	+============+-------++============+-------++============+-------++============+-------+
	        Entry #1              Entry #2              Entry #3              Entry #4

## GuardCFFunctionTable

The `GuardCFFunctionTable` table is historically the first `CFG` mitigations implemented. It protects all the indirect calls in userland as well as in kernel mode. You can dump it from a `PE` via the following command : `dumpbin /HEADERS /LOADCONFIG "some_executable.exe"`.

Some executables have extended CFG entries (i.e. with a header) in their load configuration, like `ntdll.dll` or `kernel32.dll`. The only flag used for the moment in user mode is the bit 0, marking a `CFG` target as a `SuppressedCall`.

If you try to indirectly call a `CFG` approved target containing a `SuppressedCall` bit flag (like `kernelbase!GetProcAddress`), the process will crash. If you look at some `SuppressedCall` targets, you can notice they are not so innocent looking :

* `kernelbase`:
	* `RaiseException`
	* `GetProcAddress`/`GetProcAddressForCaller`
	* `SetProcessValidCallTargets`
	* `SwitchToFiber`
	* `SetProtectedPolicy`
	* `SetThreadContext`
	* `RtlRestoreContext` (14986)

* `kernel32`:
	* `RtlUnwindExStub`
	* `GetProcAddressStub`
	* `RtlRestoreContextStub`
	* `ExecuteUmsThread`
	* `(Wow64)SetThreadContextStub`
	* `UmsThreadYield`

* `ntdll`:
	* `LdrGetProcedureAddress(Ex)`
	* `LdrInitializeThunk`
	* `longjmp`
	* `_C_specific_handler`
	* `RtlGuardRestoreContext`
	* `RtlSetProtectedPolicy`
	* `RtlFindExportedRoutineByName`
	* `KiUserApcDispatch` (14986)
	* `KiUserException` (14986)
	* `RtlProtectHeap` (14986)
	* `NtContinue` (14986)
	* `NtSetInformationVirtualMemory` (14986)

From this list we can infer most of them either have a direct impact on the way `CFG` is implemented (`SetProcessValidCallTargets`, `SetProtectedPolicy`, `RtlSetProtectedPolicy`) or are legit control flow mechanisms (`RaiseException`, `RtlRestoreContextStub`, `SwitchToFiber`, `longjmp`). 

It is understable that these "dangerous" functions should be allowed to be called indirectly, but they should be deactivated by default. The function which explicitly activate the call target is `ntdll!RtlpGuardGrantSuppressedCallAccess` and for obvious reasons is not `CFG` whitelisted.

The `GuardCFFunctionTable` and `SuppressedCall` header flag parsing has been added to Process Hacker's `peview.exe` executable [(Github PR)](https://github.com/processhacker2/processhacker2/pull/101):

![If you try to look at Kernelbase's GuardCFFunctionTable in IDA, it will look ugly since IDA's PE parser does not parse correctly extended CFG entries.](/assets/PeView_with_cfg.PNG)

There is also an unknown header flag (bit 1) located in the `CFGuardFunctions` table entries present only in kernel-side `PE's`, but I haven't found the reason behind it or where it's used.

## GuardAddressTakenIatEntryTable

Only drivers PE's contains entries in `CFGuardAddressInIatEntry`, indicating a kernel-side protection. We know `IDT`, `GDT`, `SSDT`and `MSR` hooking are already watched and protected by `PatchGuard`.

The `GuardAddressTakenIatEntryTable` extend those protections by marking some imports as protected through the use of `ntosknrl!MiProcessKernelCfgAddressTakenImports`. The protection looks like to be done in `Virtual Secure Mode (VSM)`. Since I don't have a `VSM` enabled system around, I can't say much about this mitigation.

If you want to know more about `VSM`, [A.Ionescu](https://twitter.com/aionescu) bothered to investigate `VSM` and `Hyper-V` hyperguard protection here :

* [http://www.alex-ionescu.com/syscan2015.pdf](http://www.alex-ionescu.com/syscan2015.pdf)
* [http://www.alex-ionescu.com/blackhat2015.pdf](http://www.alex-ionescu.com/blackhat2015.pdf)



## CFGuardLongJumpTarget

The MS CRT (`msvcrt`, `libcmt`, etc.) has a non-local `goto` functionality using `_setjmp`/`longjmp` coming from the Unix world which completely bypass CFG *by design*. Since it's a legit control flow feature (although I never used it) MS cannot just blacklist it.

Instead, on a `longjmp` call, the process will call `kernelbase!GuardCheckLongJumpTargetImpl` which check that the target stack pointer (`*sp`) is actually located in the thread stack space.

For more information on this security feature, read this extremely detailed blog post : `http://blog.trendmicro.com/trendlabs-security-intelligence/control-flow-guard-improvements-windows-10-anniversary-update`


## RFG

`RFG` has not been released yet in build 14986. Once it's out there, this section will be improved.
In the meantime, you can read this white paper : [http://xlab.tencent.com/en/2016/11/02/return-flow-guard](http://xlab.tencent.com/en/2016/11/02/return-flow-guard)


## Conclusion

With `CFG`, Microsoft and the Windows team has done a tremendous job of raising the bar for attackers to actually use their exploit primitives. Combined with `RFG`, `CFG` has the potential of annihilating memory corruption exploits. (well the same thing was also said for `ASLR+DEP` before popularization of `ROP` and `heap spraying` techniques)

One thing to mention is `CFG` is the second mitigation technique taking advantage of the fact that Windows is an integrated environment, the first being `KASLR`. It is interesting from a "marketing" point of view : `CFG` can not be easily ported in a `linux` since there is a myriad of C CRT running in the same userland. We might see more mitigations techniques based on this uniqueness in the future.

Alas, `CFG` is not the perfect mitigiation technique (none really are). Firstly it only check the call target, not the call arguments. Secondly, the backward compatibility legacy code does not benefit from `CFG` until they are re-compiled by a compiler >= VS2015. Lastly, there are already some `CFG` bypass publicly present on the Internet : [https://medium.com/@mxatone/mitigation-bounty-4-techniques-to-bypass-mitigations-2d0970147f83#.a5oytwbnq](@mxatone/mitigation-bounty-4-techniques-to-bypass-mitigations). 


Maybe in the future a new type of generic bypass will be invented (like `ROP` for `DEP`) but actually all the existing bypass relies on pretty strong hypothesis (`CLR` module loaded, etc.). Here is the take of a `pwn2own` winner on current NT kernel security state and post-CFG exploitation : [http://www.slideshare.net/PeterHlavaty/you-didnt-see-its-coming-dawn-of-hardened-windows-kernel](you-didnt-see-its-coming-dawn-of-hardened-windows-kernel)


## References :

- MS blog post from CFG dev : [https://blogs.msdn.microsoft.com/vcblog/2014/12/08/visual-studio-2015-preview-work-in-progress-security-feature/](https://blogs.msdn.microsoft.com/vcblog/2014/12/08/visual-studio-2015-preview-work-in-progress-security-feature/)
- Overview of CFG : [https://blog.trailofbits.com/2016/12/27/lets-talk-about-cfi-microsoft-edition](https://blog.trailofbits.com/2016/12/27/lets-talk-about-cfi-microsoft-edition)
- CFG From windows : [https://msdn.microsoft.com/en-us/library/windows/desktop/mt637065(v=vs.85).aspx](https://msdn.microsoft.com/en-us/library/windows/desktop/mt637065(v=vs.85).aspx)
- GrantSuppressedCallAccess : [https://breakingmalware.com/documentation/documenting-undocumented-adding-control-flow-guard-exceptions/](https://breakingmalware.com/documentation/documenting-undocumented-adding-control-flow-guard-exceptions/)
- CFG limitations : [https://blog.scrt.ch/2017/01/27/exploiting-a-misused-c-shared-pointer-on-windows-10](https://blog.scrt.ch/2017/01/27/exploiting-a-misused-c-shared-pointer-on-windows-10/)
- setjmp : [http://man7.org/linux/man-pages/man3/setjmp.3.html](http://man7.org/linux/man-pages/man3/setjmp.3.html)
- setjmp in MS : [https://msdn.microsoft.com/en-us/library/yz2ez4as.aspx](https://msdn.microsoft.com/en-us/library/yz2ez4as.aspx)
- LongJump hardening : [http://blog.trendmicro.com/trendlabs-security-intelligence/control-flow-guard-improvements-windows-10-anniversary-update](http://blog.trendmicro.com/trendlabs-security-intelligence/control-flow-guard-improvements-windows-10-anniversary-update)
- RFG : [http://xlab.tencent.com/en/2016/11/02/return-flow-guard](http://xlab.tencent.com/en/2016/11/02/return-flow-guard)
