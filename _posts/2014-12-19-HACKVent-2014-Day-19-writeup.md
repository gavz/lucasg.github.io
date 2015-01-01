---
layout: post
title: "HACKVent 2014 - Day 19 writeup"
date: 2014-12-19
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

<!--more-->

## YYYYYYYYY Part :

- - - - - - -


For the Day 19 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 19](/assets/hackvent/19/riddle.png)

Again, there is no exploration part in this challenge : the image has a code hidden somewhere in it and we have to use the logical equation written to decode it.

For that matter, I've converted the steg bmp image into a serie of hex numbers and striped the header. Then I used the following routine :

{% highlight python %}
f = open("stegbool.str","rb").read().split(" ")

bmp_header = "424d363e0d0000000000360000002800000080020000c40100000100180000000000003e0d0000000000000000000000000000000000"
out = bmp_header

for i in range(0,len(f),3):

	r,g,b = map( lambda c: int(c, 16), f[i:i+3])
	r,g,b = r&1,g&1,b&1

	v = ((not r) & g^b)

	out += ( ("000000","ffffff")[v] )
	

print "".join(out).decode('hex')
{% endhighlight python %}

This is the result :

![ugly ugly ugly code](/assets/hackvent/19/img.png)

The qr code is not readable yet. We have to clean it up a little. I used a median filter with a 5px radius on it : 

<pre><code> (imagemagick's) convert.exe img.bmp -median 5 img_med5.bmp</code></pre>

![That's way better](/assets/hackvent/19/rgbstegbool_median5.png)