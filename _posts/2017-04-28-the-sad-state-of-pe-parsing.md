---
layout: post
title: "The sad state of PE parsing"
date: 2017-04-28
---


The title of this blog's post might be clickbaity, but here the executive summary : there is no gold standard open source library for `PE` parsing and features extraction in "native" (i.e. unmanaged) code. Moreso all
widely-used `PE` parsing libraries contains subtle flaws.


<!--more-->

If there is one thing the Open Source ™ has done right, it's to be able to construct and build upon true and tested libraries for file format handling. Here is an excerpt :

* `zlib` for zipped contents
* `libpng`, `libtiff`, `libtiff` for image files
* `ffmpeg`, `libav`, `libvpx`  for audio/video 
* etc.

However there is none for decoding and manipulating binairies, especially `PE` binaries.

There is a trove of commercial software for `PE` features extraction (`PEStudio`, `CFF Explorer`, `PE Insider`, `PE explorer`, `Ida`, `010 editor` with community-sourced parsers, `PEview`) as well as closed source freeware (`depends`, `dumpbin`, `windbg`) there is actually no leading open-source library for `PE` parsing.

That's weird since I think every reverse engineer and Windows security specialist must write half a dozen shitty `PE` parsers every year in order to extract specific informations from third-party binaries.

That's doubly weird that, in this day and age, no AV companies decided to pool resources in order to ship an secure open-source library (like hardware vendors did for `Linux`). I'm pretty sure every AV engine on Windows embed in some way a `PE` parser, even partially.  


I recently wanted to extract features from a set of binairies in a scriptable way. However, I needed for the script to be fully autonomous (i.e. I could drop it on an external system and run it). Here's the options I found :

* `Python` : `pefile` (for `PE` parsing) + `pyexe/freeze/etc.` in order to create a static runtime. I've been previously been burned by subtle ``pyexe` to not wanting to goin that direction again
* `Javascript` : `PE.js` for `PE` parsing. Problem : the only way I think of creating a static runtime out of it is to compile and deploy a custom version of `V8/SpyderMonkey/Chackra` or go full way and build an `Electron` application.
* `Powershell` : currently the "sexiest" way to do it, however there is no rock solid `PE` parser in `Powershell` or `.NET`. Maybe one day it will pop out in Forshaw's `NtApiDotNet` project.
* `Go/Java/Scala/Rust/${YOUR_FAVORITE_LANGUAGE}` : I'm not familiar with those, sorry.


Since there was no satisfying solution (at least to me) I decided to go the `C` way (or `C++` with `C`-like `API`) since it's the *lingua franca* of software interoperability. Again, I was kinda let down by the "choice" proposed :

* `Windows` offers some internal structures and few `API` to read `PE` files (`IMAGE_XXX_HEADER`, etc.) but you have to roll your own implementation every time.
*[`COFFI`](https://github.com/serge1/COFFI) Unsupported PE/COFF parser in C++.
* [`trailsofbits/pe-parse`](https://github.com/trailofbits/pe-parse) :  my main gripe against this parser is it does not return structured data, you can only registers callbacks on parsing events. (Granted, it's only a "lightweight" parser.)
* `lldb/Plugins` : every debugger implement a `PE` parser, `lldb` is among those. However it does have some missing features (it does not list exports by ordinals).
* `radare\bin` : `radare2` (like `ida`) embed binary parsers. It's probably the most solid and feature-full one I've seen, even if the source code style deter me from wanting to use it.
* [`quarkslab/leif`](https://lief.quarkslab.com/) : the new kid one the `PE` parsing block. However I did not managed to compile it on Windows (it redefine as C++ enums name that macro-defined in `Windows` WDK like `IMAGE_FILE_MACHINE_AMD64`). After two hours of trying, my patience ran short.
* [`ProcessHacker\phlib`](https://github.com/processhacker2/processhacker) : ProcessHacker actually implement a pretty feature wide `PE` parser in its library `phlib`.

Since I've previously worked on `ProcessHacker` and I was somewhat familiar with the codebase, I decided to give it a shot. Surprinsingly enough, it was easy to rip the necessary code : just copy `phlib` and `phnt` (headers only)  folders in your project root dir and as long as you compile as a static runtime, you're set.

Anyway, I'm working on removing bugs in `phlib` whenever I found some :

* [missing exports by ordinals](https://github.com/processhacker2/processhacker2/pull/125)
* [rva for exports not correctly computed](https://github.com/processhacker2/processhacker2/commit/f73aea5e353b40cc1048fe64d8b26a6fdc2adfd9)
* [undecorate C++ names](https://github.com/processhacker2/processhacker/pull/139)
