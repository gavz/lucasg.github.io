---
layout: post
title: "HACKVent 2014 - Day 10 writeup"
date: 2014-12-10
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the tenth challenge, in which I debug SQL queries without knowing a thing about SQL. 

<!--more-->

## How I Learned to Stop Worrying and Love SQL Part :

- - - - - - -


For the Day 10 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 10](/assets/hackvent/10/riddle.png)

For this challenge, the authors didn't even gave a damn about obfuscating the way to the solution. It's in plain sight : use your sql-fu to get the answer.
<a href="http://sqlfiddle.com"> SQLfiddle.com</a> is a great site to test and run your sql database and queries. It also has a script to converse text-based table into proper SQL schema :


![Of course the sql query don't fucking run ...](/assets/hackvent/10/sqlfiddle.png)

We got a resulting code with 5 texts separated by dashes. But the submission code (the one encoded in qr xmas ball) has the following format : <code>HV14-xxxx-xxxx-xxxx-xxxx-xxxx</code>. We have to tweak the query in order to get the valid code.

In other words we have to transform the <code>&1</code> into <code>HV14</code>. the query used to get the first part is the following :
<pre><code> translate('&1','abcdefghijklmnopqrstuvwxyz','MEZ4VPG5NS6RYH7CT1QW2FO3AI')</code></pre>

It's a simple substitution table, so you have to enter <code>nerd</code> in order to get <code>HV14</code>. We obtain the next code :

<pre><code>HV14-3212-7553-ED2D-1597-6897</code></pre>

However, it's not a valid code, so let's take another look at the whole query. The thing is, in SQl, the <code>&1</code> char also mean it's the first input argument of a progrem (kinda like <code>$1</code> in bash or <code>sys.argv[1]</code> in python) so we have to replace every occurences of <code>&1</code> with <code>nerd</code>. The resulting code is valid :

<pre><code>HV14-3212-7553-BCF8-1597-6897</code></pre>

That was easier than I anticipated.



