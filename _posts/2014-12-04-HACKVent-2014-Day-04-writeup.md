---
layout: post
title: "HACKVent 2014 - Day 04 writeup"
date: 2014-12-04
---
<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the fourth challenge, which has nothing to do with dominoes.

<!--more-->

## Encoding Part :

- - - - - - -


For the Day 04 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 04](/assets/hackvent/04/riddle.png)

That's pretty straightforward substitution cipher, only with dominoes symbols instead of letters. First, we have to encode it into letters, using their order of appearance : 

<pre><code>ab cde fgbh ij
 cde kfabhl cd
e ibemeneh ogb
 ap qabr</code></pre>

To decode it, you will need an interactive tools where you can update the sustitution table and see immediately the decoded result. Personnally I used the <a href="www.cryptoclub.org/tools/cracksub_topframe.php"> tool provided by CryptoTools.</a> To break a substitution cipher, you need to guess where the right letters are, by using letter frequencies.

In English (I assume the message is in English), the most common letters are <code>'e'</code>, <code>'t'</code> and <code>'a'</code> so, by looking at the ciphertext, I can assume <code>'e'</code> is encoded either as <code>'b'</code>, <code>'e'</code> or <code>'a'</code>.

The most frequent digrams are <code>OF</code>, <code>TO</code>, <code>IN</code>, <code>IS</code>, and <code>IT</code>, so it's likely that <code>'ab'</code>, <code>'ij'</code> and <code>'ap'</code> are encoded versions of one of 5 former digrams. Again for the trigrams, the most common ones being <code>THE</code>, <code>AND</code>, <code>FOR</code>, <code>WAS</code>, and <code>HIS</code>.

The real breakthrough comes by looking at trigrams : there is a "cde" word which is repeated three times in the ciphertext. Moreover, two of the repetitions are enclosing a word - <code>"kfabhl"</code> - so it can only be <code>"AND"</code> or <code>"THE"</code>. After many trials and errors, here is the result :

![Result of the substitution](/assets/hackvent/04/substitution.png)

(The cryptoclub tool not supporting other characters than simple alphabet, I used <code>'x'</code> instead of <code>"-"</code>.)