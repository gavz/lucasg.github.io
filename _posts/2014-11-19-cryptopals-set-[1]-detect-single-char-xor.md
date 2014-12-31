---
layout: post
title: "Cryptopals - Set [1] - Detect single-character XOR"
date: 2014-11-19
---

Not a long time ago, I started to give myself into the Cryptopals Matasano challenge. Knowing little about cryptographics, I thought that
was a good idea to learn a thing or two about encryption, as well as brushing off my C skills (I'm currently a C++ guy).

<!--more-->

The first set of exercises, which use a qualifing set of basics challenges, is pretty easy to solve (though it's not a matter of minutes as they
say if you really want properly implement the solution). However the first hurdle is located in the fourth exercise : "Detect single-character XOR".

In this challenge, you're asked to find the line which has been encrypted using caesar cipher. This exercise combine the three previous exercise - which are resp. hex string manipulation, caesar cipher implementation and caesar cipher breaking - into a single challenge, which is a good way to validate your previous code running against a "real" problem. For example, I've struggled a longer time than I would admit to crack the exercise, because I didn't know whether the input string were hex encoded data or simple char symbols array : they didn't had any letter above <code>'f'</code>, indicating a hex encoded data, but they also included a <code>'\n'</code> ! Finally I had to write down a script to decrypt each string using every ascii char possible - roughly 327*127 differents outputs - in order to brute force the problem and then trace my way back to find the bugs in my solver.


Bruteforce decoding:
------


As a said previously, I mainly implemented the solvers in C, but for the bruteforce script I chose Python, because that's where the language really does shine :

{% highlight python %}
d = open(sys.argv[1], "rb").readlines()

for i,line in enumerate(d):
  hexline = line[:-1].decode("hex")
  print "\n", i,": hexline ->", hexline

  for c in range(127):
    print c," :" , chr(c), " -> ", ''.join([ chr(c ^ ord(x)) for x in hexline]) 
{% endhighlight python %}

Not having to deal with dynamic buffer allocation is a cool breeze compared to C. C has many advantages, but string manipulation is definitively not
one of them. Anyway, caesar encryption breaking is a well-known subject since it's vulnerable to frequency analysis (using letters, digrams and so on) so the challenge is quite straightforward.

Next : to the sixth exercice which is vigenere cipher breaking, which surely be harder.