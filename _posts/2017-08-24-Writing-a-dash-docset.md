---
layout: post
title: "Writing a custom dash docset for Powershell docs"
date: 2017-08-24
---

Powershell has became the default shell since Windows 10 Creator's Update and it's starting to become more than just a framework for malware deployment ([not my words](https://twitter.com/MalwareTechBlog/status/891005091985203201)). Apart from the langage itself which feel alien to me (is it a shell ? a scripting langage ? a programming langage ? a duck ?) my biggest gripe with Powershell is the lack of documentation accessible from an offline network (or simply without direct access to the Internet). For a shell that have been created for sysadmins, you would imagine MS would have thought of shipping Powershell with "batteries included". I can not count the amount of times I needed to do a powershell-related search on my smartphone while operating on a detached network.

Secondly, until recently there was no easy way to query help for a specific API (for example `Get-ChildItem`). There is evidently the `Get-Help` cmdlet which show the "manpage" associated with the specific cmdlet but that too grab the documentation from the Internet ! At least since Powershell 3.0 there is also the `Save-Help` cmdlet which can do a bulk download of the manpages of every posh modules installed on a system and `Update-Help` to update it on a separate machine.

![Look Ma ! It's like on a Unix system !](/assets/posh-help-gci.PNG)


However I'm not a haxx0r elite programmer and for the life of me I can't spend my time in a text-based console world. I grew up with click-based interfaces and browsers (not necessarly web browsers) therefore I'm way more at ease searching for information in an environment where a ["mistype"](https://en.wiktionary.org/wiki/mistype) cannot do serious damages on the system. I also like to click-click on colored boxes and purple links :smile:

Maybe to alleviate my silent issue (I'm surely not the only one bothered by this), the people from`microsoft.docs.com` recently launched a Powershell modules browser in which you can do full-text search for Powershell Cmdlet : 

{% raw %}

<blockquote class="twitter-tweet tw-align-center" data-lang="fr" >
<p lang="en" dir="ltr">Announcing the <a href="https://twitter.com/hashtag/PowerShell?src=hash">#PowerShell</a> Module Browser - an easy way to search all PowerShell modules and cmdlets from Microsoft<a href="https://t.co/0zX7dCjOhE">https://t.co/0zX7dCjOhE</a> <a href="https://t.co/OBmyKVcOZe">pic.twitter.com/OBmyKVcOZe</a></p>&mdash; docs.microsoft.com (@docsmsft) <a href="https://twitter.com/docsmsft/status/894951611306569729">8 août 2017</a>
</blockquote>
<script async src="//platform.twitter.com/widgets.js" charset="utf-8"></script>
{% endraw %}

This a great improvement but unfortunately it's still online only. However the docs structure is sufficiently simple and well structured enough to be packaged in a dash docset for offline viewing. What follow in this blog post is how I proceed to build the docset as well as some "tips" for more advanced/obscure topics such as package navigation links and themes support.


**TLDR** : the generation script is here [https://github.com/lucasg/powershell-docset](https://github.com/lucasg/powershell-docset) but is subject to regular changes and breakages. Better download only the generated docsets : [https://github.com/lucasg/powershell-docset/releases](https://github.com/lucasg/powershell-docset/releases)
<!--more-->

The document on how to build a dash-compatible docset is quite clear and straightforward : [https://kapeli.com/docsets#dashDocset](https://kapeli.com/docsets#dashDocset).

# Archive structure


Any dash compatible docset must follow a certain folder hierarchy, like any packaging format. Nothing crazy here, 
we have a layer of metadata on top of arbitrary data : 

```
.
└ $docset_name.docset/
  ├ icon.png
  ├ icon@2x.png 		# optional for retina displays
  └ Contents/
    ├ Info.plist
    └ Resources/
      ├ LICENSE
      ├ docSet.dsidx
      └ Documents/
        └  * 			# your documentation lives here

```

Documents can contain whatever you want, but usually you want to reproduce the url paths if your docs comes from a website.
In my case :

```
.
└Documents\
  └ docs.microsoft.com\
     └ en-us\
        ├ index.html                                   # docset start page
        └ powershell\
            └ module\
               ├ CimCmdlets\
               ├ Microsoft.PowerShell.Archive\
               ├ Microsoft.PowerShell.Core\
               ├ Microsoft.PowerShell.Diagnostics\
               ├ Microsoft.PowerShell.Host\
               ├ Microsoft.PowerShell.LocalAccounts\
               ├ Microsoft.PowerShell.Management\
               ├ Microsoft.PowerShell.Security\
               ├ Microsoft.PowerShell.Utility\
               ├ Microsoft.WSMan.Management\
               ├ PackageManagement\
               ├ Pester\
               ├ PowerShellGet\
               ├ PSDesiredStateConfiguration\
               ├ PSDiagnostics\
               └ PSReadLine\
                   ├ Get-PSReadlineKeyHandler.html
                   ├ Get-PSReadlineOption.html
                   ├ index.html                        # PSReadLine module index page
                   ├ PSConsoleHostReadline.html
                   ├ Remove-PSReadlineKeyHandler.html
                   ├ Set-PSReadlineKeyHandler.html
                   └ Set-PSReadlineOption.html
```
<br>

Icons are straigthforward :
{% highlight python %}
download_binary("https://github.com/PowerShell/PowerShell/raw/master/assets/Powershell_16.png", docset_path / "icon.png")
download_binary("https://github.com/PowerShell/PowerShell/raw/master/assets/Powershell_32.png", docset_path / "icon@2x.png")
{% endhighlight python %}
<br>

`Info.plist` is a xml file describing the archive:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
  <dict>
    <!-- identifier for sub search -->
    <key>CFBundleIdentifier</key>   
    <string>posh</string>

    <!-- display name -->
    <key>CFBundleName</key>
    <string>Powershell</string>

    <!-- display fallback url, when the path is broken -->
    <key>DashDocSetFallbackURL</key>
    <string>https://docs.microsoft.com/en-US/powershell/module/</string>

    <!-- start page -->
    <key>dashIndexFilePath</key>
    <string>docs.microsoft.com/en-US/index.html</string>

    <!-- not sure what is it -->
    <key>DashDocSetFamily</key>
    <string>posh</string>

    <!-- not sure what is it -->
    <key>DocSetPlatformFamily</key>
    <string>posh</string>

    <!-- Yes for any Dash docset. Otherwise it is treated as an Apple docset  -->
    <key>isDashDocset</key>
    <true/>

    <!-- enable this if you need to execute some Javascript -->
    <key>isJavaScriptEnabled</key>
    <true/>

  </dict>
</plist>
{% endhighlight xml %}

The only tricky item in `Info.plist` is `dashIndexFilePath` : its value must respect the relative path within `Documents`.

(NB: Duplicate entries in `Info.plist` will make Velocity crash when importing a docset, so double check there is none when you create your archive).

# Crawling website endpoint

There is several ways to create a dash docset from existing documentation :

* Generating it from a compatible code comments system (Doxygen, GoDocs, etc) : [Docset Generation Guide](https://kapeli.com/docsets#docsetSources)
* [docsets generators](https://github.com/zealdocs/zeal/wiki/Third-Party-Resources#docset-generators) such as [dashing](https://github.com/technosophos/dashing) which can automagically create a docset from an existing html documentation using CSS selectors. (Fun fact : dashing is created by a Microsoft Azure guy and written in `Go`. Go figure.)
* Custom creating one with lots of scraping, copious amounts of sweat and a bit of luck/skill.

In my case, the Powershell Modules documentation is public and reside here : [https://github.com/PowerShell/PowerShell-Docs](https://github.com/PowerShell/PowerShell-Docs). However the documentation is written in a markup langage suited for [DocFx](https://dotnet.github.io/docfx/), Dotnet documentation generator. DocFx is not dash-compatible yet, so the first option is out.

Theoretically, I could have tried to generate the html documentation using `DocFx` and converting it in a docset using `dashing`, but that would have implied to use two tools I don't have any experience in and the resulting docset can be difficult to debug (as you will see further below) if anything went wrong somewhere. If the Microsoft Azure people want to do it this way, they are more than welcome. So out with the second option.

Third option it is, then.


Fortunately, as I said previously, the Powershell modules doc website is well structured and provide a json file describing the table of contents : [https://docs.microsoft.com/en-us/powershell/module/psdocs/toc.json?view=powershell-6](https://docs.microsoft.com/en-us/powershell/module/psdocs/toc.json?view=powershell-6) (you change the powershell version number for previous major versions). The `toc.json` is basically the sitemap and list all the modules and cmdlets for a given Powershell version.

Python `urllib` and `requests` are really life savers in those situations : 

{% highlight python %}
import os
import json
import urllib.parse

import requests # pip install requests

def download_textfile(url, output_filename):
  """ wget a text file """
  r = requests.get(url)
  with open(output_filename, 'w', encoding="utf8") as f:
    f.write(r.text)

def crawl_posh_documentation(documents_folder : str):
  """ Parse and download Powershell modules documentation """

  index = "https://docs.microsoft.com/en-us/powershell/module/?view=powershell-6"
  modules_toc = "https://docs.microsoft.com/en-us/powershell/module/powershell-6/toc.json?view=powershell-6"

  index_filepath = os.path.join(documents_folder, "docs.microsoft.com", "en-us", "index.html")
  download_textfile(index, index_filepath)

  modules_filepath = os.path.join(documents_folder, "modules.toc")
  download_textfile(modules_toc, modules_filepath)

  with open(modules_filepath, 'r') as modules_fd:
    modules = json.load(modules_fd)

    for module in modules['items'][0]['children']:
      module_url = urllib.parse.urljoin(modules_toc, module["href"])

      module_dir = os.path.join(documents_folder, base_url, module['toc_title'])
      os.makedirs(module_dir, exist_ok = True)

      r = requests.get(module_url)
      module_filepath = os.path.join(module_dir, "index.html")
      download_textfile(module_url, module_filepath)
        

      for cmdlet in module['children']:
        cmdlet_name = cmdlet['toc_title']
        
        
        if cmdlet_name == "About" or cmdlet_name == "Providers": 
          continue # skip special toc
        cmdlet_urlpath = cmdlet["href"]
        cmdlet_url = urllib.parse.urljoin(modules_toc, cmdlet_urlpath)

        cmdlet_filepath = os.path.join(module_dir, "%s.html" % cmdlet_name)
        download_textfile(cmdlet_url, cmdlet_filepath)
{% endhighlight python %}

With only 40 lines of Python, I was able to do a full website save into disk.

# Creating database

Not a database expert, so I've copied from [`llvm-to-dash.py`](https://github.com/iamaziz/llvm-dash/blob/master/llvm-to-dash.py).
It works for my purposes.

{% highlight python %}
sqlite_filepath = os.path.join(resources_dir, "docSet.dsidx")
db = sqlite3.connect(sqlite_filepath)
cur = db.cursor()
cur.execute('CREATE TABLE searchIndex(id INTEGER PRIMARY KEY, name TEXT, type TEXT, path TEXT);')
cur.execute('CREATE UNIQUE INDEX anchor ON searchIndex (name, type, path);')
...
cur.execute('INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?,?,?)', (name, type, path))
cur.execute('INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?,?,?)', (name2, type2, path2))
cur.execute('INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?,?,?)', (name3, type3, path3))
cur.execute('INSERT OR IGNORE INTO searchIndex(name, type, path) VALUES (?,?,?)', (name4, type4, path4))
etc.
db.commit()
db.close()
{% endhighlight python %}
<br>

As said in the official documentation, there is a limited number of entry "types" : [https://kapeli.com/docsets#supportedentrytypes](https://kapeli.com/docsets#supportedentrytypes). Even though Microsoft keeps on using `Cmdlet` to designate Powershell commands I've listed them under the "Command" entry type.


# Packaging the docset

Once you've got every html source files and you've populated the database, just tar-gz the folder while conserving the structure within the archive (relative path folders). 
This python snippet does the trick :

{% highlight python %}
def make_docset(source_dir, dst_filepath, filename):
    """ 
    Tar-gz the build directory while conserving the relative folder tree paths. 
    Copied from : https://stackoverflow.com/a/17081026/1741450 
    """
    dst_dir = os.path.dirname(dst_filepath)
    tar_filepath = os.path.join(dst_dir, '%s.tar' % filename)
    
    with tarfile.open(tar_filepath, "w:gz") as tar:
        tar.add(source_dir, arcname=os.path.basename(source_dir))

    shutil.move(tar_filepath, dst_filepath)
{% endhighlight python %}


At the end of it, you -hopefully- get a well formed docset archive that can be imported into `Dash`, `Velocity` or `Zeal` : 

![Powershell docset on Velocity](/assets/posh-no-nav.PNG)

Notice anything ? It's really ugly, except if you're a [brutalist afficionado](http://brutalistwebsites.com/) (disclaimer : I'm not). Furthermore, you can't tell from an image, but navigation links are broken : you can't click-click and turning those items purple, which makes me sad.


Fixing paths imply to parse and rewrite the downloaded html files, while supporting css "themes" usually ends up using a headless web browser to do the scraping. They are cumbersome and brittle operations, which is probably why most [user-contributed docsets](https://github.com/Kapeli/Dash-User-Contributions#readme) don't bother doing it.

# Fixing paths

In order to have "clickable" links, you need to rewrite `<a>` anchors and convert "dynamic" `href` to "static" ones. For example transform :

`<a href="/powershell/module/Microsoft.PowerShell.Archive/?view=powershell-6" data-linktype="relative-path">` 

into :

`<a href="powershell/module/Microsoft.PowerShell.Archive/index.html" data-linktype="relative-path">`


There is also a fuckton a nav bars , dropdown menu and sidebars DOM elements that need to be removed. Fortunately for me, the docs html pages are well-formated and there is little to none variability in how the DOM is structured. I have basically three types of pages : the `index.html`, each own modules' index.html and each cmdlets page. 

[`BeautifulSoup`](https://www.crummy.com/software/BeautifulSoup/bs4/doc/) is the go-to python library for parsing html contents and it does the job admirably. 


# downloading themes and javascript



With working uri paths your docset is more "dynamic" and "discoverable", but it's still fugly. I don't know for `Dash` but `Velocity` and `Zeal` are basically web browsers (`Zeal` use `QtWebEngine` and `Velocity` relies on the `Chromium Embedded Framework`) so it's possible to "theme" your docset via css.

Problem though, the css theme location is not present in the `toc.json` file so you need to find it on your own. That's where Chrome/Firefox/Edge's devtools are your friend :

![Chrome Devtools's Source pane](/assets/posh-docs-microsoft-sources-tree.PNG)

You will need to reconstruct the exact same tree directory structure in your docset package and also fix urls. I fortunately didn't had to replicate the cross-domain "cdn" requests. "Dynamic" pages like powershell `index.html` rely on Javascript to defer resource loading and some DOM elements are lazy-loaded via a `Promise`. In order to locate and retrieve all the necessary resources I end up using a `webdriver` (`selenium` + `phantomjs`) which is a headless web browser you can automate. This is way slower than using simple HTTP requests, but the result is more accurate.

Follow below is the result with the css theme correctly applied. That's much nicer to see !

![Powershell docset on Velocity](/assets/posh-docset.PNG)


# Overengineering the whole process

I've set up a `_travis.yml` script that allow me to generate docsets on a new commit, as well as push them on my Github releases' repository. `Travis` is also set to regularly fire a build ("`cron` job") in order to keep a "up-to-date" build.

{% highlight html  %}
language: python
dist: precise
python:
- '3.6'

# Used to properly name build artifacts
env:
- ARTIFACT_NAME="posh-docsets-`git describe --tags`.zip"

deploy:
  provider: releases
  api_key:
    secure: WU9VIEFSRSBBIENIRUVLWSBMSVRUTEUgQkFTVEFSRCwgQUlOJ1QgWU9VID8=
  file: $ARTIFACT_NAME
  skip_cleanup: true  
  all_branches: true
  on:
    repo: lucasg/powershell-docset
    tags: true

addons:
  artifacts: true

install:
- pip install selenium requests bs4

script:
- python posh-to-dash.py --verbose --temporary --output=Powershell/versions/6/Powershell.tgz --version=6
- python posh-to-dash.py --verbose --temporary --output=Powershell/versions/5.1/Powershell.tgz --version=5.1
- python posh-to-dash.py --verbose --temporary --output=Powershell/versions/5.0/Powershell.tgz --version=5.0
- python posh-to-dash.py --verbose --temporary --output=Powershell/versions/4.0/Powershell.tgz --version=4.0
- python posh-to-dash.py --verbose --temporary --output=Powershell/versions/3.0/Powershell.tgz --version=3.0

- cp static/icon.png Powershell/icon.png
- cp static/icon@2x.png Powershell/icon@2x.png
- cp Powershell/versions/6/Powershell.tgz Powershell/Powershell.tgz

- zip -r $ARTIFACT_NAME Powershell
{% endhighlight html  %}


<br>
I even went as far as envisaging using [backstroke.us](http://backstroke.us) to monitor/sync `https://github.com/Powershell/PowerShell-Docs` and trigger a travis scraping job on a new commit in order to always have an up-to-date documentation. But on second though I think that's a bit overboard for a simple weekend side project.