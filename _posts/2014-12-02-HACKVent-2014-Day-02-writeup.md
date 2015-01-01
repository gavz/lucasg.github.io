---
layout: post
title: "HACKVent 2014 - Day 02 writeup"
date: 2014-12-02
---
<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the second challenge, which is focused on base64 encoding and internet time machines. 

<!--more-->

## Decoding Part :

- - - - - - -


For the Day 02 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 2](/assets/hackvent/02/riddle.png)

Having to write a base64 codec at work the previous week, I instantly recognized a base64-encoded string (or a base32 but that's far-fetched). The decoding gives the following url :

<pre> <code> http://hackvent.org/ </code> </pre>

![hackvent.org](/assets/hackvent/02/hackventorg.png)

The italicised word 'archive' and the fact that we have to find a 'time machine' are pretty good clues leading to the Wayback machine created by the Archive.org website, which is in fact an internet time machine.

## Time Rewind Part :

- - - - - - -

The wayback machine has recorded two versions of the webpage before December, on November 26th and 28th. The 28th version linked against the 'sorry.png' image that we've previously see, whereas the 26th version include a 'Ball_of-Santa.png'. Upon accessing www.hackvent.org/'Ball_of-Santa.png, we obtain the following image :

![QR ball for day 2](/assets/hackvent/02/d2.png)

That's it, this challenge is over :p
