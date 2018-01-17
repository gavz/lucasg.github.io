---
layout: post
title: "Renaming a Win32 executable to main.exe launch the Xbox DVR popup"
date: 2017-01-07
---

Recently I've stumble upon the following question on Stack Overflow : <a href ="http://stackoverflow.com/questions/36712801/windows-10-naming-programs-main-exe-cause-them-to-show-pop-up">windows-10-naming-programs-main-exe-cause-them-to-show-pop-up</a>. The user ```Ether Frog``` has noticed that renaming any executable to ```main.exe``` triggers the Xbox Game DVR recorder when launching the executable on Windows 10.


I've done some digging over the weekend and I have found over 2000 special exe names which will trigger the same behavior, not just ```main.exe```.

This post does contain most of my answer on the SO platform, but I also explain my reverse engineering process.

<!--more-->
<br>
<br>


After confirming the quirky behavior on my machine, I spin up ```Process Hacker``` (fantastic software by the way) to get additional informations on the Xbox popup. That's where it gets tricky : the popup is a modal that disappear whenever it lose focus. Which means every time I click somewhere in the ```Process Hacker``` GUI, the XBox process exit.

![I kinda cheated to create this snapshot, can you see how ?](/assets/ProcHackerSnapshot.png)

Anyway, If I can't have any detailed informations about the ```GameLauncher``` process (loaded modules, etc.) I still got the most important one : ```explorer.exe``` is the parent process.

That means I need to break into ```explorer.exe```, not the nicest process to suspend on Windows if you want to multitask. For example, if a breakpoint fires up and suspend ```explorer```, you can still use the ```Alt-Tab``` shortcut (to switch applications) but not the ```Home-Ctrl-Arrow``` (to switch Virtual Desktops). Handle with care.


## Tried and true reversing

Since ```explorer.exe``` does launch the executable and the xbox popup, I figured there must be a callback mechanism involved in the explorer process. To analyze it, I simply put breakpoints in various ```CreateProcess``` imports:

`0:108> x *!CreateProcess*`
{% highlight text %}
00007ff9`6650cee0 KERNEL32!CreateProcessAsUserWStub (<no parameter info>)
00007ff9`6650bec0 KERNEL32!CreateProcessWStub (<no parameter info>)
00007ff9`66529780 KERNEL32!CreateProcessAsUserAStub (<no parameter info>)
00007ff9`6650b970 KERNEL32!CreateProcessAStub (<no parameter info>)
00007ff9`665297f0 KERNEL32!CreateProcessInternalAStub (<no parameter info>)
00007ff9`66529870 KERNEL32!CreateProcessInternalWStub (<no parameter info>)
00007ff7`ff8ef7f8 Explorer!CreateProcessW (<no parameter info>)
00007ff9`3dc3bac8 werconcpl!CreateProcessSafe (<no parameter info>)
00007ff9`63e3fda0 KERNELBASE!CreateProcessExtensions::ErrorContext::LogError (<no parameter info>)
00007ff9`63d89244 KERNELBASE!CreateProcessExtensions::IsDefaultBrowserCreation (<no parameter info>)
00007ff9`63db1380 KERNELBASE!CreateProcessInternalW (<no parameter info>)
00007ff9`63da5210 KERNELBASE!CreateProcessInternalA (<no parameter info>)
00007ff9`63e3fb78 KERNELBASE!CreateProcessExtensions::CreateSharedLocalFolder (<no parameter info>)
00007ff9`63def16c KERNELBASE!CreateProcessExtensions::ReleaseAppXContext (<no parameter info>)
00007ff9`63d892c4 KERNELBASE!CreateProcessExtensions::PreCreationExtension (<no parameter info>)
00007ff9`63df3400 KERNELBASE!CreateProcessAsUserA (<no parameter info>)
00007ff9`63da2ed0 KERNELBASE!CreateProcessA (<no parameter info>)
00007ff9`63d93714 KERNELBASE!CreateProcessExtensions::VerifyParametersAndGetEffectivePackageMoniker (<no parameter info>)
00007ff9`63dee250 KERNELBASE!CreateProcessAsUserW (<no parameter info>)
00007ff9`63dea3e0 KERNELBASE!CreateProcessW (<no parameter info>)
00007ff9`643a693c windows_storage!CreateProcessWithImpersonation (<no parameter info>)
00007ff9`644fe9d4 windows_storage!CreateProcessAttributeList::DeleteProcThreadList (<no parameter info>)
00007ff9`644f14d8 windows_storage!CreateProcessAttributeList::UpdateAttribute (<no parameter info>)
00007ff9`644f14ac windows_storage!CreateProcessAttributeList::~CreateProcessAttributeList (<no parameter info>)
00007ff9`645aae5c windows_storage!CreateProcessAttributeList::AttributeNameAndValue::~AttributeNameAndValue (<no parameter info>)
00007ff9`64dc59f0 SHELL32!CreateProcessWithImpersonation (<no parameter info>)
00007ff9`666282b0 advapi32!CreateProcessAsUserWStub (<no parameter info>)
00007ff9`6663df20 advapi32!CreateProcessAsUserAStub (<no parameter info>)
00007ff9`66612790 advapi32!CreateProcessWithTokenW (<no parameter info>)
00007ff9`666127fc advapi32!CreateProcessWithLogonCommonW (<no parameter info>)
00007ff9`66655c10 advapi32!CreateProcessWithLogonW (<no parameter info>)
{% endhighlight text %}
`0:108> bp explorer!CreateProcessW` <br>
`0:108> bp advapi32!CreateProcessWithLogonW` <br>
`0:108> bp advapi32!CreateProcessWithLogonCommonW` <br>
`0:108> bp advapi32!CreateProcessWithTokenW` <br>
`0:108> bp KERNELBASE!CreateProcessW` <br>
`0:108> bp KERNELBASE!CreateProcessA` <br>
`0:108> g` <br>
{% highlight text %}
Breakpoint 5 hit
KERNELBASE!CreateProcessW:
00007ff9`63dea3e0 4c8bdc          mov     r11,rsp
0:107> r
rax=0000000000000000 rbx=0000000012e8ca00 rcx=0000000012e8de7c
rdx=00000000129ade00 rsi=0000000000000000 rdi=0000000000000000
rip=00007ff963dea3e0 rsp=000000007489e528 rbp=000000007489e690
 r8=0000000000000000  r9=0000000000000000 r10=0000000000000001
r11=000000007489e588 r12=0000000004080414 r13=0000000012e8caf8
r14=0000000012e8de7c r15=0000000000000000
iopl=0         nv up ei pl nz na po nc
cs=0033  ss=002b  ds=002b  es=002b  fs=0053  gs=002b             efl=00000206
KERNELBASE!CreateProcessW:
00007ff9`63dea3e0 4c8bdc          mov     r11,rsp
0:107> d rcx
00000000`12e8de7c  46 00 3a 00 5c 00 44 00-65 00 76 00 5c 00 53 00  x.:.\.x.x.x.\.x.
00000000`12e8de8c  63 00 79 00 6c 00 6c 00-61 00 5c 00 76 00 31 00  x.x.x.x.x.\.x.x.
00000000`12e8de9c  34 00 30 00 5f 00 78 00-36 00 34 00 5c 00 44 00  x.x._.x.x.x.\.x.
00000000`12e8deac  65 00 62 00 75 00 67 00-5c 00 6d 00 61 00 69 00  x.x.x.x.\.m.a.i.
00000000`12e8debc  6e 00 2e 00 65 00 78 00-65 00 00 00 00 00 00 00  n...e.x.e.......
00000000`12e8decc  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000000`12e8dedc  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000000`12e8deec  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
0:107> k
 # Child-SP          RetAddr           Call Site
00 00000000`7489e528 00007ff9`6650bf13 KERNELBASE!CreateProcessW
01 00000000`7489e530 00007ff9`644503bb KERNEL32!CreateProcessWStub+0x53
02 00000000`7489e590 00007ff9`64450097 windows_storage!CInvokeCreateProcessVerb::_CallCreateProcess+0x11f
03 00000000`7489e810 00007ff9`6444fd76 windows_storage!CInvokeCreateProcessVerb::_PrepareAndCallCreateProcess+0x203
04 00000000`7489e8a0 00007ff9`64450f70 windows_storage!CInvokeCreateProcessVerb::_TryCreateProcess+0x36
05 00000000`7489e8d0 00007ff9`644517ae windows_storage!CInvokeCreateProcessVerb::Launch+0x58
06 00000000`7489e910 00007ff9`6445288b windows_storage!CInvokeCreateProcessVerb::Execute+0x3e
07 00000000`7489e940 00007ff9`64452bc4 windows_storage!CBindAndInvokeStaticVerb::_DoCommand+0x123
08 00000000`7489e9a0 00007ff9`64452310 windows_storage!CBindAndInvokeStaticVerb::_TryCreateProcessDdeHandler+0x68
09 00000000`7489ea20 00007ff9`644c3793 windows_storage!CBindAndInvokeStaticVerb::Execute+0x1b0
0a 00000000`7489ed30 00007ff9`644c33f6 windows_storage!CRegDataDrivenCommand::_TryInvokeAssociation+0xaf
0b 00000000`7489eda0 00007ff9`64d2102d windows_storage!CRegDataDrivenCommand::_Invoke+0x102
0c 00000000`7489edf0 00007ff9`64d1fea6 SHELL32!CRegistryVerbsContextMenu::_Execute+0x8d
0d 00000000`7489ee50 00007ff9`64d3a459 SHELL32!CRegistryVerbsContextMenu::InvokeCommand+0xa6
0e 00000000`7489f150 00007ff9`64d4f80e SHELL32!HDXA_LetHandlerProcessCommandEx+0x111
0f 00000000`7489f250 00007ff9`64df864c SHELL32!CDefFolderMenu::InvokeCommand+0x13e
10 00000000`7489f5c0 00007ff9`64df8273 SHELL32!SHInvokeCommandOnContextMenu2+0x1a8
11 00000000`7489f800 00007ff9`63b55aad SHELL32!s_DoInvokeVerb+0xc3
12 00000000`7489f870 00007ff9`664f8364 SHCORE!Microsoft::WRL::Details::RuntimeClass<Microsoft::WRL::Details::InterfaceList<CRandomAccessStreamBase,Microsoft::WRL::Details::InterfaceList<Windows::Storage::Streams::IRandomAccessStreamWithContentType,Microsoft::WRL::Details::InterfaceList<Windows::Storage::Streams::IContentTypeProvider,Microsoft::WRL::Details::InterfaceList<Microsoft::WRL::Implements<Microsoft::WRL::RuntimeClassFlags<3>,Microsoft::WRL::CloakedIid<IRandomAccessStreamMode>,Microsoft::WRL::CloakedIid<IRandomAccessStreamFileAccessMode>,Microsoft::WRL::CloakedIid<IObjectWithDeferredInvoke>,Microsoft::WRL::CloakedIid<IObjectWithFileHandle>,Microsoft::WRL::CloakedIid<IUnbufferedFileHandleProvider>,Microsoft::WRL::CloakedIid<IRandomAccessStreamPrivate>,Microsoft::WRL::CloakedIid<ITransactedModeOverride>,Microsoft::WRL::CloakedIid<CFTMCrossProcServer>,Microsoft::WRL::Details::Nil>,Microsoft::WRL::Details::Nil> > > >,Microsoft::WRL::RuntimeClassFlags<3>,1,1,0>::~RuntimeClass<Microsoft::WRL::Details::InterfaceList<CRandomAccessStreamBase,Microsoft::WRL::Details::InterfaceList<Windows::Storage::Streams::IRandomAccessStreamWithContentType,Microsoft::WRL::Details::InterfaceList<Windows::Storage::Streams::IContentTypeProvider,Microsoft::WRL::Details::InterfaceList<Microsoft::WRL::Implements<Microsoft::WRL::RuntimeClassFlags<3>,Microsoft::WRL::CloakedIid<IRandomAccessStreamMode>,Microsoft::WRL::CloakedIid<IRandomAccessStreamFileAccessMode>,Microsoft::WRL::CloakedIid<IObjectWithDeferredInvoke>,Microsoft::WRL::CloakedIid<IObjectWithFileHandle>,Microsoft::WRL::CloakedIid<IUnbufferedFileHandleProvider>,Microsoft::WRL::CloakedIid<IRandomAccessStreamPrivate>,Microsoft::WRL::CloakedIid<ITransactedModeOverride>,Microsoft::WRL::CloakedIid<CFTMCrossProcServer>,Microsoft::WRL::Details::Nil>,Microsoft::WRL::Details::Nil> > > >,Microsoft::WRL::RuntimeClassFlags<3>,1,1,0>+0x135
13 00000000`7489f960 00007ff9`675070d1 KERNEL32!BaseThreadInitThunk+0x14
14 00000000`7489f990 00000000`00000000 ntdll!RtlUserThreadStart+0x21
{% endhighlight text %}
We successfully broke in the `CreateProcess` launching our `main.exe` executable. <br>
`0:107> g`
{% highlight text %}
Breakpoint 5 hit
KERNELBASE!CreateProcessW:
00007ff9`63dea3e0 4c8bdc          mov     r11,rsp
0:106> d rcx
00000000`185064ac  43 00 3a 00 5c 00 57 00-69 00 6e 00 64 00 6f 00  C.:.\.W.i.n.d.o.
00000000`185064bc  77 00 73 00 5c 00 53 00-79 00 73 00 74 00 65 00  w.s.\.S.y.s.t.e.
00000000`185064cc  6d 00 33 00 32 00 5c 00-47 00 61 00 6d 00 65 00  m.3.2.\.G.a.m.e.
00000000`185064dc  50 00 61 00 6e 00 65 00-6c 00 2e 00 65 00 78 00  P.a.n.e.l...e.x.
00000000`185064ec  65 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  e...............
00000000`185064fc  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000000`1850650c  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
00000000`1850651c  00 00 00 00 00 00 00 00-00 00 00 00 00 00 00 00  ................
0:106> k
 # Child-SP          RetAddr           Call Site
00 00000000`7499e848 00007ff9`6650bf13 KERNELBASE!CreateProcessW
01 00000000`7499e850 00007ff9`644503bb KERNEL32!CreateProcessWStub+0x53
02 00000000`7499e8b0 00007ff9`64450097 windows_storage!CInvokeCreateProcessVerb::_CallCreateProcess+0x11f
03 00000000`7499eb30 00007ff9`6444fd76 windows_storage!CInvokeCreateProcessVerb::_PrepareAndCallCreateProcess+0x203
04 00000000`7499ebc0 00007ff9`64450f70 windows_storage!CInvokeCreateProcessVerb::_TryCreateProcess+0x36
05 00000000`7499ebf0 00007ff9`644517ae windows_storage!CInvokeCreateProcessVerb::Launch+0x58
06 00000000`7499ec30 00007ff9`6445288b windows_storage!CInvokeCreateProcessVerb::Execute+0x3e
07 00000000`7499ec60 00007ff9`64452bc4 windows_storage!CBindAndInvokeStaticVerb::_DoCommand+0x123
08 00000000`7499ecc0 00007ff9`64452310 windows_storage!CBindAndInvokeStaticVerb::_TryCreateProcessDdeHandler+0x68
09 00000000`7499ed40 00007ff9`644c3793 windows_storage!CBindAndInvokeStaticVerb::Execute+0x1b0
0a 00000000`7499f050 00007ff9`644c33f6 windows_storage!CRegDataDrivenCommand::_TryInvokeAssociation+0xaf
0b 00000000`7499f0c0 00007ff9`64d2102d windows_storage!CRegDataDrivenCommand::_Invoke+0x102
0c 00000000`7499f110 00007ff9`64d1fea6 SHELL32!CRegistryVerbsContextMenu::_Execute+0x8d
0d 00000000`7499f170 00007ff9`64d3a459 SHELL32!CRegistryVerbsContextMenu::InvokeCommand+0xa6
0e 00000000`7499f470 00007ff9`64d4f80e SHELL32!HDXA_LetHandlerProcessCommandEx+0x111
0f 00000000`7499f570 00007ff9`64d1ed53 SHELL32!CDefFolderMenu::InvokeCommand+0x13e
10 00000000`7499f8e0 00007ff9`64d1ec1b SHELL32!CShellExecute::_InvokeInProcExec+0x103
11 00000000`7499f9e0 00007ff9`64d1e537 SHELL32!CShellExecute::_InvokeCtxMenu+0x4b
12 00000000`7499fa20 00007ff9`64d8ad3e SHELL32!CShellExecute::_DoExecute+0x15b
13 00000000`7499fa70 00007ff9`63b55aad SHELL32!ShouldShowHomegroupUsersForStatus+0x32
14 00000000`7499faa0 00007ff9`664f8364 SHCORE!Microsoft::WRL::Details::RuntimeClass<Microsoft::WRL::Details::InterfaceList<CRandomAccessStreamBase,Microsoft::WRL::Details::InterfaceList<Windows::Storage::Streams::IRandomAccessStreamWithContentType,Microsoft::WRL::Details::InterfaceList<Windows::Storage::Streams::IContentTypeProvider,Microsoft::WRL::Details::InterfaceList<Microsoft::WRL::Implements<Microsoft::WRL::RuntimeClassFlags<3>,Microsoft::WRL::CloakedIid<IRandomAccessStreamMode>,Microsoft::WRL::CloakedIid<IRandomAccessStreamFileAccessMode>,Microsoft::WRL::CloakedIid<IObjectWithDeferredInvoke>,Microsoft::WRL::CloakedIid<IObjectWithFileHandle>,Microsoft::WRL::CloakedIid<IUnbufferedFileHandleProvider>,Microsoft::WRL::CloakedIid<IRandomAccessStreamPrivate>,Microsoft::WRL::CloakedIid<ITransactedModeOverride>,Microsoft::WRL::CloakedIid<CFTMCrossProcServer>,Microsoft::WRL::Details::Nil>,Microsoft::WRL::Details::Nil> > > >,Microsoft::WRL::RuntimeClassFlags<3>,1,1,0>::~RuntimeClass<Microsoft::WRL::Details::InterfaceList<CRandomAccessStreamBase,Microsoft::WRL::Details::InterfaceList<Windows::Storage::Streams::IRandomAccessStreamWithContentType,Microsoft::WRL::Details::InterfaceList<Windows::Storage::Streams::IContentTypeProvider,Microsoft::WRL::Details::InterfaceList<Microsoft::WRL::Implements<Microsoft::WRL::RuntimeClassFlags<3>,Microsoft::WRL::CloakedIid<IRandomAccessStreamMode>,Microsoft::WRL::CloakedIid<IRandomAccessStreamFileAccessMode>,Microsoft::WRL::CloakedIid<IObjectWithDeferredInvoke>,Microsoft::WRL::CloakedIid<IObjectWithFileHandle>,Microsoft::WRL::CloakedIid<IUnbufferedFileHandleProvider>,Microsoft::WRL::CloakedIid<IRandomAccessStreamPrivate>,Microsoft::WRL::CloakedIid<ITransactedModeOverride>,Microsoft::WRL::CloakedIid<CFTMCrossProcServer>,Microsoft::WRL::Details::Nil>,Microsoft::WRL::Details::Nil> > > >,Microsoft::WRL::RuntimeClassFlags<3>,1,1,0>+0x135
15 00000000`7499fb90 00007ff9`675070d1 KERNEL32!BaseThreadInitThunk+0x14
16 00000000`7499fbc0 00000000`00000000 ntdll!RtlUserThreadStart+0x21
{% endhighlight text %}

Just using this simple dynamic analysis, I've managed to get the executable responsible for the XBox popup : `C:\Windows\System32\GamePanel.exe`. By looking and comparing the two stack traces, we can see the last common return call is this monstruosity of a class : 

<div class="highlight"><pre><span class="nf">SHCORE!Microsoft::WRL::Details::RuntimeClass&lt;Microsoft::WRL::Details::InterfaceList&lt;CRandomAccessStreamBase,Microsoft::WRL::Details::InterfaceList&lt;Windows::Storage::Streams::IRandomAccessStreamWithContentType,Microsoft::WRL::Details::InterfaceList&lt;Windows::Storage::Streams::IContentTypeProvider,Microsoft::WRL::Details::InterfaceList&lt;Microsoft::WRL::Implements&lt;Microsoft::WRL::RuntimeClassFlags&lt;3&gt;,Microsoft::WRL::CloakedIid&lt;IRandomAccessStreamMode&gt;,Microsoft::WRL::CloakedIid&lt;IRandomAccessStreamFileAccessMode&gt;,Microsoft::WRL::CloakedIid&lt;IObjectWithDeferredInvoke&gt;,Microsoft::WRL::CloakedIid&lt;IObjectWithFileHandle&gt;,Microsoft::WRL::CloakedIid&lt;IUnbufferedFileHandleProvider&gt;,Microsoft::WRL::CloakedIid&lt;IRandomAccessStreamPrivate&gt;,Microsoft::WRL::CloakedIid&lt;ITransactedModeOverride&gt;,Microsoft::WRL::CloakedIid&lt;CFTMCrossProcServer&gt;,Microsoft::WRL::Details::Nil&gt;,Microsoft::WRL::Details::Nil&gt; &gt; &gt; &gt;,Microsoft::WRL::RuntimeClassFlags&lt;3&gt;,1,1,0&gt;::~RuntimeClass&lt;Microsoft::WRL::Details::InterfaceList&lt;CRandomAccessStreamBase,Microsoft::WRL::Details::InterfaceList&lt;Windows::Storage::Streams::IRandomAccessStreamWithContentType,Microsoft::WRL::Details::InterfaceList&lt;Windows::Storage::Streams::IContentTypeProvider,Microsoft::WRL::Details::InterfaceList&lt;Microsoft::WRL::Implements&lt;Microsoft::WRL::RuntimeClassFlags&lt;3&gt;,Microsoft::WRL::CloakedIid&lt;IRandomAccessStreamMode&gt;,Microsoft::WRL::CloakedIid&lt;IRandomAccessStreamFileAccessMode&gt;,Microsoft::WRL::CloakedIid&lt;IObjectWithDeferredInvoke&gt;,Microsoft::WRL::CloakedIid&lt;IObjectWithFileHandle&gt;,Microsoft::WRL::CloakedIid&lt;IUnbufferedFileHandleProvider&gt;,Microsoft::WRL::CloakedIid&lt;IRandomAccessStreamPrivate&gt;,Microsoft::WRL::CloakedIid&lt;ITransactedModeOverride&gt;,Microsoft::WRL::CloakedIid&lt;CFTMCrossProcServer&gt;,Microsoft::WRL::Details::Nil&gt;,Microsoft::WRL::Details::Nil&gt; &gt; &gt; &gt;,Microsoft::WRL::RuntimeClassFlags&lt;3&gt;,1,1,0&gt;+0x135</span></pre></div>

When opening `shell32.dll` in IDA, you can see that the `CreateProcess` call for the `GamePanel.exe` process is located in a Thread procedure (tagged as `pfnThreadProc` by IDA) in the `shell32!CShellExecute::_RunThreadMaybeWait` function. Backtracking a bit, I managed to disclose where the caller is : 

`0:101> bp Shell32+0x9e1e5` <br>
`0:101> g`
{%highlight text %}
ModLoad: 00007ff9`5c730000 00007ff9`5c743000   C:\Windows\System32\smartscreenps.dll
F9624C9962: (caller: 00007FF94D52BEC0) LogHr(76) tid(2ec8) 80070002 Le fichier spécifié est introuvable.
Breakpoint 7 hit
SHELL32!CShellExecute::ExecuteNormal+0x7d:
00007ff9`64d1e1e5 84c0            test    al,al
0:065> r al
al=1
0:065> k
 # Child-SP          RetAddr           Call Site
00 00000000`6e59e970 00007ff9`64cb62e7 SHELL32!CShellExecute::ExecuteNormal+0x7d
01 00000000`6e59e9a0 00007ff9`64cb6245 SHELL32!ShellExecuteNormal+0x53
02 00000000`6e59e9d0 00007ff9`4d528e18 SHELL32!ShellExecuteExW+0x35
03 00000000`6e59ea00 00007ff9`4d52a7a9 TwinUI!BroadcastDVRComponent::LaunchGameOverlayWithStartupTipsFlag+0xf8 [d:\rs1\shell\twinui\broadcastdvrcomponent\lib\broadcastdvrprovider.cpp @ 2235]
04 00000000`6e59eee0 00007ff9`4d4254e9 TwinUI!BroadcastDVRComponent::SendShellNotificationToDVR+0x19d [d:\rs1\shell\twinui\broadcastdvrcomponent\lib\broadcastdvrprovider.cpp @ 2370]
05 00000000`6e59efa0 00007ff9`4d2acf9c TwinUI!BroadcastDVRComponent::OnApplicationViewChangedTask+0x17afa9 [d:\rs1\shell\twinui\broadcastdvrcomponent\lib\broadcastdvrprovider.cpp @ 1819]
06 (Inline Function) --------`-------- TwinUI!BroadcastDVRComponent::OnApplicationViewChanged::__l26::<lambda_28e381728a25b292addb57396d526bfd>::operator()+0x11 [d:\rs1\shell\twinui\broadcastdvrcomponent\lib\broadcastdvrprovider.cpp @ 1965]
07 (Inline Function) --------`-------- TwinUI!Windows::Internal::ComTaskPool::CTaskWrapper<<lambda_28e381728a25b292addb57396d526bfd> >::Run+0x11 [d:\rs1.public.fre\internal\sdk\inc\winrt\comtaskpool.h @ 414]
08 00000000`6e59f3d0 00007ff9`4d3ab8d3 TwinUI!Windows::Internal::ComTaskPool::CThread::_ThreadProc+0x65c [d:\rs1.public.fre\internal\sdk\inc\winrt\comtaskpool.h @ 1201]
09 (Inline Function) --------`-------- TwinUI!Windows::Internal::ComTaskPool::CThread::s_ExecuteThreadProc+0x8 [d:\rs1.public.fre\internal\sdk\inc\winrt\comtaskpool.h @ 994]
0a 00000000`6e59f710 00007ff9`674f2af4 TwinUI!Windows::Internal::ComTaskPool::CThread::s_ThreadPoolCallback+0x23 [d:\rs1.public.fre\internal\sdk\inc\winrt\comtaskpool.h @ 1006]
0b 00000000`6e59f740 00007ff9`674d3272 ntdll!TppSimplepExecuteCallback+0x74
0c 00000000`6e59f780 00007ff9`664f8364 ntdll!TppWorkerThread+0x4b2
0d 00000000`6e59fb80 00007ff9`675070d1 KERNEL32!BaseThreadInitThunk+0x14
0e 00000000`6e59fbb0 00000000`00000000 ntdll!RtlUserThreadStart+0x21
{% endhighlight text %}


`twinui.dll` has an instance called ```BroadcastDVRComponent``` which is fired up whenever an "Application" GUI is updated.  The `TwinUI!BroadcastDVRComponent::LaunchGameOverlayWithStartupTipsFlag` method tells me I'm on the right track. 

Here is the flow graph of `BroadcastDVRComponent::OnApplicationViewChanged`:

![IDA flowgraph of BroadcastDVRComponent::OnApplicationViewChanged](/assets/OnApplicationChanged.PNG)


`BroadcastDVRComponent::EvaluateIfViewIsGame` has several branches to treat differently Universal applications, .NET managed ones or traditionnal Win32 exe. In the latter branch, it end up relying on the `resourcepolicyclient.dll` to retrieve the GameId using the application's PID. 

{% highlight C %}
__int64 __fastcall IsGameManager::CheckGameConfigStoreForIsGameFromProcessId(
  IsGameManager *this,
  unsigned int dwProcessId,
  bool *pfIsGame,
  unsigned int *pdwRevision,
  _GUID *pGameId,
  Windows::Internal::String *pwsPresenceId
)
{
  /* 
  * ...
  *  Lots of stack variables
  * ...
  */

  v6 = this;
  v7 = pdwRevision;
  Result = 0;
  v8 = pfIsGame;
  *pfIsGame = 0;

  // Check some revision
  *pdwRevision = 0;
  if ( pwsPresenceId->_hstring )
  {
    WindowsDeleteString();
    pwsPresenceId->_hstring = 0i64;
  }

  // Get "GameId" using only process's PID 
  v9 = v6->_spGameConfigStore._Myptr;
  v10 = v9->vfptr->GetGameIdByPID;
  ErrorStatus = _guard_dispatch_icall_fptr(v9); // resourcepolicyclient!ExecutionModel::GameConfigStoreClient::GetGameIdByPID
  if ( ErrorStatus >= 0 )
  {
    
    _mm_store_si128((__m128i *)&bGameProp, bIsGame);
    ErrorStatus = IsGameManager::IsEnabledInGameConfigStore(v6, (_GUID *)&bGameProp, v8, (unsigned int *)&Result);

    if ( ErrorStatus >= 0 )
    {
      /* 
       * ...
       * tracer code
       * ...
       */

      _mm_store_si128((__m128i *)&bGameProp, bIsGame);
      v17 = IsGameManager::GetTitleIdFromGameConfigStore(v6, (_GUID *)&bGameProp, v13, pwsPresenceId);
      if ( v17 < 0 )
      {
        /* 
         * ...
         * error tracer code
         * ...
         */
      }

      *v7 = v18;
    }
  }
  return (unsigned int)ErrorStatus;
}
{% endhighlight C %}

`resourcepolicyclient!ExecutionModel::GameConfigStoreClient::GetGameIdByPID : `  
{% highlight C %}
unsigned int __fastcall ExecutionModel::GameConfigStoreClient::GetGameIdByPID(
  ExecutionModel::GameConfigStoreClient *this,
  int a2,
  struct _GUID *a3
)
{
  /* 
  *  Lots of stack variables
  */

  v8 = a3;
  v7 = a2;
  if ( a3 )
  {
    result = DevPlat::Shared::RpcCall<long (*)(void *,unsigned long,_GUID *),void * const &,unsigned long const &,_GUID * &>(
               this,
               (char *)this + 8,
               &v7,
               &v8);
    v4 = result;
    if ( (result & 0x80000000) != 0 )
    {
      /* 
       * ...
       * error tracer code
       * ...
      */
    }
  }
  else
  {
    result = E_INVALIDARG;
  }
  return result;
}
{% endhighlight C %}

`resourcepolicyclient!ExecutionModel::GameConfigStoreClient::GetGameIdByPID` is the fork point between a traditional app and a game since for the former it return the error code `E_PROP_ID_UNSUPPORTED = 0x80070490` whereas returning `E_ERROR_SUCCESS` for the latter. Unfortunately, the `GameId` retrieval is hidden behind a `RPC` call.

RPC procedures are particulary painful to reverse, since you can't know easily if the RPC server is in the same process or in a remote process. Moreover, RPC calls are asynchronous so you can't just trace the whole ```rpccrt.dll``` dynamically until it lands somewhere in the RPC server's side. If there is one thing I don't want to do, it's reversing the NT's kernel ALPC event loop.
Most of the times I end up relying on runtime type information (RTTI) and public symbols.

<a href ="http://www.rpcview.org/"> RPCView </a> can help if it recognize an interface, but in my case it did not. The RPC engine I'm looking for is probably located in-process, which means I'm back to square one where I have to statically reverse the ```explorer.exe``` binary and its slew of loaded modules.

For the people who do some reversing, it's one of those moments where you are tempted to throw the towel. You've tried, and you're wondering if the hurdle you facing is really worth the extra effort you'll need to put into to potentially overcome it. Basically like when you're trying to reverse a needlessly obfuscated binary in a CTF and you're thinking : "I should get paid to do this shit".

## You've got handles ?

Anyway, while fishing for some ```COM``` interfaces in the ```explorer.exe``` process, I've found that ```explorer.exe``` has a handle on the following folder : ```C:\Windows\broadcastdvr```. Since I know the XBox popup is handled by the ```twinui!BroadcastDVRComponent``` class this folder got my attention.

Fortunately, the ```C:\Windows\broadcastdvr``` folder contains only one file : ```KnownGameList.bin```, which looks like a winner to me.

NB : ```explorer.exe``` also has a handle on the ```C:\Users\YOUR_USERNAME\AppData\Local\Microsoft\GamesDVR\KnownGameList.bin``` file. This file must be a copy from the ```C:\Windows\broadcastdvr``` one and updated for each logon user.

```KnownGameList.bin``` does indeed list all the special executables which triggers the XBox recorder popup. You can see the ```main.exe``` entry here (entry #1007) :

![MS, why not checking the exe's manifest instead ?](/assets/MainInKnownGameList.PNG)

I've written a [010 template file](#010-template) to parse the entry list  and it comes with 2089 entries on my computer. From what I've seen by reversing the binary file, there is three types of entry:
	
* the "simple" one where there is only a match on the executable name. <br>
  For example : ```main.exe``` or ```ai.exe```

* the more complex one where there is a match on the executable name and the path where the exe is stored must contains some strings. <br>
  For example : ```acu.exe``` must be located in a subfolder of ```Assassin's Creed Unity```.

* Some entries have additional strings to match, but I haven't found how to trigger the game DVR popup for them.


Back in `twinui` there is a whole class called `CQueryKnownGameList` which happen to open this very file (and some others in order to update the list).  `TwinUI!CQueryKnownGameList::GetGameEntryByIdentifier`  and `TwinUI!CQueryKnownGameList::GetTOCIndexByIdentifier` are called whenever a new executable (there is a lookup cache mechanism) is launched. In the latter function there is a case-insensitive string comparison made by `wcsnicmp` between the process name and the entry name.

NB : the Win32 subsystem is case-insensitive (I think) so it makes sense that the executable name's case does not matter.

## Disabling GameDVR

If that behavior is annoying, it  is always possible to globally disable it by setting the ```ShowStartup``` registry key to 0. It is located in ```HKEY_CURRENT_USER\Software\Microsoft\GameBar```.
I haven't found how to disable specifically an executable from triggering it, but I might be possible just by looking at the machine code in ```twinui```.


## Security matter

We have a situation where we can launch a process just by changing the name of an executable. That might be dangerous.

Fortunately, the game launcher command line is located in ```HKEY_LOCAL_MACHINE\Software\Microsoft\GameOverlay``` which needs admin level to write into, so there is no UAC bypass possible here. The game launcher executable is located in the system dir `C:\Windows`, so here again admin levels are needed for modify it.
Moreover, only the `TrustedInstaller` "user" (as the way Windows defines it) has the authorization to modify the command line value in the registry : 

![You trust TrustedInstaller, do you ?](/assets/GamePanelRegKeyAuth.PNG)

(I did not found an authorative link from the msdn for registry hives auths, so here a SO answer confirming it : <a href="http://stackoverflow.com/questions/53135/what-registry-access-can-you-get-without-administrator-privleges"> what-registry-access-can-you-get-without-administrator-privileges </a> )


##  <a name="010-template"></a> 010 template :

{% highlight c %}
typedef struct  {
   BYTE Reserved[0x300];
}HEADER;

typedef struct  {
    WORD ByteLen;
    BYTE RawString[ByteLen];
    //local string sName=ReadWString(RawString);
} GAME_WSTR <read=ReadGame>;

typedef struct {
    DWORD Reserved;
    DWORD ByteLen;
    BYTE RawString[ByteLen] <fgcolor=cLtRed>;
} OPTION_STR  <read=ReadOption>;

typedef struct  {
   local int StartAddr = FTell();
   DWORD EntrySize;

   // Executable game name
   GAME_WSTR GameName <fgcolor=cLtBlue>;
   
   // Optional magic
   if (ReadUShort() == 0xca54)
        WORD OptReserved;
   
   // Optional structs based on switch values
   WORD AdditionalNamesCount;
   WORD SwitchOption2;
   
   // Additional names (probably like a hint).
   local int i =0;
   for (i = 0; i <  AdditionalNamesCount; i++){
        OPTION_STR Option;
        if (ReadUShort() == 0xca54)
            WORD OptReserved;
   }
   
   // Look for a magic
   local int Find20h = 0;
   while(!Find20h){
        Find20h = (0x20 == ReadByte());
        BYTE Res;
   }
   
   GAME_WSTR GameId;
   WORD Reserved;
   
   // Sometimes there is an additionnal name
   // sometimes not. I check the current entry
   // is less than the EntrySize declared.
   if (FTell()-StartAddr < EntrySize)
   {
       switch (SwitchOption2)
       {
       case 3:
            OPTION_STR Option3;
            break;
       case 2:
            
            OPTION_STR Option2;
       case 1:
            break;
       }
    }

} ENTRY <read=ReadGameName>;

string ReadOption(OPTION_STR &Game)
{
    local wstring GameName = L"";
    local int i ;
    for (i= 0; 2*i < Game.ByteLen; i++){
        WStrcat(GameName, Game.RawString[2*i]);
    }
    return WStringToString(GameName);
}

string ReadGame(GAME_WSTR &Game)
{
    local wstring GameName = L"";
    local int i ;
    for (i= 0; 2*i < Game.ByteLen; i++){
        WStrcat(GameName, Game.RawString[2*i]);
    }
    return WStringToString(GameName);
}

string ReadGameName(ENTRY &Entry)
{
    local string GameName = ReadGame(Entry.GameName);
    local string OptionGameName = "";
    if (Entry.AdditionalNamesCount)
        OptionGameName = " : "+ReadOption(Entry.Option);
        
    return GameName + OptionGameName;
}

//------------------------------------------
LittleEndian();
Printf("Parse KnownGameList.bin Begin.\n");
HEADER UnkwownHeader <bgcolor=cLtGray>;
while(1)
{
    ENTRY Entry <bgcolor=cLtPurple>;
    //Printf("Entry : %s -> %d.\n",ReadGameName(Entry) ,Entry.AdditionalNamesCount);
}
Printf("Parse KnownGameList.bin End.\n");
{% endhighlight c %}
