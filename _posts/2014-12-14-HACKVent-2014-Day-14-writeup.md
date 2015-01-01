---
layout: post
title: "HACKVent 2014 - Day 14 writeup"
date: 2014-12-14
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>


Here's the write-up for the challenge at day 14, in which we will rediscover the social inequalities through cracking. 

<!--more-->

## Exif through the crack shop Part :

- - - - - - -


For the Day 14 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 14](/assets/hackvent/14/riddle.png)

The file linked is a encrypted zip archive with a 128-bit AES password. When looking at the page's source code, you can notice this 'keyspace' tag :
{% highlight html %}
<p align="center"><img width="50%" src="images/Broken-cracker.jpg" keyspace="exif"></p>"
{% endhighlight python %}

I don't know what the keyspace tag is for, but I know damn well the 'exif' data is a hint. The challenge's author want us to look for <a href="http://en.wikipedia.org/wiki/Exchangeable_image_file_format"> EXIF </a> metadatas embedded in the broken cracker's image.

<pre>
Camera:	Canon PowerShot S80
Lens:	5.8 mm
(Max aperture f/2.8) (shot wide open)
Exposure:	Auto exposure, 1/8 sec, f/2.8
Flash:	Off, Red-eye reduction
XP Comment:	5 x lowerleet
Date:	January 27, 2006   3:31:52PM (timezone not specified)
(8 years, 10 months, 29 days, 13 hours, 52 minutes, 11 seconds ago, assuming image timezone of US Pacific)
File:	768 × 1,024 JPEG
133,280 bytes (130 kilobytes)
Color Encoding:	Embedded color profile: “sRGB”
</pre>

All the metadata are completely normal except for the 'XP comment' which is equal to "5 x lowerleet". Since the exif was linked under the tag 'keyspace' it means that the key needed to decrypt the zip archive is a 5 char password, all in lowercase and in leet langage so we have to look for every combination of lowercase alphanumeric char.


## Get rich or die cracking Part :

- - - - - - 

That's where the challenge became controversial : <a href="http://passwordrecoverytools.com/zip-password.asp"> this shareware software </a> coupled with a decent GPU can crack the zip file in a few minutes, while the free equivalent <a href="www.openwall.com/john"> John the Ripper </a> takes several hours at best, and is really bothersome to configure for OpenMP. People have complained to be forced to "buy" the software in order to solve the challenge in time, forgetting they were participating in a free event made by volunteers. Anyway, we will use the former software under its demo version, which gives only the two first characters, in order to concentrate the key search space and crack the three remaining chars with a regular bruteforce zip script.

After some time, we get the following key : **z1p

