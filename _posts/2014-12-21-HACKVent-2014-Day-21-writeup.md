---
layout: post
title: "HACKVent 2014 - Day 21 writeup"
date: 2014-12-21
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the challenge at day 21, in which we will learn how to get banned from casino and bingo parties.

<!--more-->

## Ocean's Se7en Part :

- - - - - - -


For the Day 21 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 21](/assets/hackvent/21/riddle.png)

In this challenge, we have to "break" the linear congruental number generator in order to predict what's the next number will be and place our bets accordingly. 

The linear congruent generator takes a seed (here a 31 bit one) and each time it multiply the current state (the first one being the seed) by a multiplier, add an increment and takes the congruent to obtain the next state. Then from the new state, it takes the congruent modulo 100 to get the ouptut value. 

From the attacker's view, we only "see" a part of the states (the bits below 2^6=128) but we know that each output has to follow the constraints on the states. Using a sequence of outputs (6-7 numbers) we might be able to bruteforce the next output.


{% highlight python %}
def init_worker(procnum, rang, output_array,  return_list):

    ret = [ i+output_array[0] for i in ifilter(lambda s: lcg(s+output_array[0]) % 100 == output_array[1], rang )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(s)) % 100 == output_array[2], ret )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(lcg(s))) % 100 == output_array[3], ret )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(lcg(lcg(s)))) % 100 == output_array[4], ret )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(lcg(lcg(lcg(s))))) % 100 == output_array[5], ret )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(lcg(lcg(lcg(lcg(s)))))) % 100 == output_array[6], ret )]

    return_list += ret
    print str(procnum), len(ret)
{% endhighlight python %}
  
This python function, which is ugly as hell, given an array of previous outputs (6 of it) and a range of values 'rang', can bruteforce all the possible values for the state of output_array[0].


{% highlight python %}
def bruteforce_lcg(output_array):
  p = Pool(8)
  

  manager = Manager()
  return_list = manager.list()
  jobs = []

  for i in range(8):
    r = xrange(100*((i*modulus/8)/100),100*(((i+1)*modulus/8)/100),100)
    print r

    p = Process(target=init_worker, args=(i,r,output_array, return_list))
    jobs.append(p)
    p.start()

  for proc in jobs:
    proc.join()


  return return_list
{% endhighlight python %}

As for <a href="/post/2014/12/17/HACKVent-2014-Day-17-writeup/"> Challenge 17 </a> it's recommended to use a multithreaded environment to run the bruteforce script. Now that we can predict an output, it's easy to lose small amounts in order to collect series of outputs and go all in when we know for sure the next number. The following routine gives a unique number - most of the times - if we gives an 7-number following sequence of outputs :


{% highlight python %}
from itertools import islice, ifilter, repeat
from multiprocessing import Pool,Manager,Process
import numpy as np
import sys

modulus = 2**31
multiplier = 1664525
increment = 1013904223

def lcg(seed):
  new_seed = (multiplier*seed + increment) % modulus
  return new_seed

def lcg_array(seed, output_count):

  def generator(seed):
    i = seed
    while True:
      i =  lcg(i)
      yield i % 100

  return list(islice(generator(seed),output_count))
  
def init_worker(procnum, rang, output_array,  return_list):
    
    ret = [ i+output_array[0] for i in ifilter(lambda s: lcg(s+output_array[0]) % 100 == output_array[1], rang )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(s)) % 100 == output_array[2], ret )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(lcg(s))) % 100 == output_array[3], ret )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(lcg(lcg(s)))) % 100 == output_array[4], ret )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(lcg(lcg(lcg(s))))) % 100 == output_array[5], ret )]
    ret = [ i for i in ifilter(lambda s: lcg(lcg(lcg(lcg(lcg(lcg(s)))))) % 100 == output_array[6], ret )]
    

    return_list += ret
    print str(procnum), len(ret)
    

def bruteforce_lcg(output_array):
  p = Pool(8)
  

  manager = Manager()
  return_list = manager.list()
  jobs = []

  for i in range(8):
    r = xrange(100*((i*modulus/8)/100),100*(((i+1)*modulus/8)/100),100)
    print r

    p = Process(target=init_worker, args=(i,r,output_array, return_list))
    jobs.append(p)
    p.start()

  for proc in jobs:
    proc.join()

  return return_list




if __name__ == "__main__":

  # arr = [7,98,93,52,51,38,85]
  arr = [ int(i) for i in sys.argv[1:8]]  
  print arr

  s = bruteforce_lcg(arr)

  print lcg_array(int(s[0]),7)

{% endhighlight python %}


Once we get up to 10000$, the qr-ball appears: 
![qr ball for rich people](/assets/hackvent/21/GFHosP0AU8qRxygypqFR.png)