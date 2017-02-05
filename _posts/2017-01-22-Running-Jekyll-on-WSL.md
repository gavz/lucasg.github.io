---
layout: post
title: "Running Jekyll on WSL (Windows Subsystem for Linux)"
date: 2017-01-22
---

Since Windows is my daily driver at home, I'm genuinely interested by the arrival of `WSL` (Windows Subsystem on Linux) with the build 14393 last year. For me, the main use of `WSL` will be to run webservers locally (dns, apache, nodejs, etc.) without having to set up a whole VM (and configuring network) to run them.

I start by converting my local version of `Jekyll`, that I run on my PC in order to "debug" my posts before pushing them to github.

<!--more-->

For several years that I run this blog, whenever I'm want to write and correct a post while being on a Windows host, I had to use a "portable" version of Jekyll that I've gotten somewhere on the Internet (which embed `curl`, `webkit`, `git`, `python` and `ruby`) just to run a static blog. Not the most simple solution nor the most maintainable one, but it worked. You know the saying : "Broken get fixed, shitty stays that way".

Fortunately WSL is here and according to the Internet, it's possible to run Jekyll on it. Some people have done a tremendous job of describing the process of installing Jekyll on WSL (and the caveats of it) so I can't recommend them enough :
	
* <a href="https://www.richard-banks.org/2016/08/jekyll-on-bash-on-ubuntu-on-windows.html"> https://www.richard-banks.org/2016/08/jekyll-on-bash-on-ubuntu-on-windows.html</a>
* <a href="http://daverupert.com/2016/04/jekyll-on-windows-with-bash"> http://daverupert.com/2016/04/jekyll-on-windows-with-bash</a>


That being said, WSL would not be a full fledged Linux system without some dependency hell :p. <br>
I encountered severals issues:

* `zlib` not install by default. Type `sudo apt-get install zlib1g-dev` to resolve it
* `bundle` not install by default. Type `sudo gem install bundler` to resolve it
*  Jekyll need `ruby > 2.0`. See <a href="http://daverupert.com/2016/04/jekyll-on-windows-with-bash/#install-ruby">Dave Rupert's post</a> for the solution
*  Jekyll rely on `execjs` which need a Javascript runtime. `sudo gem install nodejs` do the trick.

The nastiest issue was the following one : 
{% highlight bash %}
lucasg@DESKTOP-822EGME:/mnt/lucasg.github.io$ jekyll serve --config=_config_dev.yml
WARN: Unresolved specs during Gem::Specification.reset:
      listen (< 3.1, ~> 3.0)
WARN: Clearing out unresolved specs.
Please report a bug if this causes problems.
/usr/lib/ruby/vendor_ruby/bundler/runtime.rb:33:in `block in setup': You have already activated json 2.0.2, but your Gemfile requires json 1.8.6. Using bundle exec may solve this. (Gem::LoadError)
        from /usr/lib/ruby/2.4.0/forwardable.rb:228:in `each'
        from /usr/lib/ruby/2.4.0/forwardable.rb:228:in `each'
        from /usr/lib/ruby/vendor_ruby/bundler/runtime.rb:19:in `setup'
        from /usr/lib/ruby/vendor_ruby/bundler.rb:120:in `setup'
        from /var/lib/gems/2.4.0/gems/jekyll-3.3.1/lib/jekyll/plugin_manager.rb:36:in `require_from_bundler'
        from /var/lib/gems/2.4.0/gems/jekyll-3.3.1/exe/jekyll:9:in `<top (required)>'
        from /usr/local/bin/jekyll:22:in `load'
        from /usr/local/bin/jekyll:22:in `<main>'
{% endhighlight bash %}

Apparently I had an old `Gemfile/Gemfile.lock` in the blog's root folder which provoked a conflict (system-wide `Ruby` relying on `json-2.0.2` while `activerecord` asking specifically for the `1.8.6` version). I deleted the old `GemFile`, <a href="https://help.github.com/articles/setting-up-your-github-pages-site-locally-with-jekyll/#step-2-install-jekyll-using-bundler"> copied the one from Github </a> and run `bundle install` in order to have a clean and updated `Gemfile`.

It works !

## Do you inotify ?

The number one issue with Jekyll on WSL is the lack of support for `inotify`, which means jekyll has to aggressively poll in order to update the modifications made on the source. While being inelegant, the poll method is fine with me : I just need to run the website locally therefore I can afford to waste some CPU and disk I/O.

Anyway, according to MS, this issue is resolved and the support is available for builds > 14942 : <a href="https://wpdev.uservoice.com/forums/266908-command-prompt-console-bash-on-ubuntu-on-windo/suggestions/13469097-support-for-filesystem-watchers-like-inotify"> https://wpdev.uservoice.com/forums/266908-command-prompt-console-bash-on-ubuntu-on-windo/suggestions/13469097-support-for-filesystem-watchers-like-inotify</a>. Which is actually a great news since `inotify` is some much more easier to use than `ReadDirectoryForChanges` when you need to watch a file or a directory.
