---
layout: post
title: "HACKVent 2014 - Day 08 writeup"
date: 2014-12-08
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the first "medium" challenge at day 8, which consist of running potentially malicious arbitrary code on your machine. 

<!--more-->

## Pearl Threading Part :

- - - - - - -


For the Day 08 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 08](/assets/hackvent/08/riddle.png)

The candle-like code look a lot to me like this one : 

{% highlight bash %}
:(){ :|:& };:
{% endhighlight bash %}

Beware, this is a fork bomb ! This string has historically been sent to as answers to real-world linux problems, as a prank in order to educate linux newbies about the dangers of running arbitrary code from a Internet stranger. For information, this prgram replicates itself until it exhausts the host's resources (ram, processes, handles, etc. ). A real nasty piece of software.

So what's I am gonna do about the unkknown candle program ? run in bash of course ! 
Well I didn't corrupt my computer, since it didn't really run : an error was detected. That's when you have to use a clue in the title "a PEARL white candle". Okay, this hint is as subtle as a hooker's makeup. You have to run the code with Perl.


{% highlight perl %}
perl pearl.pl 
Eval-group not allowed at runtime, use re 'eval' in regex m/(?{eval"\$a=<>;chomp \$a;\$a=~tr/A-Z a-z/\"-;N-ZA-M/;((\$a eq '0BZMNDSFZNQOBNDOFGSN1SFZ!')&&(print \"right\\n\"));
"})/ at pearl.pl line 42.
{% endhighlight perl %}

So, this program doesn't work either, but I don't care since I got the body of the code, in human-readable format. From what I read, I assume I need $a to be equal to "0BZMNDSFZNQOBNDOFGSN1SFZ!" at the end in order for the script to print "right". I guess the corresponding $a input should be the winning text used to get the validation qr-ball.

the <code>chomp</code> function is only here to strip any return char in the input, the major transformation is done in the <code>$a=~tr/A-Z a-z/\"-;N-ZA-M/;</code> part. The tr function in Perl is used to translate a charset into another one, using relative position of char as table. For exemple <code>tr/a-e/1-7/</code> transform the text "deadbeef" into "4514255f". At first I tried to find the reverse table to invert the tr function, but I quicky said fuck it and did it by hand :


![I don't know my alphabet](/assets/hackvent/08/answer.png)


As you may read, the right answer was : <code>Only perl can parse Perl!</code>. Which is right.

