---
layout: post
title: "Creating an Appx package without losing it's mind"
date: 2018-01-15
---


I hack on my free time on [Dependencies](https://github.com/lucasg/dependencies) and since it's inception I wanted to create an installer for it. Microsoft introduced along with its Windows Store a new archive format `.appx` to bundle applications in a publishable way, so I had a go at it.

Here's every hurdles and pitfalls I had to overcome before managing correctly bundle my application in an `appx` package.

<!--more-->
## Introduction

Application publication and installation is something really important that is usually overlooked by developers ("just do `curl xxx | bash`" or "`./configure;make;make install`" !). This is quite unfortunate since not everyone is willing to recreate your build environment in order to compile your application. That's why it's important to also publish precompiled binaries because some of your users simply just want to use what you've created.

As your project grow by, there's a point where you can't just distribute a set of binaries in a zipped file : you need to have a installer in order to properly set up the application in a remote client. While I've not quite reach that point with [Dependencies](https://lucasg.github.io/Dependencies/), I have hit some pain points : 


* The GUI executable and the underlying engine DLL need to be closely updated since it don't bother with ABI retrocompatibilty.
* Like any C# application, Dependencies target a particular version of .NET framework which [already raise some weird bugs](https://github.com/lucasg/Dependencies/issues/5)
* If I want to cache some data on the disk, an installed application usually has a persistent way to store data on the disk.
* `smartscreen` whine at users upon launching the application since my binaries are not signed and rarely downloaded & executed.

## Installer

There are a few installer frameworks out there (NSIS, WIX, MSI, InstallShield, etc.), and they have all their pros and cons but I won't talk about it since I never attempt to write an installer using those framework. Moreover on an Insider build, a new popup recently appeared when I tried to launch a legit installer :

![Windows Store, thanks for worrying for me, but I know what I am doing.](/assets/WindowsStoreWarning.PNG)

For those who don't read French, this window is reminding the user he's launching an exe that has not been distributed via the Windows Store. While this popup is pretty innocuous and can be disabled easily from the control panel, I still can see the writing on the wall : Microsoft will try to muscle their way into making all third party applications use the Windows Store. That's probably the major reason why they went a great deal to implement Desktop Bridge.

The Windows Store has some pretty great advantages on paper : 

* **App CDN infrastructure** handled by Microsoft. This is surely useful for widely successful applications (which is not my case)
* **OS version targeting** : you can actually specify which minor version (the Store is only available since Windows 10) your application runs. That's particularly advantageous for me since I rely on undocumented structures whose layout can change between OS minor versions.
* **automatic update mechanism** : no more EvilGrade ! Why did we had to wait 30 years before having this ? Windows has a lot of experience with securely update systems (KB, WSUS, etc.) so I trust them to do it correctly.
* **proper uninstaller** : self-deleting code is always fun to write, no ?
* **app discoverability** : one stop shop to download all the application you need. No need to download backdoored exes from possibly trounced websites anymore.


## Desktop Bridge

My application being a frankenstein monster of C# code mixed with C++ (and relying on deep Win32 internals) it can't published as a traditional UWP app, so I need to use Desktop Bridge. 
[Desktop Bridge/Project Centennial](https://blogs.msdn.microsoft.com/appconsult/2016/10/13/desktop-bridge-the-bridge-between-win32-apps-and-the-universal-windows-platform/) is a major effort by Microsoft to convert traditionnal unmanaged Win32 applications into the new UWP paradigm. Well they had to do it since after several years the Windows Store still is a ghost town ...


Creating a Desktop Bridge package is [actually pretty straightforward and documented ]( https://docs.microsoft.com/en-us/windows/uwp/porting/desktop-to-uwp-packaging-dot-net) but it has some prerequisites : 

* **Only .NET 4.6.1 is supported** : I had to retarget all my .NET code, but not a big hassle.
* **No admin level privileges/no access to HKLM registry hive** : no problem for me, but It can be a show stopper for some big applications
* **no service/kernel support** : so no AV on the Windows Store ?
* **no user interactive installer** : so there is no way to bundle an application with a AskToolbar drive-by adware for some sweet sweet money ? Bummer.
* **no COM objects registering** : that can block enterprise softwares, like for example Office.


To create a new appx packaging project, choose the right solution under `Visual C#\Universal Desktop` : 
![](/assets/AppxSolution.PNG)


Once the correct solution has been generated, you can customize it by tweaking the `Package.appxmanifest` either as a xml file or via the built-in editor in Visual Studio : 


| ![](/assets/PackageAppxManifest.PNG) | ![](/assets/PackageAppxmanifestEditor.PNG)|


One tab is particularly interesting in this editor : 

![](/assets/PackageAppxmanifestEditorCapabilities.PNG)

UWP apps must explicitly tells which "features" they want to use, *a la* Android's app permissions. Even if it may bring permission fatigue in the long run, it's still a good idea to force the developer to think about which features he wants to use (and it may help to reduce surface attack and lateral movement). However, in my case the Desktop Bridge automatically allow the application every permission needed, thus making the whole thing moot :

![](/assets/FullTrustCapability.PNG)

The created solution will traditionally generate a test key in the form of a `.pfx` file for you to test-sign your package. That's the second great idea imported from iOS/Android app system : every application must be digitally signed, even for a test release. Even if it's a bit of a hassle to set it up, digitally sign binaries greatly simplify system audits and malware detection since the latter usually can't sign their binary stealthily.


However you'll need shell ~100€ for a CodeSigning certificate if you want to publish your application on the Windows Store.


## Handcrafting an appx package


Just right-click on the project solution, select `build` and voila ! Wait, what ? Where's the appx file ?
![](/assets/NoAppxFile.PNG)


It turns out, the project solution create the release folder (`Releasex64` in my case) but does not package it into a `.appx` file ! You need to use a command line executable **which is bundled with the Visual Studio SDK** to do it:


{% highlight posh %}
# Create appx package
$makeappx = "${env:ProgramFiles(x86)}\Windows Kits\10\App Certification Kit\makeappx.exe";
& $makeappx pack /d "./bin/Releasex64" /l /p "./bin/DependenciesAppx_Releasex64.appx"

Microsoft (R) MakeAppx Tool
Copyright (C) 2013 Microsoft.  All rights reserved.

The path (/p) parameter is: "./bin/DependenciesAppx_Releasex64.appx"
The content directory (/d) parameter is: "./bin/Releasex64"
Enumerating files from directory "./bin/Releasex64"
Packing 80 file(s) in "./bin/Releasex64" (content directory) to "./bin/DependenciesAppx_Releasex64.appx" (output file name).
Memory limit defaulting to 8557709312 bytes.

...

Processing ".\bin\Releasex64\resources.pri" as a payload file.  Its path in the package will be "resources.pri".
Processing ".\bin\Releasex64\DependenciesAppx.build.appxrecipe" as a payload file.  Its path in the package will be "DependenciesAppx.build.appxrecipe".
Package creation succeeded.
{% endhighlight posh %}


How difficult it would have been to add this as a custom build step in your project, Microsoft ?

Now that you've actually packaged your application, just double click on it and install it on your local test machine :

![](/assets/AppxUnsignedError.PNG)

Again, for those who don't read French, the appx installer basically tells me it can't install my application since it's not signed. It's the same thing as previously : although the `.pfx` file has been generated by the VS project and is referenced in `Package.appxmanifest`, it does not automatically sign your binaries after building it (which it makes some sense since it does not even do the packaging step before). If I remember correctly, this automatic signing step is done when compiling a driver (even a test-signing one) so it's not like it's a major technical hurdle to set it up.


The useful executable for that step is `signtool` : 

{% highlight posh %}
$makeappx = "${env:ProgramFiles(x86)}\Windows Kits\10\App Certification Kit\makeappx.exe";
$signtool = "${env:ProgramFiles(x86)}\Windows Kits\10\App Certification Kit\signtool.exe";

// Create appx package
& $makeappx pack /d "./bin/Releasex64" /l /p "./bin/DependenciesAppx_Releasex64.appx"
...

// Sign appx package
& $signtool sign /fd SHA256 /a /f "./DependenciesAppx/DependenciesAppx_TemporaryKey.pfx" "./bin/DependenciesAppx_Releasex64.appx"

Done Adding Additional Store
Successfully signed: ./bin/DependenciesAppx_Releasex64.appx
{% endhighlight posh %}

NB : if like me you are using a self generated key file, you need to add the certificate in your local machine hive.

Once correctly packaged and signed, your application is ready to be installed :

![](/assets/AppxSigned.png)

Your installed application is automatically available through the windows start menu, however logos are missing :

![](/assets/AppxMissingAssets.png)


Visual assets for an UWP application is pretty confusing. It expect 8 types of icons : small, medium, wide and large `Tile`; `App Icon`; `Splash Screen`; `Badge` and `Package Logo`. Once those 8 icons filled it will generate approx. 60 scaled icons more or less prettily : 

![](/assets/AppxManyVisualAssets.PNG)


I still don't know which icon goes where (is it the App Icon shown in the systray ? or the Badge Logo ?) and at that point I'm afraid to delete  anyone. As a note, I usually place compiled binaries in a `/build` or `/bin` folder in order not to tamper with the source folders and I learned the hard way that visual assets (i.e. icons) must be placed in the Appx folder before packaging. Here the full script I used after launching a `Release` build : 


{% highlight posh %}
$makeappx = "${env:ProgramFiles(x86)}\Windows Kits\10\App Certification Kit\makeappx.exe";
$signtool = "${env:ProgramFiles(x86)}\Windows Kits\10\App Certification Kit\signtool.exe";

// Copy assets to build folder
Copy-Item "./DependenciesAppx/Assets" -Destination "./bin/Releasex64" -Force -Recurse

// Create appx package
& $makeappx pack /d "./bin/Releasex64" /l /p "./bin/DependenciesAppx_Releasex64.appx"
...

// Sign appx package
& $signtool sign /fd SHA256 /a /f "./DependenciesAppx/DependenciesAppx_TemporaryKey.pfx" "./bin/DependenciesAppx_Releasex64.appx"

Done Adding Additional Store
Successfully signed: ./bin/DependenciesAppx_Releasex64.appx
{% endhighlight posh %}

Honestly Microsoft, how hard would it be to integrate these three simple steps as a custom msbuild step for the `wapproj` solution ?

## Conclusion

After a on-and-off month, I managed to package `Dependencies` in an installable package. I won't probably publish it on the Windows Store, since it looks like a hassle (get a CodeSigning cert, apply for Store publishing, etc.) and there is real value in doing that in my case. If you are currently looking into bundling your application in a appx package, I hope this blog post will be of use to you.


# References


// UWP and Windows Store
* [Why you should not develop apps for Windows 10](http://www.irrlicht3d.org/pivot/entry.php?id=1485) [(and HN comments)](https://news.ycombinator.com/item?id=10936565)
* [Microsoft And The UWP For Enterprise Delusion](https://deanchalk.com/microsoft-and-the-uwp-for-enterprise-delusion-f22fcbbe2757) [(and HN comments)](https://news.ycombinator.com/item?id=16163088)
* [Microsoft wants to monopolise games development on PC. We must fight it](https://www.theguardian.com/technology/2016/mar/04/microsoft-monopolise-pc-games-development-epic-games-gears-of-war)
* [UWP apps > “Normal” desktop apps](https://blog.quickbird.uk/taking-back-control-fbb9ba09e257)
* [Tiles for UWP apps](https://docs.microsoft.com/en-us/windows/uwp/design/shell/tiles-and-notifications/creating-tiles)


// Appx & Desktop Bridge
* [Desktop Bridge](https://docs.microsoft.com/en-us/windows/uwp/porting/desktop-to-uwp-root)
* [Package an app by using Visual Studio (Desktop Bridge)](https://docs.microsoft.com/en-us/windows/uwp/porting/desktop-to-uwp-packaging-dot-net)
* [Prepare to package an app (Desktop Bridge)](https://docs.microsoft.com/en-us/windows/uwp/porting/desktop-to-uwp-prepare)
* [App packager (MakeAppx.exe)](https://msdn.microsoft.com/en-us/library/windows/desktop/hh446767%28v=vs.85%29.aspx?f=255&MSPPError=-2147217396)
* [How to sign an app package using SignTool](https://msdn.microsoft.com/en-us/library/windows/desktop/jj835835(v=vs.85).aspx)
* [Time stamping AppX packages](https://blog.jayway.com/2017/01/16/time-stamping-appx-packages/)

// CLR Images
* [https://msdn.microsoft.com/en-us/library/ms173253.aspx](https://msdn.microsoft.com/en-us/library/ms173253.aspx)
* [https://docs.microsoft.com/en-us/cpp/dotnet/pure-and-verifiable-code-cpp-cli](https://docs.microsoft.com/en-us/cpp/dotnet/pure-and-verifiable-code-cpp-cli)
* [https://msdn.microsoft.com/en-us/library/ffkc918h.aspx](https://msdn.microsoft.com/en-us/library/ffkc918h.aspx)