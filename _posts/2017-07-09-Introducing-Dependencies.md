---
layout: post
title: "Introducing Depedencies"
date: 2017-07-09
---

Recently I wanted to study the way `C#` can interop with native code, notably via the use of enlightened C++ dlls ([C++/CLI](https://blogs.msdn.microsoft.com/texblog/2007/04/05/linking-native-c-into-c-applications/) as Microsoft calls it).

The best way to learn about a subject is by making stuff, so I decided to write a C# application embedding native code. Since I don't know much about `C#`, I choosed to reimplement an existing software : the venerable [`Depedency Walker`](http://www.dependencywalker.com/) which is used by pretty much all the Windows devs over 40 y.o. I.ve known to troubleshoot their dll load dependency issues.

I end up writing a full fledged GUI application that partially mimics the features of `depends.exe` and I learn a lot about `WPF` and the dying art of desktop GUI development in the process.

![dependencies-banner-tablet-unblurred](/assets/dependencies-banner-tablet-unblurred.png)

You can get it here : [https://lucasg.github.io/Dependencies/](https://lucasg.github.io/Dependencies/).

<!--more-->

<br>


`depends.exe` is what I would call a "get-shit-done-ware", i.e. a self-contained software that works reliably and add a real value to the user. However, `depends.exe` is not actively maintained anymore and for example does not correctly process the `api-min-win-*-.dll`. On the flip side, it's a marvel that `depends.exe` still pretty much works as is 10 years after being brain dead, development wise. I guess it's another example of the hardcore retro-compatibility mindset in Windows dev teams.

Anyway, I thought it was a good idea to revamp it a bit and learn probably a thing or two about dll loading on Windows.

## PE parsing

When building a dll dependency walker, the first brick to get is a `PE` parser since we need to extract `IMAGE_IMPORT_DESCRIPTOR` data structures from the target executable. 

A good developer can reliably extract thoses features using Windows APIs, but a better developer would reuse an open-source library that already does it. [Unfortunately, there is not a rock-solid open source `PE` parser library in C](https://lucasg.github.io/2017/04/28/the-sad-state-of-pe-parsing) which says a lot about the `RE` community as a whole. Nevertheless, I end up using `ProcessHacker` `phlib` library which has nice wrappers for `PE` parsing and which is pretty much standalone : it only relies on `ntdll.lib` ! Whenever I lack a feature in `phlib`, I usually write it myself and upstream it if possible.


## Embedding native code in a CLR package

Writing a CLR package over a native library is done automagically via a Visual Studio project `Visual C++\CLR\Class library` ([more info on the MSDN](https://msdn.microsoft.com/en-us/library/z6ad605x.aspx)). There are some caveats when building such libraries - and which would merit a standalone blog post in the future - but with enough `ref` tags and `gcnew` calls sprinkled over here and there, everything falls into place.

`ClrPhLib` is the CLR dll which glue the native `phlib` code with the CLR environment. I haven't seen a previous public work on this type of wrapping (apart from the examples on the MSDN) so I had to "improvise". I end up separating the classes from the "managed world" (which are garbage collected) and the "unmanaged world" (whose alloc/free are up to the library writer).

Additionally there is a console application bundled with the GUI, `ClrPhTester` which was developped to "unit" testing the CLR glue library (called `ClrPhLib`). If I get time and motivation, I might rewrite it into a `dumpbin` copycat.

## Writing a GUI in WPF/C\#

Now that we have an import processing library and a C# wrapper on top of it, the last step is to write a GUI that can present the results nicely. I have several years of C++ MFC behind me (yes I'm old), but I haven't wrote a single GUI application in a "modern" framework so I took it as a invitation to refresh my desktop GUI developer skills.

From what I've understood, this is the historical tech stack for building desktop GUI softwares on Windows :

* [[1922-End Of Universe] MFC ](https://en.wikipedia.org/wiki/Microsoft_Foundation_Class_Library), the undying GUI toolkit for win32 applications
* [[199x-2012] WTL](https://en.wikipedia.org/wiki/Windows_Template_Library), an ATL-style addon for MFC which was never really officially supported.
* [[2003-2014] Windows Forms ](https://en.wikipedia.org/wiki/Windows_Forms), a widget toolkit written in managed code and included in the .NET framework. It was destined to replace MFC (which it never did).
* [[2006-????] WPF ](https://en.wikipedia.org/wiki/Windows_Presentation_Foundation), another managed GUI toolkit that sunsetted WinForms and proposed a programming model based on XAML declarations for GUI widgets. It was destined to replace MFC (which it will never do).
* [[2012-2013] WinRT ](https://fr.wikipedia.org/wiki/Windows_Runtime) was the "Metro" style GUI toolkit developped for the Windows Phone and also shipped with Windows 8. We all know how *well* it went.
* [[2013-????] UWP ](https://en.wikipedia.org/wiki/Universal_Windows_Platform) was born on the ashes of WinRT and aim to help Windows GUI devs target desktop, mobile, IOT and xbox platforms with the same code. It is also destined to replace MFC (which it will never do) by breaking backward compatibility (UWP applications can not run on Windows version older than Win10) and blocking Win32 applications from being distributed via the Windows Store (what a cosmically bad idea that embargo was).

Apparently there is currently two GUI active toolkits, UWP for building cross-platform and cross-arch applications and WPF for building desktop applications. I've also put away third party GUI frameworks a la Facebook's React Native, Xamarin Forms or even Qt since I went after the Microsoft's all inclusive experience (I like "coherent" tech stacks). So WPF it is.



Windows Presentation Framework is a pretty powerful GUI toolkit that rely on the MVVM (Model-View-ModelView) design pattern to separate GUI logic from "core" business logic. The "View" part is done via a xaml stylesheet that defines widgets and specify their bindings with the data structures underlying. This design pattern nicely encapsulate away the drawing logic (resising elements, invalidating rects, etc.) from the developer and allow him to focus on writing data processing routines, as long as the developer stays within the given model.

However it makes harder for troubleshooting bindings issues, especially CommandBindings contexts since we can't debug the calls easily (everything is async). But if there is something I've learned as a reverse engineer, it's to throw random inputs until it starts to make sense (or luckily stumble into a working solution).

## Parting thoughts

In conclusion, WPF allow me to quickly write a somewhat decent GUI application in the span of several weeks on my free time, which is pretty nice. I don't intend to actively develop `Dependencies` any more than writing some additional features that are already on my roadmap, but I'll try to fix reported bugs here and there.
