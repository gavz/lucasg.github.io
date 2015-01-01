---
layout: post
title: "HACKVent 2014 - Day 07 writeup"
date: 2014-12-07
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

This challenge has revealed to be one of the hardest I cracked, mainly because I had a lot of difficulties to identify the encryption mechanism and was misled several times. Since I was away from home the day it went live, I also solved it only the day after. 

Here's the write-up for the seventh challenge, which will put a considerable strain on your eyesight.

<!--more-->

## Exploratory Part :

- - - - - - -

For the Day 07 Hackvent challenge, we were given the following instructions and image :

![A lovely post-modernist depiction of the House of Atreus](/assets/hackvent/07/riddle.png)


At first glance, I know that somewhere in my head I've already seen this type of image but I can't say where or what is it. A image search also reveal nothing (to bad we can't search by 'patterns'). I don't know the stegano/encryption method, but it's obvious they are periodic patterns all over the image. It's also saying that somewhere I have to 'scan' something. Since I don't have the Ball-O-Matic, either the qr-ball is encoded in the image, or there will be an clue leading me to ball. 

So, I download the image for further investigations. That's when I got the most useful clue : 

![And as a fucktard, I decided not to use this clue for a whole half day](/assets/hackvent/07/clue.png)

Well it's not that obvious of a clue : 3D QR code doesn't exists (a part from embossed ones). My first lead was to see if there wasn't anything encoded in 'layers' (png file support layering) or in alpha-channels. There wasn't. I then try to look at <a href="http://en.wikipedia.org/wiki/Steganography"> steganography </a> using ImageMagick conversion function with several sizes and offset, only to draw a blank. Again.

That's when I had my worst idea : let's try to scan it, isn't it ?


## Lost in Translation Part :

- - - - - - -

Since the beginning of the Hackvent challenge, I used <a href = "www.onlinebarcodereader.com"> www.onlinebarcodereader.com </a> to scan the qr-ball and retreive the submission code. When feeding the 3DQR image, I got this : 

![Garbage in, garbage out](/assets/hackvent/07/scan_result.png)

Moreover, the result were coherent against different online scanners. Convinced that I was on the right tracks - I wasn't - I try to decipher the UPC_E barcode. "UPC" stands for Universal Product Code, which is an international barcode used to identify uniquely products. The "E" suffix stands for a compressed form in eight numbers with a leading zero. Problem : my code does not have a leading zero. I still try to run it against various incomplete product database (amazon's, etc.) but to no avail.

I start wondering if the online scanners could have wrongly identified the barcode type - Ockham should be turning over his grave - and found another short barcode, called EAN-8. The EAN-8 code consist of 7 id numbers and a checksum 8th one, so I verify the barcode and it check out ! Now I'm really sure I got the right code ...

![What's the chances, right ?](/assets/hackvent/07/checksum.png)

But, when I try to lookup the code in the database provided by GS - the company behind the barcode format - I realise that I've hit a deadend :

![Kinda like the "tricked" image in day01, only sadder](/assets/hackvent/07/lookup.png)

I have to change my angle if I want to solve this challenge.


## Psychedelism Part :

- - - - - - -

Now is the time where I finally use the hint hidden in the file name, and I start to look at QR 3D codes with steganography. Until I stumbled upon this website : 
<a href= "http://siqr.blogspot.fr/2014/04/siqr-codes-steganographically.html"> http://siqr.blogspot.fr/2014/04/siqr-codes-steganographically.html </a>. In this site the author explain how you can hide qr code in autostereograms - that's what the image really is - and how to decode it. Basically he tell me how to solve the challenge from A to Z. 

Beware of online autostereogram decoders, they can give you wrong results : 
![The magic eye was a little drunk this day](/assets/hackvent/07/canvas.png)

The decoding technique consist of creating a second layer identical to the image, and superpose the two layers in a difference mode then move the second layer to change the offset and hopefully reveal the qr code hidden. It really looked like <a href="http://www.datagenetics.com/blog/november32013/"> Visual Cryptography</a>, only with twice the same image.

![That was a major hassle to recover](/assets/hackvent/07/d07.png)
Once the qr-code is found and cleaned up, you can decode it to get your submission code

PS: To this day I still can't see autostereograms in 3D. I guess they don't work with idiots.

 