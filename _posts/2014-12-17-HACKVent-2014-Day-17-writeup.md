---
layout: post
title: "HACKVent 2014 - Day 17 writeup"
date: 2014-12-17
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the challenge at day 17, in which we will crush handshake.


<!--more-->

## RZA distant cousin Part :

- - - - - - -


For the Day 17 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 17](/assets/hackvent/17/riddle.png)

The instructions are pretty straightforward : we have to break RSA encryption ! Fortunately, the chosen primes number are ridiculously low. The difficulty in this challenge is to automate the process in order to relieve ourselves from a really boring task.

First we have to find the two primes numbers which forms n, which means we have to factor n :

{% highlight python %}
def factors(n):    
      return set(reduce(list.__add__, 
                  ([i, n//i] for i in range(1, int(n**0.5) + 1) if n % i == 0)))
fact = [ f for f in itertools.ifilter( lambda i : i != 1 and i != int(n), factors(int(n)) )]
{% endhighlight python %}

It's not really readable, but it's a little python function which test all integer between 1 and root square of n. It's quite fast actually.

Now that we have p and q, we can compute <code>phi = (p-1)*(q-1)</code> which is the modulus used to compute d. d, the decryption key, is the modular inverse of e in N/phiN :

{% highlight python %}
def egcd(a, b):
    if a == 0:
        return (b, 0, 1)
    else:
        g, y, x = egcd(b % a, a)
        return (g, x - (b // a) * y, y)

def modinv(a, m):
    g, x, y = egcd(a, m)
    if g != 1:
        #raise Exception('modular inverse does not exist')
        return 1
    else:
        return x % m

phi = (fact[0] - 1)*(fact[1]-1)
d = modinv(int(e), phi)
{% endhighlight python %}

Using the number d, we get decrypt the ciphertext c using the following equation : <code> dec = (int(c)**d) % (n) </code>

Since there is 178 public keys to break and the task is highly parallelizable, it's recommended to use threads or any other mutliprocessing tool (or even better, a GPU based routine). Don't forget to output the worker's index since the decoded texts won't be in order.

{% highlight python %}
import sys
import itertools
from multiprocessing import Pool

def decode( inec):

  i,n,e,c = inec

  def egcd(a, b):
      if a == 0:
          return (b, 0, 1)
      else:
          g, y, x = egcd(b % a, a)
          return (g, x - (b // a) * y, y)

  def modinv(a, m):
      g, x, y = egcd(a, m)
      if g != 1:
          #raise Exception('modular inverse does not exist')
          return 1
      else:
          return x % m

  def factors(n):    
      return set(reduce(list.__add__, 
                  ([i, n//i] for i in range(1, int(n**0.5) + 1) if n % i == 0)))


   fact = [ f for f in itertools.ifilter( lambda i : i != 1 and i != int(n), factors(int(n)) )]
   #print fact

   phi = (fact[0] - 1)*(fact[1]-1)

   d = modinv(int(e), phi)

   
   print str(i).zfill(5), ":", int(c), int(n), int(e), fact, phi, d, "->", (int(c)**d) % (fact[0]*fact[1])




if __name__ == '__main__':
  p = Pool(178)

  idx = range(178)
  nfile = open("n","r").readlines()
  efile = open("e","r").readlines()
  cfile = open("c","r").readlines()
  
  print "idx, cipher, n, e, fact, phi, d, plaintext"

  p.map(decode, zip(idx,nfile,efile,cfile))
{% endhighlight python %}

This python script is quite efficient : 

![Contributing to global warming, one CPU at a time](/assets/hackvent/17/proc.png)

After some formating and ascii conversion, we get the following text : 

<pre><code>God could create the world in six days because he didn't have to make it compatible with the previous version. If we're supposed to work in Hex, why have we only got 0xA fingers?</code></pre>