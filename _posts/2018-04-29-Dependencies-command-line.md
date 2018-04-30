---
layout: post
title: "Dependencies command line tool"
date: 2018-04-29
---


I've noticed for quite some time that my tool [Dependencies](https://lucasg.github.io/Dependencies/) is popping up in various support forums and discussion to help users diagnosticate their load dll issues, which is great. Interestingly enough, most of my pingbacks comes from China.

However, Dependencies is a GUI tool, which doesn't make the ideal tool for people to report their issue via a support forum (which is usually text oriented). Most people end up screenshoting the analysis window like here [http://forum.fritzing.org/t/start-up-error-message-openssl/5209/9](http://forum.fritzing.org/t/start-up-error-message-openssl/5209/9) or here [https://forum.qt.io/topic/87700/still-trying-to-deploy-using-windeployqt-exe/15](https://forum.qt.io/topic/87700/still-trying-to-deploy-using-windeployqt-exe/15). A text version for the dependency chain would be a great addition to Dependencies.

<!--more-->

Since I got the explicit demand for a CLI based tool ([https://github.com/lucasg/Dependencies/issues/14](https://github.com/lucasg/Dependencies/issues/14)) I finally decided to do it.

Starting with `v1.7` ([https://github.com/lucasg/Dependencies/releases/tag/v1.7](https://github.com/lucasg/Dependencies/releases/tag/v1.7)), Dependencies ships two executables :

* `DependenciesGui.exe` which is the usual GUI tool (formerly called `Dependencies.exe`)
* `Dependencies.exe` which is the new CLI tool (formerly called `ClrPhTester.exe`)


`Dependencies.exe` support the following command line switches :

-  `-h` or `-help`  display the help and usage
-  `-json`  activate json output formatting
-  `-apisets`  dump the system's ApiSet schema (api set dll -> host dll)
-  `-knowndll` dump all the system's known dlls (x86 and x64)
-  `-manifest` dump FILE embedded manifest, if it exists.
-  `-sxsentries` dump all of FILE's sxs dependencies.
-  `-imports` dump FILE imports
-  `-exports` dump  FILE exports
-  `-modules` dump FILE resolved modules
-  `-chain`  dump FILE whole dependency chain

the `-chain` and `-modules` commands need to resolve the input binary file's dependencies, so it make take a while to return.

Every command switches support JSON output formatting if you need to return results in a computer-consumable way and automate reporting. However there is two notables exceptions : `-chain` and `-modules`, the reason being that I need to figure out how to deal with circular references in Json output.

Below this post are examples of commands availables :

- [ApiSet](#apiset)
- [Known dlls](#knowndlls)
- [Imports](#imports)
- [Exports](#exports)
- [Manifest](#manifest)
- [SxS dependencies](#sxsentries)
- [Module dependencies](#modules)
- [Dependency chain](#chain)

I hope this tool can help developers resolved missing dll dependencies on users' systems more easily and reliably.


### ApiSet

```
PS > .\Dependencies.exe -apisets
[-] Api Sets Map :
api-ms-onecoreuap-print-render-l1-1 -> [ printrenderapihost.dll ]
api-ms-onecoreuap-settingsync-status-l1-1 -> [ settingsynccore.dll ]
api-ms-win-appmodel-identity-l1-2 -> [ kernel.appcore.dll ]
api-ms-win-appmodel-runtime-internal-l1-1 -> [ kernel.appcore.dll ]
api-ms-win-appmodel-runtime-l1-1 -> [ kernel.appcore.dll ]
api-ms-win-appmodel-state-l1-1 -> [ kernel.appcore.dll ]
api-ms-win-appmodel-state-l1-2 -> [ kernel.appcore.dll ]
api-ms-win-appmodel-unlock-l1-1 -> [ kernel.appcore.dll ]
api-ms-win-base-bootconfig-l1-1 -> [ advapi32.dll ]
api-ms-win-base-util-l1-1 -> [ advapi32.dll ]
api-ms-win-composition-redirection-l1-1 -> [ dwmredir.dll ]
api-ms-win-composition-windowmanager-l1-1 -> [ udwm.dll ]
api-ms-win-core-apiquery-l1-1 -> [ ntdll.dll ]
[...]
ext-ms-win-wlan-scard-l1-1 -> [ winscard.dll ]
ext-ms-win-wnv-l1-1 -> [ wnv.sys ]
ext-ms-win-wpn-phoneext-l1-1 -> [  ]
ext-ms-win-wrp-sfc-l1-1 -> [ sfc.dll ]
ext-ms-win-wsclient-devlicense-l1-1 -> [ wsclient.dll ]
ext-ms-win-wwaext-misc-l1-1 -> [ wwaext.dll ]
ext-ms-win-wwaext-module-l1-1 -> [ wwaext.dll ]
ext-ms-win-wwan-wwapi-l1-1 -> [ wwapi.dll ]
ext-ms-win-xaml-controls-l1-1 -> [ windows.ui.xaml.phone.dll ]
ext-ms-win-xaml-pal-l1-1 -> [  ]
ext-ms-win-xblauth-console-l1-1 -> [  ]
ext-ms-win-xboxlive-xboxnetapisvc-l1-1 -> [  ]
ext-ms-windowscore-deviceinfo-l1-1 -> [  ]
```

```
PS > .\Dependencies.exe -apisets -json
{
  "Schema": {
    "api-ms-onecoreuap-print-render-l1-1": [
      "printrenderapihost.dll"
    ],
    "api-ms-onecoreuap-settingsync-status-l1-1": [
      "settingsynccore.dll"
    ],
    "api-ms-win-appmodel-identity-l1-2": [
      "kernel.appcore.dll"
    ],
    "api-ms-win-appmodel-runtime-internal-l1-1": [
      "kernel.appcore.dll"
    ],
    "api-ms-win-appmodel-runtime-l1-1": [
      "kernel.appcore.dll"
    ],
    "api-ms-win-appmodel-state-l1-1": [
      "kernel.appcore.dll"
    ],
    "api-ms-win-appmodel-state-l1-2": [
      "kernel.appcore.dll"
    ],
    "api-ms-win-appmodel-unlock-l1-1": [
      "kernel.appcore.dll"
    ],
[...]
    "ext-ms-win-wwaext-module-l1-1": [
      "wwaext.dll"
    ],
    "ext-ms-win-wwan-wwapi-l1-1": [
      "wwapi.dll"
    ],
    "ext-ms-win-xaml-controls-l1-1": [
      "windows.ui.xaml.phone.dll"
    ],
    "ext-ms-win-xaml-pal-l1-1": [
      ""
    ],
    "ext-ms-win-xblauth-console-l1-1": [
      ""
    ],
    "ext-ms-win-xboxlive-xboxnetapisvc-l1-1": [
      ""
    ],
    "ext-ms-windowscore-deviceinfo-l1-1": [
      ""
    ]
  }
}

```


### KnownDlls

```
PS > .\Dependencies.exe -knowndll
[-] 64-bit KnownDlls :
  C:\WINDOWS\system32\advapi32.dll
  C:\WINDOWS\system32\bcryptPrimitives.dll
  C:\WINDOWS\system32\cfgmgr32.dll
  C:\WINDOWS\system32\clbcatq.dll
  C:\WINDOWS\system32\combase.dll
  C:\WINDOWS\system32\COMCTL32.dll
  C:\WINDOWS\system32\COMDLG32.dll
  C:\WINDOWS\system32\coml2.dll
  C:\WINDOWS\system32\CRYPT32.dll
  C:\WINDOWS\system32\difxapi.dll
  C:\WINDOWS\system32\FLTLIB.DLL
  C:\WINDOWS\system32\gdi32.dll
  C:\WINDOWS\system32\gdi32full.dll
  C:\WINDOWS\system32\gdiplus.dll
  C:\WINDOWS\system32\IMAGEHLP.dll
  C:\WINDOWS\system32\IMM32.dll
  C:\WINDOWS\system32\kernel.appcore.dll
  C:\WINDOWS\system32\kernel32.dll
  C:\WINDOWS\system32\KERNELBASE.dll
  C:\WINDOWS\system32\MSASN1.dll
  C:\WINDOWS\system32\MSCTF.dll
  C:\WINDOWS\system32\msvcp_win.dll
  C:\WINDOWS\system32\MSVCRT.dll
  C:\WINDOWS\system32\NORMALIZ.dll
  C:\WINDOWS\system32\NSI.dll
  C:\WINDOWS\system32\ntdll.dll
  C:\WINDOWS\system32\ole32.dll
  C:\WINDOWS\system32\OLEAUT32.dll
  C:\WINDOWS\system32\powrprof.dll
  C:\WINDOWS\system32\profapi.dll
  C:\WINDOWS\system32\PSAPI.DLL
  C:\WINDOWS\system32\rpcrt4.dll
  C:\WINDOWS\system32\sechost.dll
  C:\WINDOWS\system32\Setupapi.dll
  C:\WINDOWS\system32\SHCORE.dll
  C:\WINDOWS\system32\SHELL32.dll
  C:\WINDOWS\system32\SHLWAPI.dll
  C:\WINDOWS\system32\ucrtbase.dll
  C:\WINDOWS\system32\user32.dll
  C:\WINDOWS\system32\win32u.dll
  C:\WINDOWS\system32\windows.storage.dll
  C:\WINDOWS\system32\WINTRUST.dll
  C:\WINDOWS\system32\WLDAP32.dll
  C:\WINDOWS\system32\wow64.dll
  C:\WINDOWS\system32\wow64cpu.dll
  C:\WINDOWS\system32\wow64win.dll
  C:\WINDOWS\system32\WS2_32.dll

[-] 32-bit KnownDlls :
  C:\WINDOWS\SysWOW64\advapi32.dll
  C:\WINDOWS\SysWOW64\bcryptPrimitives.dll
  C:\WINDOWS\SysWOW64\cfgmgr32.dll
  C:\WINDOWS\SysWOW64\clbcatq.dll
  C:\WINDOWS\SysWOW64\combase.dll
  C:\WINDOWS\SysWOW64\COMCTL32.dll
  C:\WINDOWS\SysWOW64\COMDLG32.dll
  C:\WINDOWS\SysWOW64\coml2.dll
  C:\WINDOWS\SysWOW64\CRYPT32.dll
  C:\WINDOWS\SysWOW64\difxapi.dll
  C:\WINDOWS\SysWOW64\FLTLIB.DLL
  C:\WINDOWS\SysWOW64\gdi32.dll
  C:\WINDOWS\SysWOW64\gdi32full.dll
  C:\WINDOWS\SysWOW64\gdiplus.dll
  C:\WINDOWS\SysWOW64\IMAGEHLP.dll
  C:\WINDOWS\SysWOW64\IMM32.dll
  C:\WINDOWS\SysWOW64\kernel.appcore.dll
  C:\WINDOWS\SysWOW64\kernel32.dll
  C:\WINDOWS\SysWOW64\KERNELBASE.dll
  C:\WINDOWS\SysWOW64\MSASN1.dll
  C:\WINDOWS\SysWOW64\MSCTF.dll
  C:\WINDOWS\SysWOW64\msvcp_win.dll
  C:\WINDOWS\SysWOW64\MSVCRT.dll
  C:\WINDOWS\SysWOW64\NORMALIZ.dll
  C:\WINDOWS\SysWOW64\NSI.dll
  C:\WINDOWS\SysWOW64\ntdll.dll
  C:\WINDOWS\SysWOW64\ole32.dll
  C:\WINDOWS\SysWOW64\OLEAUT32.dll
  C:\WINDOWS\SysWOW64\powrprof.dll
  C:\WINDOWS\SysWOW64\profapi.dll
  C:\WINDOWS\SysWOW64\PSAPI.DLL
  C:\WINDOWS\SysWOW64\rpcrt4.dll
  C:\WINDOWS\SysWOW64\sechost.dll
  C:\WINDOWS\SysWOW64\Setupapi.dll
  C:\WINDOWS\SysWOW64\SHCORE.dll
  C:\WINDOWS\SysWOW64\SHELL32.dll
  C:\WINDOWS\SysWOW64\SHLWAPI.dll
  C:\WINDOWS\SysWOW64\ucrtbase.dll
  C:\WINDOWS\SysWOW64\user32.dll
  C:\WINDOWS\SysWOW64\win32u.dll
  C:\WINDOWS\SysWOW64\windows.storage.dll
  C:\WINDOWS\SysWOW64\WINTRUST.dll
  C:\WINDOWS\SysWOW64\WLDAP32.dll
  C:\WINDOWS\SysWOW64\wow64.dll
  C:\WINDOWS\SysWOW64\wow64cpu.dll
  C:\WINDOWS\SysWOW64\wow64win.dll
  C:\WINDOWS\SysWOW64\WS2_32.dll
```

```
PS > .\Dependencies.exe -knowndll -json
{
  "x64": [
    "advapi32.dll",
    "bcryptPrimitives.dll",
    "cfgmgr32.dll",
    "clbcatq.dll",
    "combase.dll",
    "COMCTL32.dll",
    "COMDLG32.dll",
    "coml2.dll",
    "CRYPT32.dll",
    "difxapi.dll",
    "FLTLIB.DLL",
    "gdi32.dll",
    "gdi32full.dll",
    "gdiplus.dll",
    "IMAGEHLP.dll",
    "IMM32.dll",
    "kernel.appcore.dll",
    "kernel32.dll",
    "KERNELBASE.dll",
    "MSASN1.dll",
    "MSCTF.dll",
    "msvcp_win.dll",
    "MSVCRT.dll",
    "NORMALIZ.dll",
    "NSI.dll",
    "ntdll.dll",
    "ole32.dll",
    "OLEAUT32.dll",
    "powrprof.dll",
    "profapi.dll",
    "PSAPI.DLL",
    "rpcrt4.dll",
    "sechost.dll",
    "Setupapi.dll",
    "SHCORE.dll",
    "SHELL32.dll",
    "SHLWAPI.dll",
    "ucrtbase.dll",
    "user32.dll",
    "win32u.dll",
    "windows.storage.dll",
    "WINTRUST.dll",
    "WLDAP32.dll",
    "wow64.dll",
    "wow64cpu.dll",
    "wow64win.dll",
    "WS2_32.dll"
  ],
  "x86": [
    "advapi32.dll",
    "bcryptPrimitives.dll",
    "cfgmgr32.dll",
    "clbcatq.dll",
    "combase.dll",
    "COMCTL32.dll",
    "COMDLG32.dll",
    "coml2.dll",
    "CRYPT32.dll",
    "difxapi.dll",
    "FLTLIB.DLL",
    "gdi32.dll",
    "gdi32full.dll",
    "gdiplus.dll",
    "IMAGEHLP.dll",
    "IMM32.dll",
    "kernel.appcore.dll",
    "kernel32.dll",
    "KERNELBASE.dll",
    "MSASN1.dll",
    "MSCTF.dll",
    "msvcp_win.dll",
    "MSVCRT.dll",
    "NORMALIZ.dll",
    "NSI.dll",
    "ntdll.dll",
    "ole32.dll",
    "OLEAUT32.dll",
    "powrprof.dll",
    "profapi.dll",
    "PSAPI.DLL",
    "rpcrt4.dll",
    "sechost.dll",
    "Setupapi.dll",
    "SHCORE.dll",
    "SHELL32.dll",
    "SHLWAPI.dll",
    "ucrtbase.dll",
    "user32.dll",
    "win32u.dll",
    "windows.storage.dll",
    "WINTRUST.dll",
    "WLDAP32.dll",
    "wow64.dll",
    "wow64cpu.dll",
    "wow64win.dll",
    "WS2_32.dll"
  ]
}
```

### Imports

```
PS > .\Dependencies.exe -imports C:\Windows\System32\kernel32.dll
[-] Import listing for file : C:\Windows\System32\kernel32.dll
Import from module api-ms-win-core-rtlsupport-l1-1-0.dll :
         Function RtlVirtualUnwind
         Function RtlUnwindEx
         Function RtlRestoreContext
         Function RtlLookupFunctionEntry
         Function RtlInstallFunctionTableCallback
         Function RtlRaiseException
         Function RtlDeleteFunctionTable
         Function RtlCaptureContext
         Function RtlAddFunctionTable
         Function RtlPcToFileHeader
         Function RtlUnwind
         Function RtlCaptureStackBackTrace
         Function RtlCompareMemory
Import from module ntdll.dll :
         Function RtlSizeHeap
         Function RtlLCIDToCultureName
         Function RtlUnicodeStringToInteger
         Function _wcslwr
         Function RtlGetUILanguageInfo
         Function EtwEventEnabled
         Function RtlpConvertLCIDsToCultureNames
         Function NtEnumerateKey
         Function RtlIntegerToUnicodeString
         Function RtlTimeToTimeFields
         Function RtlTimeFieldsToTime
         Function RtlUnhandledExceptionFilter
         Function NtTerminateProcess
         Function wcsncmp
         Function wcsncpy
         Function LdrFindResourceEx_U
         Function RtlReadThreadProfilingData
         Function RtlQueryThreadProfiling
         Function RtlDisableThreadProfiling
         Function RtlNtStatusToDosErrorNoTeb
[...]
Import from module api-ms-win-core-psapi-ansi-l1-1-0.dll :
         Function K32GetModuleFileNameExA
         Function K32GetDeviceDriverBaseNameA
         Function K32GetMappedFileNameA
         Function K32GetDeviceDriverFileNameA
         Function K32EnumPageFilesA
         Function K32GetProcessImageFileNameA
         Function QueryFullProcessImageNameA
         Function K32GetModuleBaseNameA
Import from module api-ms-win-eventing-provider-l1-1-0.dll :
         Function EventRegister
         Function EventUnregister
         Function EventWriteTransfer
         Function EventSetInformation
Import from module api-ms-win-core-appcompat-l1-1-1.dll :
         Function BaseFreeAppCompatDataForProcess
         Function BaseReadAppCompatDataForProcess
[-] Import listing done
```

```
PS > .\Dependencies.exe -imports -json C:\Windows\System32\kernel32.dll
{
  "Imports": [
    {
      "Flags": 0,
      "Name": "api-ms-win-core-rtlsupport-l1-1-0.dll",
      "NumberOfEntries": 13,
      "ImportList": [
        {
          "Hint": 15,
          "Ordinal": 15,
          "Name": "RtlVirtualUnwind",
          "ModuleName": "api-ms-win-core-rtlsupport-l1-1-0.dll",
          "ImportByOrdinal": false,
          "DelayImport": false
        },
        {
          "Hint": 14,
          "Ordinal": 14,
          "Name": "RtlUnwindEx",
          "ModuleName": "api-ms-win-core-rtlsupport-l1-1-0.dll",
          "ImportByOrdinal": false,
          "DelayImport": false
        },
        {
          "Hint": 12,
          "Ordinal": 12,
          "Name": "RtlRestoreContext",
          "ModuleName": "api-ms-win-core-rtlsupport-l1-1-0.dll",
          "ImportByOrdinal": false,
          "DelayImport": false
        },
        {
          "Hint": 9,
          "Ordinal": 9,
          "Name": "RtlLookupFunctionEntry",
          "ModuleName": "api-ms-win-core-rtlsupport-l1-1-0.dll",
          "ImportByOrdinal": false,
          "DelayImport": false
        },
        {
          "Hint": 8,
          "Ordinal": 8,
          "Name": "RtlInstallFunctionTableCallback",
          "ModuleName": "api-ms-win-core-rtlsupport-l1-1-0.dll",
          "ImportByOrdinal": false,
          "DelayImport": false
        },
[...]
    },
    {
      "Flags": 0,
      "Name": "api-ms-win-core-appcompat-l1-1-1.dll",
      "NumberOfEntries": 2,
      "ImportList": [
        {
          "Hint": 5,
          "Ordinal": 5,
          "Name": "BaseFreeAppCompatDataForProcess",
          "ModuleName": "api-ms-win-core-appcompat-l1-1-1.dll",
          "ImportByOrdinal": false,
          "DelayImport": false
        },
        {
          "Hint": 8,
          "Ordinal": 8,
          "Name": "BaseReadAppCompatDataForProcess",
          "ModuleName": "api-ms-win-core-appcompat-l1-1-1.dll",
          "ImportByOrdinal": false,
          "DelayImport": false
        }
      ]
    }
  ]
}
```

### Exports

```
PS > .\Dependencies.exe -exports C:\Windows\System32\kernel32.dll
[-] Export listing for file : C:\Windows\System32\kernel32.dll
Export 1 :
         Name : AcquireSRWLockExclusive
         VA : 0x0
         ForwardedName : NTDLL.RtlAcquireSRWLockExclusive
Export 2 :
         Name : AcquireSRWLockShared
         VA : 0x0
         ForwardedName : NTDLL.RtlAcquireSRWLockShared
Export 3 :
         Name : ActivateActCtx
         VA : 0x1D680
Export 4 :
         Name : ActivateActCtxWorker
         VA : 0x183D0
Export 5 :
         Name : AddAtomA
         VA : 0x201C0
Export 6 :
         Name : AddAtomW
         VA : 0x109E0
Export 7 :
         Name : AddConsoleAliasA
         VA : 0x213F0
[...]         
Export 1616 :
         Name : uaw_lstrlenW
         VA : 0x31E30
Export 1617 :
         Name : uaw_wcschr
         VA : 0x31E80
Export 1618 :
         Name : uaw_wcscpy
         VA : 0x31EA0
Export 1619 :
         Name : uaw_wcsicmp
         VA : 0x31ED0
Export 1620 :
         Name : uaw_wcslen
         VA : 0x31EE0
Export 1621 :
         Name : uaw_wcsrchr
         VA : 0x31F00
[-] Export listing done
```

```
PS > .\Dependencies.exe -exports -json C:\Windows\System32\kernel32.dll
{
  "Exports": [
    {
      "Ordinal": 1,
      "Name": "AcquireSRWLockExclusive",
      "ExportByOrdinal": false,
      "VirtualAddress": 0,
      "ForwardedName": "NTDLL.RtlAcquireSRWLockExclusive"
    },
    {
      "Ordinal": 2,
      "Name": "AcquireSRWLockShared",
      "ExportByOrdinal": false,
      "VirtualAddress": 0,
      "ForwardedName": "NTDLL.RtlAcquireSRWLockShared"
    },
    {
      "Ordinal": 3,
      "Name": "ActivateActCtx",
      "ExportByOrdinal": false,
      "VirtualAddress": 120448,
      "ForwardedName": ""
    },
    {
      "Ordinal": 4,
      "Name": "ActivateActCtxWorker",
      "ExportByOrdinal": false,
      "VirtualAddress": 99280,
      "ForwardedName": ""
    },
[...]
    {
      "Ordinal": 1619,
      "Name": "uaw_wcsicmp",
      "ExportByOrdinal": false,
      "VirtualAddress": 204496,
      "ForwardedName": ""
    },
    {
      "Ordinal": 1620,
      "Name": "uaw_wcslen",
      "ExportByOrdinal": false,
      "VirtualAddress": 204512,
      "ForwardedName": ""
    },
    {
      "Ordinal": 1621,
      "Name": "uaw_wcsrchr",
      "ExportByOrdinal": false,
      "VirtualAddress": 204544,
      "ForwardedName": ""
    }
  ]
}
```    

### Manifest

```
PS > .\Dependencies.exe -manifest C:\Windows\System32\wuauclt.exe
[-] Manifest for file : C:\Windows\System32\wuauclt.exe

<!-- Copyright (c) Microsoft Corporation -->
<assembly xmlns="urn:schemas-microsoft-com:asm.v1" manifestVersion="1.0">
 <assemblyIdentity version="6.0.0.0" processorArchitecture="amd64" name="Microsoft.Windows.windowsupdate.wuauclt" type="win32" />
 <application xmlns="urn:schemas-microsoft-com:asm.v3">
   <windowsSettings>
     <dpiAware xmlns="http://schemas.microsoft.com/SMI/2005/WindowsSettings">true</dpiAware>
   </windowsSettings>
 </application>
 <dependency>
        <dependentAssembly>
                <assemblyIdentity type="win32" name="Microsoft.Windows.Common-Controls" version="6.0.0.0" processorArchitecture="amd64" publicKeyToken="6595b64144ccf1df" language="*" />
        </dependentAssembly>
 </dependency>
 <trustInfo xmlns="urn:schemas-microsoft-com:asm.v3">
   <security>
    <requestedPrivileges>
     <requestedExecutionLevel level="asInvoker" uiAccess="false" />
     </requestedPrivileges>
    </security>
 </trustInfo>
</assembly>
```

```
PS > .\Dependencies.exe -json -manifest C:\Windows\System32\wuauclt.exe
{
  "Manifest": "<?xml version=\"1.0\" encoding=\"UTF-8\" standalone=\"yes\"?> \r\n<!-- Copyright (c) Microsoft Corporation -->\r\n<assembly xmlns=\"urn:schemas-microsoft-com:asm.v1\" manifestVersion=\"1.0\"> \r\n <assemblyIdentity \r\n\tversion=\"6.0.0.0\"\r\n\tprocessorArchitecture=\"amd64\"\r\n\tname=\"Microsoft.Windows.windowsupdate.wuauclt\"\r\n\ttype=\"win32\"/>\r\n <application  xmlns=\"urn:schemas-microsoft-com:asm.v3\">\r\n   <windowsSettings>\r\n     <dpiAware  xmlns=\"http://schemas.microsoft.com/SMI/2005/WindowsSettings\">true</dpiAware>\r\n   </windowsSettings>\r\n </application>\r\n <dependency>\r\n\t<dependentAssembly>\r\n\t\t<assemblyIdentity\r\n\t\t\ttype=\"win32\"\r\n\t\t\tname=\"Microsoft.Windows.Common-Controls\"\r\n\t\t\tversion=\"6.0.0.0\"\r\n\t\t\tprocessorArchitecture=\"amd64\"\r\n\t\t\tpublicKeyToken=\"6595b64144ccf1df\"\r\n\t\t\tlanguage=\"*\"/>\r\n\t</dependentAssembly>\r\n </dependency>\r\n <trustInfo xmlns=\"urn:schemas-microsoft-com:asm.v3\">\r\n   <security>\r\n    <requestedPrivileges>\r\n     <requestedExecutionLevel\r\n\t level=\"asInvoker\"\r\n\t uiAccess=\"false\" \r\n\t/>\r\n     </requestedPrivileges>\r\n    </security>\r\n </trustInfo>\r\n</assembly>",
  "XmlManifest": {
    "?xml": {
      "@version": "1.0",
      "@encoding": "UTF-8",
      "@standalone": "yes"
    },
    "#text": [
      " \r\n",
      "\r\n"
    ]/* Copyright (c) Microsoft Corporation */,
    "assembly": {
      "@xmlns": "urn:schemas-microsoft-com:asm.v1",
      "@manifestVersion": "1.0",
      "#text": [
        " \r\n ",
        "\r\n ",
        "\r\n ",
        "\r\n ",
        "\r\n"
      ],
      "assemblyIdentity": {
        "@version": "6.0.0.0",
        "@processorArchitecture": "amd64",
        "@name": "Microsoft.Windows.windowsupdate.wuauclt",
        "@type": "win32"
      },
      "application": {
        "@xmlns": "urn:schemas-microsoft-com:asm.v3",
        "#text": [
          "\r\n   ",
          "\r\n "
        ],
        "windowsSettings": {
          "#text": [
            "\r\n     ",
            "\r\n   "
          ],
          "dpiAware": {
            "@xmlns": "http://schemas.microsoft.com/SMI/2005/WindowsSettings",
            "#text": "true"
          }
        }
      },
      "dependency": {
        "#text": [
          "\r\n\t",
          "\r\n "
        ],
        "dependentAssembly": {
          "#text": [
            "\r\n\t\t",
            "\r\n\t"
          ],
          "assemblyIdentity": {
            "@type": "win32",
            "@name": "Microsoft.Windows.Common-Controls",
            "@version": "6.0.0.0",
            "@processorArchitecture": "amd64",
            "@publicKeyToken": "6595b64144ccf1df",
            "@language": "*"
          }
        }
      },
      "trustInfo": {
        "@xmlns": "urn:schemas-microsoft-com:asm.v3",
        "#text": [
          "\r\n   ",
          "\r\n "
        ],
        "security": {
          "#text": [
            "\r\n    ",
            "\r\n    "
          ],
          "requestedPrivileges": {
            "#text": [
              "\r\n     ",
              "\r\n     "
            ],
            "requestedExecutionLevel": {
              "@level": "asInvoker",
              "@uiAccess": "false"
            }
          }
        }
      }
    }
  }
}
```

### SxsEntries

```
PS > .\Dependencies.exe -sxsentries C:\Windows\System32\wuauclt.exe
[-] sxs dependencies for executable : C:\Windows\System32\wuauclt.exe
  [+] comctl32.dll : C:\WINDOWS\WinSxs\amd64_microsoft.windows.common-controls_6595b64144ccf1df_6.0.17134.1_none_e4da93291059d8fb\comctl32.dll
  [x] Microsoft.Windows.Common-Controls.Resources : file ???
```

```
PS > .\Dependencies.exe -sxsentries C:\Windows\System32\wuauclt.exe -json
{
  "SxS": [
    {
      "Name": "comctl32.dll",
      "Path": "C:\\WINDOWS\\WinSxs\\amd64_microsoft.windows.common-controls_6595b64144ccf1df_6.0.17134.1_none_e4da93291059d8fb\\comctl32.dll",
      "Version": "6.0.17134.1",
      "Type": "win32",
      "PublicKeyToken": "6595b64144ccf1df"
    },
    {
      "Name": "Microsoft.Windows.Common-Controls.Resources",
      "Path": "file ???",
      "Version": "",
      "Type": "",
      "PublicKeyToken": ""
    }
  ]
}
```

### Modules

```
PS > .\Dependencies.exe -modules C:\Windows\System32\wuauclt.exe
[ROOT] wuauclt.exe : C:\Windows\System32\wuauclt.exe
[SxS] COMCTL32.dll : C:\WINDOWS\WinSxs\amd64_microsoft.windows.common-controls_6595b64144ccf1df_6.0.17134.1_none_e4da93291059d8fb\comctl32.dll
[ApiSetSchema] api-ms-win-core-com-l1-1-0.dll : C:\WINDOWS\system32\combase.dll
[ApiSetSchema] api-ms-win-core-errorhandling-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-libraryloader-l1-2-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-processenvironment-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-synch-l1-2-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-processthreads-l1-1-0.dll : C:\WINDOWS\system32\kernel32.dll
[ApiSetSchema] api-ms-win-core-profile-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-sysinfo-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-rtlsupport-l1-1-0.dll : C:\WINDOWS\system32\ntdll.dll
[ApiSetSchema] api-ms-win-eventing-provider-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-debug-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-string-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-heap-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-console-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-console-l1-2-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-datetime-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-fibers-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-file-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-handle-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-localization-l1-2-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-memory-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-namedpipe-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-synch-l1-1-0.dll : C:\WINDOWS\system32\KERNELBASE.dll
[ApiSetSchema] api-ms-win-core-util-l1-1-0.dll : C:\WINDOWS\system32\kernel32.dll
[ApiSetSchema] api-ms-win-core-apiquery-l1-1-0.dll : C:\WINDOWS\system32\ntdll.dll
[ApiSetSchema] ext-ms-win-advapi32-registry-l1-1-0.dll : C:\WINDOWS\system32\advapi32.dll
[ApiSetSchema] ext-ms-win-advapi32-registry-l1-1-1.dll : C:\WINDOWS\system32\advapi32.dll
[ApiSetSchema] ext-ms-win-kernel32-appcompat-l1-1-0.dll : C:\WINDOWS\system32\kernel32.dll
[ApiSetSchema] ext-ms-win-ntuser-string-l1-1-0.dll : C:\WINDOWS\system32\user32.dll
[ApiSetSchema] ext-ms-win-kernel32-file-l1-1-0.dll : C:\WINDOWS\system32\kernel32.dll
[ApiSetSchema] ext-ms-win-kernel32-datetime-l1-1-0.dll : C:\WINDOWS\system32\kernel32.dll
[ApiSetSchema] ext-ms-win-kernel32-quirks-l1-1-0.dll : C:\WINDOWS\system32\kernel32.dll
[ApiSetSchema] ext-ms-win-kernel32-quirks-l1-1-1.dll : C:\WINDOWS\system32\kernel32.dll
[ApiSetSchema] ext-ms-win-kernel32-sidebyside-l1-1-0.dll : C:\WINDOWS\system32\kernel32.dll
[ApiSetSchema] ext-ms-win-mrmcorer-resmanager-l1-1-0.dll : C:\Windows\System32\mrmcorer.dll
[...]
[ApplicationDirectory] rtutils.dll : C:\Windows\System32\rtutils.dll
[ApplicationDirectory] MPRMSG.dll : C:\Windows\System32\MPRMSG.dll
[ApplicationDirectory] MPRAPI.dll : C:\Windows\System32\MPRAPI.dll
[ApplicationDirectory] TAPI32.dll : C:\Windows\System32\TAPI32.dll
[ApplicationDirectory] NDFAPI.DLL : C:\Windows\System32\NDFAPI.DLL
[ApplicationDirectory] wdi.dll : C:\Windows\System32\wdi.dll
[ApplicationDirectory] fwpuclnt.dll : C:\Windows\System32\fwpuclnt.dll
[ApplicationDirectory] ktmw32.dll : C:\Windows\System32\ktmw32.dll
[NOT_FOUND] ext-ms-win-wer-xbox-l1-1-0.dll :
[NOT_FOUND] ext-ms-win-audiocore-pal-l1-2-0.dll :
[NOT_FOUND] ext-ms-onecore-appmodel-emclient-l1-1-0.dll :
[NOT_FOUND] ext-ms-onecore-shellchromeapi-l1-1-1.dll :
```

### Chain


```
PS > .\Dependencies.exe -chain C:\Windows\System32\wuauclt.exe
├ wuauclt.exe (ROOT) : C:\Windows\System32\wuauclt.exe 
|  ├ msvcrt.dll (WellKnownDlls) : C:\WINDOWS\system32\MSVCRT.dll 
|  |  ├ ntdll.dll (WellKnownDlls) : C:\WINDOWS\system32\ntdll.dll 
|  |  ├ api-ms-win-core-console-l1-1-0.dll (ApiSetSchema) : C:\WINDOWS\system32\KERNELBASE.dll 
|  ├ api-ms-win-eventing-provider-l1-1-0.dll (ApiSetSchema) : C:\WINDOWS\system32\KERNELBASE.dll 
|  |  |  ├ api-ms-win-core-apiquery-l1-1-0.dll (ApiSetSchema) : C:\WINDOWS\system32\ntdll.dll 
|  |  |  ├ ext-ms-win-advapi32-registry-l1-1-0.dll (ApiSetSchema) : C:\WINDOWS\system32\advapi32.dll 
|  |  |  |  ├ api-ms-win-eventing-controller-l1-1-0.dll (ApiSetSchema) : C:\WINDOWS\system32\sechost.dll 
|  ├ api-ms-win-core-libraryloader-l1-2-0.dll (ApiSetSchema) : C:\WINDOWS\system32\KERNELBASE.dll 
|  |  |  ├ ext-ms-win-advapi32-registry-l1-1-1.dll (ApiSetSchema) : C:\WINDOWS\system32\advapi32.dll 
|  |  |  |  ├ api-ms-win-eventing-consumer-l1-1-0.dll (ApiSetSchema) : C:\WINDOWS\system32\sechost.dll 
|  |  |  |  ├ RPCRT4.dll (WellKnownDlls) : C:\WINDOWS\system32\rpcrt4.dll 
|  ├ api-ms-win-core-errorhandling-l1-1-0.dll (ApiSetSchema) : C:\WINDOWS\system32\KERNELBASE.dll 
|  |  |  ├ ext-ms-win-kernel32-appcompat-l1-1-0.dll (ApiSetSchema) : C:\WINDOWS\system32\kernel32.dll 
[...]
|  |  |  |  |  |  ├ WINNSI.DLL (ApplicationDirectory) : C:\Windows\System32\WINNSI.DLL 
|  |  |  |  ├ ext-ms-win-appmodel-daxcore-l1-1-2.dll (ApiSetSchema) : C:\Windows\System32\daxexec.dll 
|  |  |  |  ├ ext-ms-win-winrt-storage-l1-2-1.dll (ApiSetSchema) : C:\WINDOWS\system32\windows.storage.dll 
|  |  |  |  ├ api-ms-win-devices-query-l1-1-1.dll (ApiSetSchema) : C:\WINDOWS\system32\cfgmgr32.dll 
|  |  |  |  ├ deviceassociation.dll (ApplicationDirectory) : C:\Windows\System32\deviceassociation.dll 
|  |  |  |  |  ├ ext-ms-win-rpc-ssl-l1-1-0.dll (ApiSetSchema) : C:\Windows\System32\rpcrtremote.dll 
|  |  |  |  |  |  ├ fwpuclnt.dll (ApplicationDirectory) : C:\Windows\System32\fwpuclnt.dll 
|  |  |  |  |  |  |  ├ ktmw32.dll (ApplicationDirectory) : C:\Windows\System32\ktmw32.dll 
|  |  |  |  |  ├ ext-ms-win-authz-context-l1-1-0.dll (ApiSetSchema) : C:\Windows\System32\authz.dll 
```