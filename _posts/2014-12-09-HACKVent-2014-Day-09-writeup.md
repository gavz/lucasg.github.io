---
layout: post
title: "HACKVent 2014 - Day 09 writeup"
date: 2014-12-09
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the ninth challenge, in which we will learn how to spy on our girlfriend by reading her text messages. 

<!--more-->

## iPhorensics Part :

- - - - - - -


For the Day 09 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 09](/assets/hackvent/09/riddle.png)


I dont't know what the file consist of, so I run <code>strings</code> on it and I directly get <code>SQLite format 3</code>. So it's a SQLite Database. I could have installed a database browser, but it wasn't needed it since every info was here : 

<pre>
<code>
__kIMMessagePartAttributeName
NSNu
7EA9C4B7-DC8D-41FE-9577-DA4EE85C3E15==Nn0EUp68lYbS2LeMKMhEaYbS2Leyzoa1PouWzYw9JoiRQJFS3qT10IIuxY3Szq
streamtyped
NSMutableAttributedString
NSAttributedString
NSObject
NSMutableString
NSString
p sprl jhlzhy zhshk
NSDictionary
__kIMMessagePartAttributeName
NSNumber
NSValue
SMSe:AFF414A2-007E-4753-B7B1-395C0BFA3ADB
7EA9C4B7-DC8D-41FE-9577-DA4EE85C3E15
chat
message
handle
USMS;-;+41796666666-
AFF414A2-007E-4753-B7B1-395C0BFA3ADBbplist00
CKChatWatermarkMessageID_
CKChatWatermarkTime
I+41796666666SMSE:2FE6EFB3-EA2E-43D3-88D3-504515DECECE
USMS;-;+4179666ROT13-
AFF414A2-007E-4753-B7B1-395C0BFA3ADBbplist00
CKChatWatermarkMessageID_
CKChatWatermarkTime
I+4179666ROT13SMSE:2FE6EFB3-EA2E-43D3-88D3-504515DECECE
SMS;-;+41796666666
SMS;-;+4179666ROT13
!+41796666666chSMS0796666666
#+4179666ROT13chSMS079666ROT13
+41796666666SMS
+4179666ROT13SMS
+41796666666
+4179666ROT13
</code>
</pre>

It's a SMS sent by <code>+4179666ROT13</code> to <code>+41796666666</code> containing the text <code>==Nn0EUp68lYbS2LeMKMhEaYbS2Leyzoa1PouWzYw9JoiRQJFS3qT10IIuxY3Szq</code>.

At first I was looking for GSM PDU encodings - in fact Iphone text messages are plaintext stored - to try to make sense with the sms payload. <a href="http://stackoverflow.com/a/21110449"> While looking at encoding examples </a>, I get the idea to reverse the string : 

<code>qzS3YxuII01Tq3SFJQRioJ9wYzWuoP1aozyeL2SbYaEhMKMeL2SbYl86pUE0nN==</code>.

It obviously is a base64 string, the two ending "=" characters being a padding. However it does not translate into a proper ascii text. That's where another hint comes to play : <code>+4179666ROT13</code>. This cell phone number is not valid, and it tells me somewhere a ROT13 encryption has been applied. By rot13'ing the string and then base64-decoding it we get : 

<pre><code>vaw.HUWMFwqRX1/moc.bal-gnikcah.tnevkcah//:ptth</code></pre>
<pre><code>http://hackvent.hacking-lab.com/1XRqwFMWUH.wav</code></pre>



## Two Tone Army Part :

- - - - - - -


The wav file is a serie of dial tones - also known as <a href="http://en.wikipedia.org/wiki/Dual-tone_multi-frequency_signaling"> Dual tone multifrequency dialing (DTMF) </a>, from the times where phones had sounds when pressing the numbers to dial. Decoding a dial audio sequence was a movie trope for hackers.

<a href="http://www.teworks.com/dtmf.htm"> This executable</a>, once the wav file is converted to a 44110 Hz PCM signal can automatically decode the dtmf sequence and output this number sequence :
<pre><col>66#97#122#33#110#103#97</col></pre>

That's ASCII characters separated by a "sharp" symbol. When translating into their representation, we get <code>Baz!nga</code>. 
