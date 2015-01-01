---
layout: post
title: "HACKVent 2014 - Day 20 writeup"
date: 2014-12-20
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the twentieth challenge, in which we will constantly be called a "hobo".

<!--more-->

## Stop ! Hammer Time Part :

- - - - - - -

For the Day 20 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 20](/assets/hackvent/20/riddle.png)

The url links to this website : 

![Basic math calculations, isn't it ?](/assets/hackvent/20/riddler_website.png)

It's quite the minimalist page, where there is only a mathematical equation embedded in a gif image - the equation being randomly selected at every new request - and two very interesting headers in the GET response : 

<pre><code>X-Riddler : "INFO"
X-Riddler-Howto :"I will reward you with a nice christmas ball if you solve 30 of my riddles in just a minute. Just send me back my cookies and POST your answer as 'result'. However, if one of your answers is wrong, you'll have to start over."
</code></pre>

Fortunately, I got this hint quite early in the game (like about 15 mins since I've begun) but I know it has stumped other contestants. Along the headers , there is also some cookies : 

<pre><code>HACKVent_Ticket : "edb20784c6810ed9aef2f1ec0031f071"
HACKVent_User   : "bHVjYXNn"
PHPSESSID : "epflifvtjn8p1v9ghsnt2i3dt2"
</code></pre>

The instructions are quite clear : I have to "solve" one way or another 30 equations under a minute, and send a POST request for every answer. There is still two hurdles : how am I gonna solve the 30 equations, and how I have to send back the answer.

One equation to solve every two seconds seemed difficult to do, knowing that there is at least 300ms of latency between the POST answer and the reception of the next challenge. Which leave less about 1.5 sec to crack the equation automatically. That's why I initially though there were some kind of <a href="http://en.wikipedia.org/wiki/Session_fixation"> session fixation </a> where you could have an impact on the <code>SESSID</code> cookie and always getting the same string of equations. Kinda like when you set the seed for a pseudo-random generator. I try to set the <code>PHPSESSID</code> value but to no avail. So instead I tried to solve the equations automatically.


##  Part :

- - - - - - -




##  Part :

- - - - - - -

