---
layout: post
title: "HACKVent 2014 - Day 05 writeup"
date: 2014-12-05
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the fifth challenge, in which we will compare apples and oranges.

<!--more-->

- - - - - - -

For the Day 05 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 05](/assets/hackvent/05/riddle.png)

I'm not an English native person, so I don't know half of the idiomatic expressions used here like "baker's dozen" or "donkeypower". Like any other human being, I google'd them. To my surprise, Google directly returns the numerical value in the calculator :

![Bakers like to brag](/assets/hackvent/05/baker_dozen.png)


Results from google calc : 

* megasecond : <code> 10e6 s </code>
* baker dozen : <code> 13 </code>
* donkeypower : <code> 250.033167 W </code>
* number of horns on a unicorn : <code> 1 </code>
* once in a blue moon : <code> 1.16699016 10e-8 Hz </code>
* answer to life the universe and everything : <code> 42 </code>
* a beard second : <code> 5 nm </code>
* earth mass : <code> 5.97219 10e24 kg </code>


Once I got all this number, I didn't feel like I was getting progress. I got a hint previously from the author of the challenge saying that the result is a integer and all I got was floating-point values due to earth mass and once in a blue moon being floats. Moreover, dimensionnal analysis wasn't concluant : I think the resulting unit is in N.s-2*, which mean nothing to me. 

The breakthrough came when I directly typed the following phrase into google : 

![A rubbish computation](/assets/hackvent/05/baker_sec_sq_earth_mass.png)

Google translated a floating operation into a integer ! From then on, I knew I just had to punch the whole equation into Gooogle's search bar. You can use brackets "[ ]" to encapsulate sub-operations :

<pre><code> [number of horns on a unicorn once in a blue moon]/[answer to life the universe and everything][bakers dozen donkeypower]/[a beard second squared earth mass]*[(half a megasecond) squared squared]</code></pre>



![a completely non-sensical result](/assets/hackvent/05/final_result.png)