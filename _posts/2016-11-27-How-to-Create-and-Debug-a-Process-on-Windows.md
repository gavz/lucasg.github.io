---
layout: post
title: "How to create and debug a process automatically on windows"
date: 2016-11-27
---


Recently I had to write a custom loader which will dynamically retrieve a bunch of informations (loaded modules list, imports, etc.) for several hundreds of executables. The first way involves to launch every exe with ```cdb``` and cry when it comes to write windbg scripts in order to exports the needed infomations (I'm not frankly excited about <a href = "https://blogs.msdn.microsoft.com/windbg/2016/10/27/new-insider-sdk-and-javascript-extensibility/"> their new shiny Javascript scripting engine built for windbg </a>). The other way is to write a lightweight debugger using specific Windows API.


Guess which way I went ?
<!--more-->



[tldr : give me the code !](#tldr-code)

I felt to write a blog post about it since the functions used here (```psapi``` basically) are confusing to use and sometimes overlapping. Finally they are not as well documented (particularly on error cases) as the rest of Win32 API. Here a tutorial on how to retrieve (as in rva, not statically) the base module address and entry point of an executable.

## The naive way :

- - - - - - -

{% highlight c %}
#include <windows.h>

int main(int argc, char *argv[])
{
  STARTUPINFO si = {0};
  PROCESS_INFORMATION pi = {0};

  if (argc <= 1)
    return 0x01;

  // Start process
  si.cb = sizeof(STARTUPINFO);
  if (!CreateProcess(0, argv[1], 0, 0, TRUE, 0, 0, 0, &si, &pi))
  {
    return 0x01;
  }

  // Do stuff here

  // Clean process
  TerminateProcess(pi.hProcess, 0);
  return 0x00;
}

{% endhighlight c %}

This approach has two problems :
  
Firstly, ```CreateProcess``` does not have a way to prevent the created process from running. Unless you have a way to control the created process (since you have the source for example) most
binaries will just exit before our program is able to look into their memory. Because of this issue, we can only use this method on executables with a message pump (GUI, servers, etc.) which
will prevent them from an early exit.

Secondly, the process identifier returned in ```PROCESS_INFORMATION.dwProcessId``` may not always be the created process PID !

![Returned PID for the mstsc create process](/assets/created_mstsc_process.png)
![ProcessHacker is a beast of program !](/assets/created_mstsc_process_phacker.png)

For some process, the returned PID (```8440``` in the example) does not refer to the created process PID (```10400```) but it's parent. I did not exactly pinpoint the root cause of this behaviour, but nevertheless it's pretty annoying to handle : we need to iterate over every running process and find which has our PID as a parent.


{% highlight c %}

DWORD GetCreatedProcessPID(DWORD ControllerPID)
{
  DWORD UniquePID = NULL;
  HANDLE  hProcessSnap = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
  PROCESSENTRY32 ProcessInformation;
  ProcessInformation.dwSize = sizeof(PROCESSENTRY32);


  Process32First(hProcessSnap, &ProcessInformation);
  while (ProcessInformation.th32ParentProcessID != ControllerPID)
  {
    Process32Next(hProcessSnap, &ProcessInformation);

    // Found child process
    if (ProcessInformation.th32ParentProcessID == ControllerPID)
    {
      UniquePID = ProcessInformation.th32ProcessID;
    }
  }

  CloseHandle(hProcessSnap);
  return UniquePID;
}

{% endhighlight c %}

Anyway, not a good way to go.


## The suspended way :

- - - - - - -

CreateProcess have a bunch of creations options : <a href ="https://msdn.microsoft.com/en-us/library/windows/desktop/ms684863(v=vs.85).aspx."> Process Creation Flags </a>. ```CREATE_SUSPENDED``` is an interesting one :

<pre>
<code>

CREATE_SUSPENDED    The primary thread of the new process is created in a suspended state
0x00000004      and does not run until the ResumeThread function is called.

</code>
</pre>

Basically we can't create any process and get back our informations whithout running it. Unfortunately, reality is not as nice.

{% highlight c %}
#include <windows.h>
#include <stdio.h>
#include <TlHelp32.h>


DWORD_PTR GetExeModuleBase(DWORD dwProcessId)
{
  MODULEENTRY32 lpModuleEntry = { 0 };
  HANDLE hSnapShot = 0;

  hSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, dwProcessId);
  if (hSnapShot == INVALID_HANDLE_VALUE)
  {
    printf("Could not snap process : %d \n", GetLastError());
    return 0x00;
  }

  // Executable is always the first module loaded
  lpModuleEntry.dwSize = sizeof(lpModuleEntry);
  if (!Module32First(hSnapShot, &lpModuleEntry))
  {
    printf("Could not enumerate the first module : %d \n", GetLastError());
    CloseHandle(hSnapShot);
    return 0x00;
  }   

  CloseHandle(hSnapShot);
  return (DWORD_PTR)lpModuleEntry.modBaseAddr;
}


int main(int argc, char *argv[])
{
  STARTUPINFO si = {0};
  PROCESS_INFORMATION pi = {0};

  if (argc <= 1)
    return 0x01;

  // Start process
  si.cb = sizeof(STARTUPINFO);
  if (!CreateProcess(0, argv[1], 0, 0, TRUE, CREATE_SUSPENDED, 0, 0, &si, &pi))
  {
    return 0x01;
  }
  printf("Created process %s : %d \n", argv[1], pi.dwProcessId);
  // Do stuff here
  printf("Process base address : %p \n", (void*) GetExeModuleBase (pi.dwProcessId));

  // Clean process
  TerminateProcess(pi.hProcess, 0);
  CloseHandle(pi.hProcess);
  CloseHandle(pi.hThread);
  return 0x00;
}
{% endhighlight c %}


<pre>
<code>
OUTPUT : 
------------------------------------------------------
Created process C:\Windows\system32\mspaint.exe : 19288
Could not snap process : 299
Process base address : 0000000000000000    
</code>
</pre>

A non-invasive ```cdb``` injection reveal what's happening :
![Windbg is kinda lost here](/assets/suspended_mspaint_process.png)

We don't have any informations since nothing has been loaded ! ```CREATE_SUSPENDED``` is 
useful for separating the process creation from the process run, not for our use case.

We need to do the whole system initialization (system dll loading, x86/x64 emulation layer init, etc.) and stop on a System breakpoint. That's why we're going the debug way.

                                                                          
##  <a name="tldr-code"></a> The debug way :

- - - - - - -

The only "proper" way to do it is to write a simple debugger which can allow the debugged process to do all the loading and stop just before jumping in to its entry point. Here is the code necessary :


``` DebugProcess.h ```

{% highlight c %}
#pragma once
#include <windows.h>
#include <stdbool.h>
#include <stdint.h>

// Opaque handle for this API.
typedef HANDLE DBG_PROC_HANDLE;

typedef struct DEBUG_PROCESS_INFOS_T_
{
  uintptr_t ExeBaseAddress;
  uintptr_t ExeEntryPoint;
  size_t    ProcessPID;

} DEBUG_PROCESS_INFOS;

// Launch a process and break it before executing any of it's code.
bool FreezeProcessOnStartup(const TCHAR *tExePath, DBG_PROC_HANDLE *phDbgProc);

// Get Basic informations about debugged process
bool GetProcessInfos(DBG_PROC_HANDLE hDbgProc, DEBUG_PROCESS_INFOS *pDbgProcInfos);

// Stop debugging process and kill it.
bool StopProcess(DBG_PROC_HANDLE hDbgProc);
{% endhighlight c %}

``` DebugProcess.c ```
{% highlight c %}
#include "DebugProcess.h"
#include <psapi.h>
#include <tlhelp32.h>
#include <tchar.h>

typedef struct DEBUG_PROCESS_T_
{
  HANDLE hDebuggedProc;
  HANDLE hDebuggedThread;
  size_t DebuggedPID;

} DEBUG_PROCESS_T;

// Return the target executable module base address
DWORD_PTR PrivateGetExeModuleBase(size_t dwProcessId)
{
  MODULEENTRY32 lpModuleEntry = { 0 };  
  HANDLE hProcessSnapShot = 0x00;
  DWORD_PTR ModuleBaseAddress = 0x00;

  hProcessSnapShot = CreateToolhelp32Snapshot(TH32CS_SNAPMODULE, (DWORD) dwProcessId);
  if (INVALID_HANDLE_VALUE == hProcessSnapShot)
    return 0x00;

  // Executable is always the first module loaded
  lpModuleEntry.dwSize = sizeof(lpModuleEntry);
  if (Module32First(hProcessSnapShot, &lpModuleEntry))
  {
    ModuleBaseAddress = (DWORD_PTR)lpModuleEntry.modBaseAddr;
  }

  CloseHandle(hProcessSnapShot);
  return ModuleBaseAddress;
}

// Return the target executable entry point (if it does exist)
uintptr_t PrivateGetExeEntryPoint(size_t dwProcessId)
{
  HANDLE hProcess = NULL;
  HMODULE hMods[1024];
  DWORD cbNeeded;
  MODULEINFO ModuleInfo;
  uintptr_t EntryPoint = NULL;
  
  hProcess = OpenProcess(PROCESS_QUERY_INFORMATION | PROCESS_VM_READ, FALSE, (DWORD) dwProcessId);
  if (!hProcess)
  {
    goto PrivateGetExeEntryPoint_END;
  }

  // Executable is always the first module loaded
  if (!EnumProcessModules(hProcess, hMods, sizeof(hMods), &cbNeeded))
  {
    goto PrivateGetExeEntryPoint_END;
  }
  
  if (!GetModuleInformation(hProcess, hMods[0], &ModuleInfo, sizeof(MODULEINFO)))
  {
    goto PrivateGetExeEntryPoint_END;
  }

  EntryPoint = (uintptr_t) ModuleInfo.EntryPoint;

PrivateGetExeEntryPoint_END:
  if (hProcess) 
  {
    CloseHandle(hProcess);
  }
  
  return EntryPoint;
}

bool FreezeProcessOnStartup(const TCHAR *tExePath, DBG_PROC_HANDLE *phDbgProc)
{
  STARTUPINFO si = {0};
  PROCESS_INFORMATION pi = {0};
  DEBUG_EVENT debugEvent = { 0 };
  DEBUG_PROCESS_T* pPrivateDbgProc = 0;

  // allocate private memory for returned handle
  pPrivateDbgProc = (DEBUG_PROCESS_T*) calloc (1, sizeof(DEBUG_PROCESS_T));
  if (!pPrivateDbgProc)
    return false;


  // Create target process
  si.cb = sizeof(STARTUPINFO);
  if (!CreateProcess(0, (TCHAR*) tExePath, 0, 0, TRUE, DEBUG_PROCESS, 0, 0, &si, &pi))
  {
    goto FreezeProcessOnStartup_ERROR;
  }

  // Check whether calling and debugged process have the same arch
  // In theory x64 process can debug x86 process, but to do so we need
  // to handle the Wow64 emulation layer (EXCEPTION_DEBUG_EVENT will fire 
  // when all x64 dll are loaded, but x86 aren't) and it's a major hassle.
  BOOL bTargetProcessArch = FALSE;
  BOOL bCallerProcessArch = FALSE;
  IsWow64Process(pi.hProcess, &bTargetProcessArch);
  IsWow64Process(GetCurrentProcess(), &bCallerProcessArch);
  if (bTargetProcessArch != bCallerProcessArch)
  {
    // Indicating the error type for the caller to understand why it failed.
    SetLastError(ERROR_BAD_EXE_FORMAT);
    goto FreezeProcessOnStartup_ERROR;
  }   

  pPrivateDbgProc-> hDebuggedProc = pi.hProcess;
  pPrivateDbgProc-> hDebuggedThread = pi.hThread;
  pPrivateDbgProc-> DebuggedPID = pi.dwProcessId;

  // Monitoring debug events for EXECPTION_DEBUG_EVENT,
  // indicating our process has loaded all the needed dll and resources
  // and it's ready to go.
  DebugSetProcessKillOnExit(TRUE);
  DebugActiveProcess(pi.dwProcessId);

  do {
    if (!WaitForDebugEvent(&debugEvent, 10*1000 /*INFINITE*/))
    {
      // Timeout => go on error;
      goto FreezeProcessOnStartup_ERROR;
    }

    // Stop on debug exception
    switch (debugEvent.dwDebugEventCode)
    {
    case EXCEPTION_DEBUG_EVENT:
      goto FreezeProcessOnStartup_SUCCESS;

    case UNLOAD_DLL_DEBUG_EVENT:
      // Allow the x64 system dlls to be unloaded when dealing with a x86 process
      // To be really clean, we need to check if the currently unloaded dll is a 
      // well-known DLL. Unfortunately, there is no easy way to do so.
      if (!bTargetProcessArch )
      {
        goto FreezeProcessOnStartup_ERROR;
      }
      
    case LOAD_DLL_DEBUG_EVENT:
    case CREATE_THREAD_DEBUG_EVENT:
    case CREATE_PROCESS_DEBUG_EVENT:
      ContinueDebugEvent(
          debugEvent.dwProcessId,
          debugEvent.dwThreadId,
          DBG_CONTINUE);
      break;

    default:
      goto FreezeProcessOnStartup_ERROR;
    }   
    
  } while (true);


  // return a null handle on error => the caller need to check it.
FreezeProcessOnStartup_ERROR:
  
  StopProcess((DBG_PROC_HANDLE) pPrivateDbgProc); // mem free is done here
  pPrivateDbgProc = NULL;
  
  return false;

  
FreezeProcessOnStartup_SUCCESS:
  *phDbgProc = (DBG_PROC_HANDLE) pPrivateDbgProc;
  return true;
}

// Get Basic informations about debugged process
bool GetProcessInfos(DBG_PROC_HANDLE hDbgProc, DEBUG_PROCESS_INFOS *pDbgProcInfos)
{
  DEBUG_PROCESS_T* pPrivateDbgProc = (DEBUG_PROCESS_T*) hDbgProc;

  if (!pPrivateDbgProc)
    return false;

  memset(pDbgProcInfos, 0, sizeof (DEBUG_PROCESS_INFOS));
  
  pDbgProcInfos->ExeBaseAddress = PrivateGetExeModuleBase(pPrivateDbgProc->DebuggedPID);
  // No module can be mapped at null address
  if (!pDbgProcInfos->ExeBaseAddress)
    return false;

  pDbgProcInfos->ExeEntryPoint = PrivateGetExeEntryPoint(pPrivateDbgProc->DebuggedPID);
  // No entry point => is it a dll ?
  if (!pDbgProcInfos->ExeEntryPoint)
    return false;

  pDbgProcInfos->ProcessPID = pPrivateDbgProc->DebuggedPID;
  return true;
}

bool StopProcess(DBG_PROC_HANDLE hDbgProc)
{
  DEBUG_PROCESS_T* pPrivateDbgProc = (DEBUG_PROCESS_T*) hDbgProc;

  if (!pPrivateDbgProc)
    return false;

  // Kill target process
  DebugActiveProcessStop((DWORD) pPrivateDbgProc->DebuggedPID);
  TerminateProcess(pPrivateDbgProc->hDebuggedProc, 0);

  // Cleanup
  CloseHandle(pPrivateDbgProc->hDebuggedProc);
  CloseHandle(pPrivateDbgProc->hDebuggedThread);
  free(pPrivateDbgProc);

  return true;
}
{% endhighlight c %}



## Final comments

- - - - - - -

The last method is not a silver bullet. Some limitations I've noticed :

  * There are some hard (and hidden) incompatibilities between x86 and x64 on Windows. The <a href="https://msdn.microsoft.com/library/windows/desktop/ms682489(v=vs.85).aspx"> MSDN documentation about CreateToolhelp32Snapshot specify that x86 callers process cannot debug x64 process</a> and the function will return the ever cryptic ```ERROR_PARTIAL_COPY``` error status in that particular case.

  * In the mirror case (x64 caller debugging a x86 process) it is possible, but way more complicated than for same-arch process injection. The WOW64 emulation layer will
  load and unload system dll like ```kernel32.dll``` or ```user32.dll``` before loading their x86 equivalent from the ```SysWOW64``` folder. You have to take that corner case in the debugger main loop (knowing that the API is not really well designed since you can't know on a ```DLL_UNLOAD_EVENT``` event the name of the DLL being unloaded ) and only accept ```x64 Known Dlls``` to be unloaded.

  * In the general case, the loader might also miss <a href="http://resources.infosecinstitute.com/debugging-tls-callbacks/">tls callbacks</a> being fired up and others treacheries in PE's. Writing debuggers is a full-time job, you know ...