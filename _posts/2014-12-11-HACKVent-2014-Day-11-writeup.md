---
layout: post
title: "HACKVent 2014 - Day 11 writeup"
date: 2014-12-11
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>


HERE_S_THE_WRITE-UP_FOR_THE_ELEVENTH_CHALLENGE__IN_WHICH_WE_WILL~1. 

<!--more-->



## Hex-Ray vision Part :

- - - - - - -


For the Day 11 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 11](/assets/hackvent/11/riddle.png)


The "Good Old Times" title suggest some kind of nostalgia among hackers/dev, so I have to look for an ancient something, whether it' an OS, a filesystem, a processor, a programing language, etc. There is also a .exe file to be downloaded. When run, it output a popup :

![What's the author's problem with being late ?](/assets/hackvent/11/too_recent.png)

Since I can't run it natively (WinXp/Win7/Win8.1) I might as look at it's innards. It's always useful to look at file signature in order to see if a file wouldn't lie about it's real nature. <a href="http://en.wikipedia.org/wiki/List_of_file_signatures"> You can find here a list of the most common magic strings </a>. A Windows executable file should begin by the magic string "PE" - for portable executable.


![Download this software to look under the skirts of executable files !!!](/assets/hackvent/11/xray.png)

Before the PE header, there is another one : 'MZ', the magic string for DOS-executable. There is also another readable string present: "Keep on trying till you run out of cake!". Let's fire up the DOSBox.


## Only 90's kids will recognize this OS Part :

- - - - - - -

![GOOOOOOOOOOODOL~1](/assets/hackvent/11/dosbox.png)

I've nothing much to say :  I got directly the solution, which is <code>I'm Still Alive</code>
