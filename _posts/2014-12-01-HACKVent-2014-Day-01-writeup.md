---
layout: post
title: "HACKVent 2014 - Day 01 writeup"
date: 2014-12-01
---
<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the first challenge, which revolves around url shorteners. 

<!--more-->

## Investigation Part :

- - - - - - -

For the Day 1 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 1](/assets/hackvent/01/riddle.png)

The '@' character can only reference to a Twitter handle (I didn't found any other meaning). Since I'm not using Twitter, I spend several minutes to navigate in order to locate the page linked to the handle. On the profile, there is this message : 


![trim link from santa](/assets/hackvent/01/@hackvent.png)


the tr.im url shortener service being blocked at work (yep, I jerk around on my employers' time), I have to use a tool to un-shorten the url. My go-to site for this is <a href = 'www.urlquery.net' > www.urlquery.net </a> :


![Results from urlquery on redirection](/assets/hackvent/01/urlquery.png)


So we have 3 redirections :

* tr.im redirect to hackvent.hacking-lab.com/ch01.php
* ../ch01.php is a 302 redirection to bit.ly/1y04yV4
* bit.ly/1y04yV4 is a 301 redirection to hackvent.hacking-lab.com/images/tricked.png

The final page "tricked.png" smells bad. I wasn't wrong :

![Hackvent Day 01 tricksters](/assets/hackvent/01/old_tricked.png)

(The write-up has been done only several days after the "hacking" phase. Since then, there was additionnal info on the tricked ball.)

I'm out of luck since there is no information on the tricked image. I just know that I've hit a dead-end. I will spare the details of my exploration (cookies inspections, encoding scheme on urls, etc.) I learn that every url shortener service provides a tool to have the stats on a particular url. For example, you can append the '+' char to the tr.im and bit.ly url in order to know the stats :

![Bit.ly stat page for the tricked image](/assets/hackvent/01/bitly+.png)

On the stat page we can access the number of clicks and the registered people who shared it, as well as the creator 'hackvent'. On parcouring the hackvent profile page, we got this ball :

![QR ball for day 1](/assets/hackvent/01/d1.png)

That's actually the QR-encoded submit code for the first challenge !
