---
layout: post
title: "Porting an historic Python2 module into Python3"
date: 2017-07-21
---

Python 3 is almost 10 years old. If its adoption has been long and arduous, [now it's recognized that Python 3 will end up supplanting Python 2.7](https://blogs.msdn.microsoft.com/pythonengineering/2016/03/08/python-3-is-winning). However, if most of the popular libraries already are Python 3 ready, that's not the case for the rest of the tail.

While [there is a trove of fantastic RE tools written in Python](http://pythonarsenal.com/), most of them are written for Python 2.7. This is partially explained by the fact that the reverse engineering community had a significant "old guard" that still insist on using Hiew, SoftIce and VC6 when reversing code (along with software "jocks" activating every experimental security bells and whistle for fun and profit). I also hold HexRays partially responsible for this situation since they keep on shipping IDA without any Python3 support (*maybe in the v7, who knows ?*). 

[`pdbparse`](https://github.com/moyix/pdbparse) is a really useful Python module/script for parsing PDB files (debug symbols files on Windows) especially on a Linux host. It's pretty much the historical goto solution across the RE board. Problem though, it's becoming deprecated (the project is 5 y.o) and there is only a Python 2.7 build available. What following is my adventure into attempting to "resuscitate" a Python2 legacy project and hopefully a tutorial for others.

Summary :

* [Step 0 : Prepare virtual environments](#Step0)
* [Step 1 : Try to build the lib](#Step1)
* [Step 2 : Test a few examples](#Step2)
* [Step 3 : Write some unit tests if possible](#Step3)
* [Step 4 : Use a code coverage tool](#Step4)
* [Final Step : Make your PR accepted by the maintainer](#Step5)
* [Bonux Steps](#Step6)
 
NB : [This cheatsheet from the `future` people is awesome when dealing with Python2/3 compatibility issues]( http://python-future.org/compatible_idioms.html)

<!--more-->

## <a name="Step0"></a>  Step 0 : Prepare virtual environments

You will want to switch python versions and have several concurential build to check for non-regressions.

Problem in this particular project : there is a C extension for demangling C++ names. Which means we also need the Visual Studio and GCC
compiler corresponding to each Python versions.

* (Windows) Python 3.6 : msvc 2015
* (Linux) Python 3.4 : GCC 4.8.4
* (Linux) Python 2.7.6 : GCC 4.8.4
* (Windows) Python 2.7 : mvsc 2008. Ouch, time for some software archeology. Thankfully Microsoft plays nice and conveniently [provides the exact compile toolchain needed to build C modules for Python 2.7](https://www.microsoft.com/en-us/download/details.aspx?id=44266).

[WSL](https://msdn.microsoft.com/fr-fr/commandline/wsl/about) really shine here : I could manually test my modifications on the 4 virtualenv on the same Windows host !

However that is not sufficient to build the C extension since [the module extension definition format has evolved between Python2 and Python3](http://adamlamers.com/post/NUBSPFQJ50J1), now we need to define a `PyModuleDef` structure and a `PyMODINIT_FUNC` entry point in the C code. Thankfully I've some experience tinkering with Python modules, otherwise I would have thrown the towel before even started porting the actual Python code to Python 3.

## <a name="Step1"></a> Step 1 : try to build the lib

Just type `python setup.py build`.

Usually it will catch the following syntax errors :

	
Python 2.7 only 			                     | 			Python 2.7 & Python 3				   
-------------------------------------------------|----------------------------------------------------------------
`except Error, e` 						  	     |  		`except *Error as e`				   
`print "something"` 						     | 			`print ("something")`				   
`print >> sys.stderr, "usage: %s <exe> <pdb>" % sys.argv[0]` | `from __future__ import print_function` <br/> `print ("usage: %s <exe> <pdb>" % sys.argv[0], file=sys.stderr)` |

## <a name="Step2"></a>  Step 2 : test a few examples

if you're not really familliar with the library or there is no test suite, play around with the modules and see what breaks.

For example :

Python 2.7 only 				  | 			Python 2.7 & Python 3				   
-----------------------------------|----------------------------------------------------------------
`open(raw_file, 'w')` | `open(raw_file, 'wb') # if it was meant to read binary data`
`xrange` | `from builtins import range`
`cStringIO.StringIO` | `io.BytesIO`
`from urllib2 import urlopen` | `try :` <br/> `from urllib2.request import build_opener` <br/> `except ImportError:` <br/> `from urllib2 import urlopen`  
`self.streams = zip(sizes, page_lists)` | `# zip return a generator on Py3` <br/> `self.streams = list(zip(sizes, page_lists))`
`dbg.GUID.Data4.encode('hex')`  |  `binascii.hexlify(dbg.GUID.Data4).decode('ascii')`


And that's where the "fun" begins.

Usually, a dev writing a Python 2.7 library will not discriminate between bytes-like buffers and ASCII strings (since the language doesn't either) which means to have to code review everything str operations and ask yourself if there is a bytes operation implied or is it really a string one.

Moreover there are catchable errors, but also silent bugs :

{%highlight python %}
>>> b"RSDS\0ksjgsg".split("\0")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: a bytes-like object is required, not 'str'
>>> "RSDS" in b"RSDS\0ksjgsg"
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: a bytes-like object is required, not 'str'
>>> b"RSDS\0ksjgsg".find("DS")
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: a bytes-like object is required, not 'str'
>>> b"RSDS\0ksjgsg"[0:4] == "RSDS"
False						# SILENT !!
>>> "RSDS" + b"0ksjgsg"
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: Can't convert 'bytes' object to str implicitly
{%endhighlight python %}
<br/> 

`pdbparse` is among the worst offendants I've seen since, as any parsing library, it will happily handle and transform bytes-like data as string.

Examples found in the wild :

* `construct.Magics` defined as a strings, not bytestrings
* `construct.CString` now return a bytestring in Python3, which has to be decoded after
* a ton of `bytes/str` incompatibilities 

As a side note, I'm kinda bummed out that Python devs decided not to backport [Python type hints](https://www.python.org/dev/peps/pep-0484/) in Python2.7. That would have been an fantastic tool to explicit which type of str/bytes buffer expected as input or output. Instead I wrote it down in the functions' comments, `javadoc`-style :


{%highlight python %}
def get_external_codeview(filename):
    """
        Extract filename's debug CodeView information.
        Parameter:
            * (str) filename, path to input PE 
        Return :
            * (str) the GUID
            * (str) the pdb filename
    """
    pe = PE(filename, fast_load=True)
    dbgdata = get_debug_data(pe, DEBUG_TYPE[u'IMAGE_DEBUG_TYPE_CODEVIEW'])
    if dbgdata[:4] == b'RSDS':
        (guid,filename) = get_rsds(dbgdata)
    elif dbgdata[:4] == b'NB10':
        (guid,filename) = get_nb10(dbgdata)
    else:
        raise TypeError(u'Invalid CodeView signature: [%s]' % dbgdata[:4])
    guid = guid.upper()
    return guid, filename
{%endhighlight python %}

## <a name="Step3"></a> Step 3 : write some unit tests if possible

Writing unit tests for a parsing library is difficult (not to mention tedious), and doubling so for PDB since it's a complex file format embedding several "streams" of file informations and carries a lot of deprecated features. As an example, nowadays every `PE` executable embed `Codeview` debug infos in a `RSDS` header, there is no `NB_09`/`NB_10`/`DEBUG_MISC` header used anymore.

The `LLVM` project  has a test executable `llvm-pdbutil` which can produce pdb files based on a yaml specification. That should be a good idea to use it for a test suite, but since I'm lazy I went instead for best effort and use a code coverage tool.

## <a name="Step4"></a>  Step 4 : use a code coverage tool



Since I can't prove my modifications does not introduce regressions, at least I can try to exercice every paths in the source code in order to catch the most obvious bugs. [`coverage` is a really good Python code coverage tool](https://pypi.python.org/pypi/coverage). It can be run as a script and generate a useful html report upon completion showing you which paths were never taken. In order to cover the most ground, I mainly used three differents PE & PDB :

* A recent `rpcrt4.dll` (`9A2A73D551A746701290A1681D18F0BF1`) from my own System32 folder
* A somewhat recent C++ `1394ohci.pdb` (`8fc534deed5f4c0d98e17471b68a0c402`) file which does contain a lot of type information as well as mangled C++ names.
* A reaaaaally old Win2K `ntdll.dll` (`41E648E07D000`) to survey the legacy code (`DEBUG_MISC_INFORMATION` in PE, `PDBv2` parsing, etc.). Surprisingly enough, Microsoft still "support" the symbol lookup on their public symbol server even for a 10+ y.o. DLL.

![Code coverage of pdbparse testing](/assets/pdparse_coverage.PNG)

The missing statements are mostly errors branchs that raise exceptions. To be honest, even if I covered most of the library I'm pretty sure there still subtle bugs in how parsed data is handled. However in my opinion, people forgive software bugs in open-source "free" libraries but less so when it's syntax bugs, especially infuriating ones such as str/bytes incompatibilities.

Here is the coverage script: 

{%highlight python %}
import os
import os.path
from examples import symchk, pdb_files, pdb_dump, pdb_print_gvars

import pdbparse
from pdbparse import dbi
from pdbparse.symlookup import Lookup
from pdbparse.undname import undname
import binascii
import re

""" COMMAND LINE """
# coverage run --source pdbparse --omit="pdbparse/postfix_eval.py,pdbparse/undecorate.py" test_coverage.py

if __name__ == '__main__':

    os.chdir("./test")
                
    """ Testing PDB parse """
    pdb = pdbparse.parse("./1394ohci.pdb") # 8fc534deed5f4c0d98e17471b68a0c402
    dbg = pdb.STREAM_PDB
    guidstr = u"%08x%04x%04x%s%x" % (
        dbg.GUID.Data1, dbg.GUID.Data2, dbg.GUID.Data3,
        binascii.hexlify(dbg.GUID.Data4).decode('ascii'),
        dbg.Age)
    print(guidstr)

    pdb = pdbparse.parse("./rpcrt4.pdb", fast_load = False)
    pdb.read(list(range(60)))
    old_pdb = pdbparse.parse("./ntdll.pdb", fast_load = False)
    old_pdb.read(list(range(60)))

    """ Testing PE debug information parsing """
    symchk.handle_pe("./rpcrt4.dll") # 9A2A73D551A746701290A1681D18F0BF1
    symchk.handle_pe("./NTDLL.DLL") # 41E648E07D000
    pdb_files.main("./rpcrt4.dll")
    
    """ Testing PDB Stream informations handling """
    pdb_dump.main("./1394ohci.pdb")
    pdb_dump.main("./rpcrt4.pdb")
    pdb_dump.main("./ntdll.pdb")

    """ Testing symbol lookup """
    rpcrt4_ba = 0x100000;
    rpcrt4_sym = Lookup([("./1394ohci.pdb", rpcrt4_ba)])
    for i in range(0x1337):
        sym_name = rpcrt4_sym.lookup(rpcrt4_ba + i*0x10)
        sym_name = rpcrt4_sym.lookup(rpcrt4_ba + i*0x10) # triggering sym lookup cache 
        """ Testing symbol undecoration """
        mod, symbol = sym_name.split("!")
        symbol = re.split("(\+|\-)", symbol)[0]
        print("raw [%s], undecorated [%s]" % (sym_name, undname(symbol)))

    """
    Testing PDB parsing on the local symbol cache
    Beware : several hours of processing ahead since it's single threaded
    i = 0
    for dirpath, dnames, fnames in os.walk(os.environ[NT_SYMBOL_PATH]):
        for f in fnames:
            if f.endswith(".pdb"):
                fpath = os.path.join(dirpath,f)
                try:
                    print(fpath)
                    pdb = pdbparse.parse(fpath)
                    #pdb_print_gvars.main(fpath, "0xdeadbeef")
                    i+=1
                except Exception as e:
                    raise e
                    #pass
        if i > 10000:
            break
    """

    os.chdir("../")
{%endhighlight python %}


## <a name="Step5"></a> Final Step : Make your PR accepted by the maintainer

When you've sufficiently confident/proud in your fixes, all that rest is to wrap it up and send it to the library maintainer in a pull request for him/her to upstream the modifications. [While sending a PR on Github](https://yangsu.github.io/pull-request-tutorial/)[ implies having a black-belt in Git](https://stackoverflow.com/questions/14680711/how-to-do-a-github-pull-request), the last hurdle is mostly societal not technical.

There are various reasons why you Pull Request may not be accepted : [orphaned project](https://github.com/NtQuery/Scylla/pull/39), [the maintainer does not accept PR](https://github.com/Microsoft/MSRC-Security-Research), [insufficient or incomplete bug fix](https://github.com/aquynh/capstone/pull/511), [need to sign NDA/CLA for sending PR](https://github.com/Microsoft/microsoft-pdb/pull/27), [no sticking to the maintainer's path guidelines](https://github.com/android/platform_dalvik/pull/2), [no test proving there is no regression](https://github.com/dotnet/corefx/pull/55), etc.

This is the last step but also the first step : before going into hacking/fixing/improving a third-party library you need to guesstimate the success ratio of your fix of being upstreamed and decide if it worth taking the shot. Fortunately in my case even though `pdbparse` haven't seen major progress and new features for several years now, the maintainer do minor fixes ([here is a recent example](https://github.com/moyix/pdbparse/commit/a8630a788509a8bd1c654d89629de15d01f9ce00)) and review PR ([merged PR for pdbparse](https://github.com/moyix/pdbparse/pulls?utf8=%E2%9C%93&q=is%3Apr%20is%3Aclosed%20is%3Amerged)) which is a good heuristic for PR acceptation.

Also take into account that most of the public third-party tool are usually created and maintained by a single person. Moreover this man or woman probably does not live from his/her open-source work which means he/she has another full-time job and thus limited attention span and motivation to review your patch. Try to make it really palatable for them by explaining in length what the patch do, your motivation behind it, steps taken to ensure correctness and non-regression and the possible pitfalls behind your fixes. Also be patient : the maintainer has to double check your patch (which implies setting up the correct dev's environment) and it can take weeks for your PR to be accepted.

Anyway, [my PR for Python 3 support has recently landed in pdbparse's main tree](https://github.com/moyix/pdbparse/pull/35) an hopefully will be present on pipy in the future.


## <a name="Step6"></a>  Bonux Steps 

Semi-ophaned projects like `pdbparse` stopped evolving along with the Python ecosystem so they tend to "accumulate" backwards compatilibity issues and deprecated constructions. They also miss out on new features that can greatly simplify the code like context managers or async calls for downloading pdb. When dealing with such library, one temptation would be to digress every time we find a "code smell" in the code source. However do not forget the objective in mind : you are here to porting the library to Python 3, not to fix every bug in it (which could make the object of another pull request).

As an example, `pdbparse` relies of `construct` for parsing PDB internal binary structures. [`construct`](https://github.com/construct/construct) is also a popular semi-ophaned open-source library useful to create binary parsers in Python. After several years of sleep, the `construct` recently became active again which is a great news. However they recently did introduce a backward compatilibity break in the API for `Struct` and integer parsing (in the v2.8 release). Instead of downstreaming the fixes in `pdbparse`, I instead future-proofed the dependency by [limiting the version supported in the `setup.py`](https://github.com/moyix/pdbparse/pull/36/files) file (and in a separate pull request) until I'm confident the new API is stable enough. This fix really is far from perfect and it add upon Python's dependency hell : now `pdbparse` probably need to be installed in a `virtualenv` since it is incompatible with `pip`'s default `construct` library.  :nauseated_face:

Forgive me for raging, but **who the fuck breaks a 5+ years old API ?** `construct` guy, I'm really glad that you're dusting off a really useful library but [this change](https://github.com/construct/construct/issues/271) is way too radical. At this point you're just giving ammunitions to people that claim that "Python is not for serious work" (well, who can blame them when you see the whole Python2/Python3 debacle ?). There must have been hundreds if not thousands of scripts and libraries depending on `construct` that insta-broke as soon as the users typed `pip install --upgrade` on their machines. If you introduce drastic breaking changes, at least update the project name to `construct2` or provide a "legacy" namespace for backwards fixes : `from construct import *` can become `from construct.legacy import *` for third-party libraries that does/can not downstream your API modifications.

On a final note, I want to get backtrack on the C module used in `pdbparse`, `undname`. `undname` is a C source code copied from Wine that add support for undecorate symbol on Unix hosts (on Windows machines, `dbgengine.dll` does the work), which make them more human readable : `1394ohci!?AddConfigRomEntry@ConfigRom@@AEAAJPEAU_SET_LOCAL_HOST_PROPS3@@PEAX@Z` becomes `__int32 __fastcall ConfigRom::AddConfigRomEntry(ConfigRom *this, struct _SET_LOCAL_HOST_PROPS3 *, void *)`.

However I have several gripes with writing a C extension module for CPython. As a start, it's hard to write correct C/Python glue code manually : you need to understand how `PyArg_ParseTuple`, `PyArg_ParseTupleAndKeywords` and `PyArg_Parse` works, what the correct string formats are, and how to manipulate `PyObject` and `Py_buffer` data structures. And also you need to update it every time you make a change in the API, which means you need to be also a decent software architect. The amount of Python developers that are also fluent in C and do not shy away from get their hands dirty manipulating Python logic in a low level langage is vanishingly small. By writing a C extension, statistically you're probably the only
developer in your vicinity that can maintain your Python library.

There is also the whole reference counting insanity. [From what I've understand](https://docs.python.org/3/extending/extending.html#reference-counts) (which is probably close to nothing) you need to explicitely tell the Python `GC` when it can kill temporary `PyObject` variables by using the `Py_INCREF(x)` and `Py_DECREF(x)` macros. Which I found incredibly difficult to do it correctly except for trivial examples. That's probably why smart Python projects that use C code like [Capstone](https://github.com/aquynh/capstone) use `ctypes` to provide Python bindings, and those who use C++ bindings like [`PyQt`](https://riverbankcomputing.com/software/pyqt/intro) ends up using [`sip`](https://www.riverbankcomputing.com/news/sip-4193).

Building a C extension for Python (whether via `ctypes`, `ffi`, `sip` or natively) also greatly encumber the building process since you're not platform-agnostic anymore. One strengh of Python is it does not matter (most of the times) that the script run on a unix box or a windows one, on which CPU arch and -partly- on which Python's minor version : this is encapsulated away by the Python interpreter. If you go native, you will need to provide binaries for every release of a new major and minor version of the CPython's interpreter for 32 and 64-bit CPU on linux, darwin and windows platforms alike : at that point it is mandatory to plug your project to Continuous Integration tools like Travis or Appveyor, otherwise you might as well go mad maintaing dozens of builds for EACH release of your own library. Moreover on Windows the building process was historically so bad that you usually end up downloading [binairies from some random guy on the Internet](http://www.lfd.uci.edu/~gohlke/pythonlibs), over HTTP of course. And no, the python `wheel` file format does not protect against MITM attacks. :disappointed:

Personally I reserve the use of Python C extensions to specific libraries that have to rely on native performances (like `numpy` of `capstone`) or those tightly coupled to a platform (like [Local KD](https://github.com/sogeti-esec-lab/LKD/)). I told `moyix`, the `pdbparser` maintainer, my stance on `undname` but he finds it useful and still wants to keep it, and that's okay since it's his project.
