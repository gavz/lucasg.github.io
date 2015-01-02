---
layout: post
title: "HACKVent 2014 - Day 13 writeup"
date: 2014-12-13
---


<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>



Here's the write-up for the challenge at day 13, in which we will talk to extraterrestrial beings. 

<!--more-->

## Ground Control to Major Tom Part :

- - - - - - -


For the Day 13 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 13](/assets/hackvent/13/riddle.png)

The alien DJ image is a regular png file. Unlike in <a href="2014-12-11-HACKVent-2014-Day-11-writeup.md"> challenge 11 </a>, looking at the file in hex mode doesn't reveal anything. However, you can notice a weird 1-px line at the bottom of the image : 

![this is not a pipe](/assets/hackvent/13/d13_line.png)

I initially thought that was some morse encoded data, but it turns out not to be true (the only combinations that are valid outputs random strings). For a moment, I also though is was a barcode. <a href="2014-12-07-HACKVent-2014-Day-07-writeup.md"> Challenge 07 proved me previously that barcodes were a bad idea </a>. There is something I haven't tried : convert b/w pixels to binary values. Who knows, maybe aliens also use 8-bit words PCM data streams ?


<pre><code>01101000 01110100 01110100 01110000 00111010 00101111 00101111 01101000 01100001 01100011 01101011 01110110 01100101 01101110 01110100 00101110 01101000 01100001 01100011 01101011 01101001 01101110 01100111 00101101 01101100 01100001 01100010 00101110 01100011 01101111 01101101 00101111 00110101 01110110 01010000 01001011 01000111 01011001 00110000 01110100 01100100 01011000 00101110 01101101 01110000 00110011

=> http://hackvent.hacking-lab.com/5vPKGY0tdX.mp3
</code></pre>

## Absolute Hearing Part :

- - - - - - -

<a href="2014-12-09-HACKVent-2014-Day-09-writeup.md"> Like in Challenge 09 </a>, we receive a link to an audio file, which is a succesion of tones. Unlike challenge 9, it's not DTMF encoded data :
<a href="assets/hackvent/13/5vPKGY0tdX.mp3"> 5vPKGY0tdX.mp3 </a> 

![Riddle from hackvent.hacking-lab.com for Day 13](/assets/hackvent/13/tone-spectrum.png)


When looking at a slowed down version, you can hear there is in fact only two differents tones (E6 and E7) :
<a href="assets/hackvent/13/5vPKGY0tdX-slowed-down.mp3"> 5vPKGY0tdX-slowed-down.mp3 </a>

It's again PCM encoded data. Here a simple python script I used to automatically recognize the tone fundamental and convert it into an array of 1's and 0's :

{% highlight python %}
from pylab import*
from scipy.io import wavfile
import numpy as np 


sampFreq, snd = wavfile.read('wav5vPKGY0tdX.wav')
signal = snd[:,0] 

tones_count = int(snd.shape[0]*10.0/(sampFreq*1.0))

for i in range(0,tones_count):
  s = signal[i/10.0*sampFreq : (i+1)*sampFreq/10.0]
  fftData=abs(np.fft.rfft(s))**2  
  
  print i, fftData[1:].argmax() / 200  
{% endhighlight python %}

<pre><code> 011000010110110001101001011001010110111001101000011000010110001101101011011000010111010001110100011000010110001101101011

=> alienhackattack
</code></pre>
