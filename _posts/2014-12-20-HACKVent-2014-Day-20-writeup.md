---
layout: post
title: "HACKVent 2014 - Day 20 writeup"
date: 2014-12-20
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the twentieth challenge, in which we will constantly be called a "hobo".

<!--more-->

## Stop ! Hammer Time Part :

- - - - - - -

For the Day 20 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 20](/assets/hackvent/20/riddle.png)

The url links to this website : 

![Basic math calculations, isn't it ?](/assets/hackvent/20/riddler_website.png)

It's quite the minimalist page, where there is only a mathematical equation embedded in a gif image - the equation being randomly selected at every new request - and two very interesting headers in the GET response : 

<pre><code>X-Riddler : "INFO"
X-Riddler-Howto :"I will reward you with a nice christmas ball if you solve 30 of my riddles in just a minute. Just send me back my cookies and POST your answer as 'result'. However, if one of your answers is wrong, you'll have to start over."
</code></pre>

Fortunately, I got this hint quite early in the game (like about 15 mins since I've begun) but I know it has stumped other contestants. Along the headers , there is also some cookies : 

<pre><code>HACKVent_Ticket : "edb20784c6810ed9aef2f1ec0031f071"
HACKVent_User   : "bHVjYXNn"
PHPSESSID : "epflifvtjn8p1v9ghsnt2i3dt2"
</code></pre>

The instructions are quite clear : I have to "solve" one way or another 30 equations under a minute, and send a POST request for every answer. There is still two hurdles : how am I gonna solve the 30 equations, and how I have to send back the answer.

One equation to solve every two seconds seemed difficult to do, knowing that there is at least 300ms of latency between the POST answer and the reception of the next challenge. Which leave less about 1.5 sec to crack the equation automatically. That's why I initially though there were some kind of <a href="http://en.wikipedia.org/wiki/Session_fixation"> session fixation </a> where you could have an impact on the <code>SESSID</code> cookie and always getting the same string of equations. Kinda like when you set the seed for a pseudo-random generator. I try to set the <code>PHPSESSID</code> value but to no avail. So instead I tried to solve the equations automatically.


## Can you read it ? Part :

- - - - - - -

The first thing to check is to be sure to send back the cookies every time we want to send an answer. For that matter, <code>curl</code> has some pretty nice built-ins options.

For the initial download :
<pre>
<code>curl -c cookies.txt "http://212.254.178.162/1JnjqflWseX_29Ow-YXH/" -o 1JnjqflWseX_29Ow-YXH1.gif</code>
</pre>

* <code> -c cookies.txt</code> can save the current cookies in the provided file
* http://hackvent.hacking-lab.com/1JnjqflWseX_29Ow-YXH is a redirection, so instead of tell curl to follow redirs, I rather used the IP 212.254.178.162.
*  <code> -o 1JnjqflWseX_29Ow-YXH1.gif  </code> : save the gif file
 
For the answer :
<pre>
<code>curl -X POST -b cookies.txt --data "result="$payload "http://212.254.178.162/1JnjqflWseX_29Ow-YXH/" -o 1JnjqflWseX_29Ow-YXH2.gif
</code>
</pre>

* <code>-X POST</code> tells curl to send the request as POST. 
* <code>-b cookies.txt</code> loads the cookies and send them with the message. 
* <code>--data "result="$payload.</code> This option add a result field in the sent message, telling the riddle what is our answer. I spent way too long trying to find this. 
*  <code> -o 1JnjqflWseX_29Ow-YXH2.gif  </code> : save the next equation (if our result is correct).


Now that we've solved the communication part, we need to be able to detect and solve correctly the equations given by the riddler. The only command-line OCR tool which is free is called <a href="https://code.google.com/p/tesseract-ocr/">tesseract</a>, and I was impressed by the result. The only alternative to detect the symbol would be to build a database of letters and operators, and run a least-square estimator on the pixels difference to retrieve what could be the equation. Being lazy, I went with the external tool solution, even if I think the latter should work quite reliably.

Once installed, you can call tesseract this way :
<pre><code>tesseract equation.gif equation_output</code></pre>

The decoded text is then saved into <code>equation_output.txt</code>. After the equation decoding, you also need a numerical evaluator, one you can tweak a bit in order to correct the errors made in the OCR part. I used <a href="http://stackoverflow.com/a/2371789/1741450"> this python's numerical parser found in a StackOverflow answer</a> :

{% highlight python %}
from __future__ import division
from pyparsing import (Literal,CaselessLiteral,Word,Combine,Group,Optional,
                       ZeroOrMore,Forward,nums,alphas,oneOf)
import math
import operator

__author__='Paul McGuire'
__version__ = '$Revision: 0.0 $'
__date__ = '$Date: 2009-03-20 $'
__source__='''http://pyparsing.wikispaces.com/file/view/fourFn.py
http://pyparsing.wikispaces.com/message/view/home/15549426
'''
__note__='''
All I've done is rewrap Paul McGuire's fourFn.py as a class, so I can use it
more easily in other places.
'''

class NumericStringParser(object):
    '''
    Most of this code comes from the fourFn.py pyparsing example

    '''
    def pushFirst(self, strg, loc, toks ):
        self.exprStack.append( toks[0] )
    def pushUMinus(self, strg, loc, toks ):
        if toks and toks[0]=='-': 
            self.exprStack.append( 'unary -' )
    def __init__(self):
        """
        expop   :: '^'
        multop  :: '*' | '/'
        addop   :: '+' | '-'
        integer :: ['+' | '-'] '0'..'9'+
        atom    :: PI | E | real | fn '(' expr ')' | '(' expr ')'
        factor  :: atom [ expop factor ]*
        term    :: factor [ multop factor ]*
        expr    :: term [ addop term ]*
        """
        point = Literal( "." )
        e     = CaselessLiteral( "E" )
        fnumber = Combine( Word( "+-"+nums, nums ) + 
                           Optional( point + Optional( Word( nums ) ) ) +
                           Optional( e + Word( "+-"+nums, nums ) ) )
        ident = Word(alphas, alphas+nums+"_$")       
        plus  = Literal( "+" )
        minus = Literal( "-" )
        mult  = Literal( "*" )
        div   = Literal( "/" )
        lpar  = Literal( "(" ).suppress()
        rpar  = Literal( ")" ).suppress()
        addop  = plus | minus
        multop = mult | div
        expop = Literal( "^" )
        pi    = CaselessLiteral( "PI" )
        expr = Forward()
        atom = ((Optional(oneOf("- +")) +
                 (pi|e|fnumber|ident+lpar+expr+rpar).setParseAction(self.pushFirst))
                | Optional(oneOf("- +")) + Group(lpar+expr+rpar)
                ).setParseAction(self.pushUMinus)       
        # by defining exponentiation as "atom [ ^ factor ]..." instead of 
        # "atom [ ^ atom ]...", we get right-to-left exponents, instead of left-to-right
        # that is, 2^3^2 = 2^(3^2), not (2^3)^2.
        factor = Forward()
        factor << atom + ZeroOrMore( ( expop + factor ).setParseAction( self.pushFirst ) )
        term = factor + ZeroOrMore( ( multop + factor ).setParseAction( self.pushFirst ) )
        expr << term + ZeroOrMore( ( addop + term ).setParseAction( self.pushFirst ) )
        # addop_term = ( addop + term ).setParseAction( self.pushFirst )
        # general_term = term + ZeroOrMore( addop_term ) | OneOrMore( addop_term)
        # expr <<  general_term       
        self.bnf = expr
        # map operator symbols to corresponding arithmetic operations
        epsilon = 1e-12
        self.opn = { "+" : operator.add,
                "-" : operator.sub,
                "*" : operator.mul,
                "/" : operator.truediv,
                "^" : operator.pow }
        self.fn  = { "sin" : math.sin,
                "cos" : math.cos,
                "tan" : math.tan,
                "abs" : abs,
                "trunc" : lambda a: int(a),
                "round" : round,
                "sgn" : lambda a: abs(a)>epsilon and cmp(a,0) or 0}
    def evaluateStack(self, s ):
        op = s.pop()
        if op == 'unary -':
            return -self.evaluateStack( s )
        if op in "+-*/^":
            op2 = self.evaluateStack( s )
            op1 = self.evaluateStack( s )
            return self.opn[op]( op1, op2 )
        elif op == "PI":
            return math.pi # 3.1415926535
        elif op == "E":
            return math.e  # 2.718281828
        elif op in self.fn:
            return self.fn[op]( self.evaluateStack( s ) )
        elif op[0].isalpha():
            return 0
        else:
            return float( op )
    def eval(self,num_string,parseAll=True):
        self.exprStack=[]
        results=self.bnf.parseString(num_string,parseAll)
        val=self.evaluateStack( self.exprStack[:] )
        return val

if __name__ == '__main__':
    import sys

    m = open(sys.argv[1],"r").read().rstrip("=?\n")
    nsp = NumericStringParser()


    clean_m = m.rstrip("= ?")      \
           .replace("o","0")   \
           .replace(" It ","*")\
           .replace(" ; ","*") \
           .replace(' 1 ','*') \
           .replace(' 6 ','*') \
           .replace(' 5 ','*') \
           .replace(' 2 ','*') \
           .replace(' 8 ','*') \
           .replace(' 4 ','*') \
           .replace(' 7 ','-') \
           .replace('x','*')   \
           .replace(" 3 ", "*")\
           .replace("\xe2\x80\x94","-")

    print int(nsp.eval( clean_m))
{% endhighlight python %}
Notice the ugly serie of <code>replace</code>'s which are here to sanitize the decoded equation. the <code>*</code> symbol was notoriously incorrectly recognized.

## CSI Enhancing Part :

- - - - - - -

However, that wasn't enough since tesseract had difficulties differenciating <code>3</code>'s from <code>8</code>'s. I could answer several equations, but I always got one wrong answer every ten-ish equations, which wouldn't cut it. I had to ease the work done for tesseract.

I used ImageMagick (I never ceased to be amazed by its efficiency) convert tool to enbiggen the gif equations before feeding into tesseract : <code>convert $equation.gif -resize "200%" big_equation.gif</code>.

Once we get every part right, you just need to wrap it up using a simple shell script :


{% highlight sh %}
curl -c cookies.txt "http://212.254.178.162/1JnjqflWseX_29Ow-YXH/" -o 1JnjqflWseX_29Ow-YXH1.gif
for i in {1..30}
do
  fday=1JnjqflWseX_29Ow-YXH$i
  eqday=equation$i
  convert $fday.gif -resize "200%" big.gif
  tesseract big.gif $eqday
  payload=$(python numerical_parser.py $eqday.txt | tr -d '\r')
  echo $payload
  curl -X POST -b cookies.txt --data "result="$payload "http://212.254.178.162/1JnjqflWseX_29Ow-YXH/" -o 1JnjqflWseX_29Ow-YXH$((i+1)).gif
done
{% endhighlight sh %}


You get 30 differents gif files, the latest one being a x-mas qr-ball :p
