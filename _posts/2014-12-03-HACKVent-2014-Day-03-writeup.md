---
layout: post
title: "HACKVent 2014 - Day 03 writeup"
date: 2014-12-03
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the third challenge, which will refresh your historical knowledge of the Antiquity Era. 

<!--more-->

## Cipher Identification Part :

- - - - - - -


For the Day 03 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 03](/assets/hackvent/03/riddle.png)

We have an octagonal candle, what's look like to be an encrypted text and a subtitle pointing towards an ancient encryption method. This challenge is also the first one where you don't have to find directly the submit ball, but instead we have to enter a text into a validator (the "Ball-O-Matic"). This validator (as well as the "press F5 message") has confused the hell of many participants, myself included, because we weren't able to know whether we had the wrong input, or the right one but the validator wasn't responsive. For I, I sent directly POST requests with my solution to be certain that I was transmiting it.

I'm quite the beginner on ciphers, meaning I know the usual ones (caesar, vignere) but I don't have any clues for more "arcane" ones. That's why I tested almost every ciphers on <a href="http://rumkin.com/tools/cipher/"> Rumkin's page </a>. A lot of ciphers are easily discarded since they work only on a [a-z] charset, like atbash or rot13, and my ciphertext had also numbers.

The <a href = "http://rumkin.com/tools/cipher/railfence.php" > Railfence cipher </a> seemed interesting to me since you could interweave rubbish letters between your message. The ciphertext having 88=8x11 chars, I tried several line length ranging from 8 to 11 chars. The 9-char line version gives something which looked like a railfence code :

![Railfence 9](/assets/hackvent/03/railfence9.png)

So we got a  <code>"we wish you a merry christmas"</code> message , but there's a break at line 9 where you have to read the character just below, not in the right-bottom corner. You can also read "yours" and "key", but that's don't make much sense. Instead of reading diagonnally, I've switched to a 10-char line (and I've also modified the line before last to align the right characters) :

<pre><code>
WAIYTELZER
EMSOK3TEBZ
WETUE2EB2H
IRMRYGBUGK
SRASIIXGQ5
HYSESYWCZD
YCACBS43JI
OHNRAA3DA
URDES2DGO
</code></pre>

When reading vertically, we get <code>"we wish you a merry christmas and your secret key is base32 GIYSA2LTEBXW43DZEBUGC3DGEB2GQZJAORZHK5DI" </code> (beware when copying the hash not to make a mistake !). On decoding - it's a base32 hash - we get the following phrase :

<pre><code>21 is only half the truth</code></pre>

The geek in you  will recognize the reference. Anyway, when fed to the Ball-o-Matic you get the qr-ball.

PS : At that point, I must say I've wrongly identified the cipher used here, but fortunately it led me to the right solution. The encryption method is called a scytale - it's an octogonal cylinder - allegedly used by the Spartans during Antiquity to transmit secret orders on the battlefield.


