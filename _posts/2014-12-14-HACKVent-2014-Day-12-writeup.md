---
layout: post
title: "HACKVent 2014 - Day 12 writeup"
date: 2014-12-14
---

I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 

Here's the write-up for the mid-point challenge at day 12, concerning reverse engineering SQL scripts. 

<!--more-->

## Investigation Part :

- - - - - - -


For the Day 12 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 12](/assets/hackvent/day12/riddle.png)

There are some "clues"/red herrings in this page: 

* The "oracle" word in the title can mean either the Oracle company (Java, SQL and so on) or an oracle attack, the latter being highly unlikely to be a good lead. 
* "Wrap it up" : it didn't mean anything specific, but it looks suspiscious.
* The ciphertext "617B7E0A0870637F710.." is not a base[32-64] string. It has 64 characters and is hex-encoding compliant, but the ascii equivalent representation isn't readable.
* the ciphertext counts 64 characters,  possibly indicating a sha-256 hash, but then I would be out of luck since it's not known to be easily breakable.
* In ascii representation, the ciphertext counts 32 characters, possibly indicating a md5 hash, but again the representation includes non-printable chars.


Now let's take a look at the attached file : AwesomeCryptTools.pls. Since I don't recognize the file extension, I string'd it to look for file signatures, which will reveal useless since the file is plaintext encoded :

{% highlight pl %}

CREATE OR REPLACE PACKAGE HACKvent wrapped 
a000000
b2
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
9
9e aa
WvN7G0Z97FJZxrcs6FN4mZS8BegwgwL/f54VaS9GOPauf2NhYSanELc0bMJC2BOPRH472079
5mfaCjcsInUbe5dndzKa0MGZwNrhc4rs3619e5RBRwVcR7f+NkctWhGNU27zR628a2pk0WgN
owpf8LWz0sIQk30xpCJbEj5a

/
CREATE OR REPLACE PACKAGE BODY HACKvent wrapped 
a000000
b2
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
abcd
b
372 270
BwGVYsJ7/qYMVlBQVrtLCEIPNvowg41eLiAFfHRVPfmUNJX8eu+9Swzwy8hsbG/gDDciws6O
jlQ30tMfTDw4Z6rkWtoux1Rt/990+PfPBoxBPHHYzgk1AQFbKvl6VaRKDhDsG20RolA8qWUV
8o3eA0hT3zV5HREd/bmi11VuN16oReqp5ftkjfyHS37fkGVTvDf6Gnbg3Dr+4AN41rp8LTuJ
2Yt+NkUMyiZ3Cf2KAAjlzGapA7OFWSs7mq1IGnltsiBR5oPPIgF0MjZtbpkXusj3eEOqp5+c
Y1QM7C0FBKtWkofnuWrRVJIcWH4N44e4q9UGYZMpaaCb1dffQJAo3BNsaM/WzVzGaSjM0dgd
Lh1PlOmMR2V3nNqvDi2f8N76fN9xunfRhocRkDpUqIYBn+JOiAtPKtbBwTj/GuqIrch04REL
yBQuEWGWZcWkn7oewvMu+WNKhVT53OHcQMwTSVxJcnCIYgxiX9HV+7+B5G5iFj36rOZk9kMi
iq+rMk+vr1ld7AMMQHy+Crn+MMG4aJq1RgcFxu/kKaqv+TMpy0oA5H1rJC+b53O7HYkPtBrP
PWcUjB8I6fLUycyh7Boa1nx7o/0C9E/54UwY6yM=

/

{% endhighlight pl %}

The ALL UPPERCASE text is a pretty obvious hint : the file is running some kind of SQL query. Some googling reveals me that a pls file is a scripting langage for SQL and created by the Oracle firm : I am on the right track. Now I have to unmangle the obfuscated code. I noticed the use of the "wrapped" word in the first line  : 

<pre> <code> CREATE OR REPLACE PACKAGE BODY HACKvent wrapped </code> </pre>

It refers to the hackvent riddle page's subtitle "Let's wrap it up". After more googling, I learn that a "wrapper" is an obfuscating procedure for sql code meant to be shipped to external parties and run as-is. Now I need to find an online unwrapper tool.


## UnWrapping Part :

- - - - - - -

![Online unwrapped tools at www.codecrete.net](/assets/hackvent/day12/unwrapper.png)
![De-obfuscated code for the pls script](/assets/hackvent/day12/unwrapped.png)

The unwrapper tells me little from the obfuscated code : I only know that it's a package containing two functions, one being really interesting. Alas I only have their prototype, not the actual body. Thinking that the online tools, I try various ones -python-based ones; php based-ones; 500 internal error-based ones- without success. I'm now resigned to try to run the script in a online sql environnmenent in order to try to find a new information on it.

The sqlfiddler website being DDOS'ed from the hackvent candidates trying to decrypt day 12 riddle, I use insted the APEX, oracle web environnement for running sql queries. When running the mangled pls script in the apex, I'm getting this error message :

<pre> <code> Error at line 0: PLS-00753: malformed or corrupted wrapped unit 0.02 seconds </code> </pre> 

Google's first anwser to this error message is a blog post in which the author claim that the error result from the following condition : “This only happens if the last character of the wrapped code is at the end of a line”. Well let's take another look at the script :


![Script text structure](/assets/hackvent/day12/2_queries.png)

When looking at the script's structure, we can deduct that there is two queries, and the second one has the "BODY" tag in it ! I must have spent two hours before finding this hint ... From now on, let's stop being stupid and decoding the body part of the script:


{% highlight pl %}

PACKAGE BODY HACKvent IS

	FUNCTION ENCODE(INPLAINTEXT IN VARCHAR2) RETURN VARCHAR2 IS
    KEY VARCHAR(100) := '';
    RES VARCHAR(100) := '';
    RES1 VARCHAR(100) := '';
    X NUMBER(2);
    Y NUMBER(2);
	BEGIN

		SELECT ROUND(DBMS_RANDOM.VALUE(10,20)) INTO X FROM DUAL;
		SELECT ROUND(DBMS_RANDOM.VALUE(10,20)) INTO Y FROM DUAL;

		KEY := UTL_RAW.CAST_TO_RAW(DBMS_OBFUSCATION_TOOLKIT.MD5(INPUT_STRING => X*Y));


		FOR I IN 1..LENGTH(INPLAINTEXT) LOOP
			RES1 := CHR(ASCII(SUBSTR(INPLAINTEXT,I,1))+I) || RES1;
		END LOOP;

		RES := 
	    UTL_RAW.BIT_XOR(
	    	UTL_RAW.CAST_TO_RAW(RES1),
	    	UTL_RAW.CAST_TO_RAW(KEY)
			);


		RETURN RES;

	END ENCODE; 
  
  
  
  FUNCTION DECODE(INCIPHERTEXT IN VARCHAR2) RETURN VARCHAR2 IS
    HINT VARCHAR2(100);
  BEGIN

		HINT := 'Hohoho, this part you have to do on your own ;-)';
		RETURN HINT;
    
  END DECODE; 

	

  
END HACKVENT; 

{% endhighlight pl %}


Okay, this is much better good looking ! I feel I'm definitively on the right track, although I'm not done : there is no real decoding function. So I'm stuck with reverse engineering the encoding scheme (which is not secure by the way) to reveal the encoded secret.


## Decoding Part :

- - - - - - -


At this point, I have to say that I know almost nothing about SQL, which complicates a lot when you have to write ad-hoc code in it. So at first I went the Python's way, only to hit a dead-end (see the last part of the post to know why). I'm back to trying to do it in PL/SQL. The first thing I've learned is that online pl/sql tools don't provide a printf-like functionnality, so you have to emulate it. <a href = "http://stackoverflow.com/a/19143032/1741450" > Fortunately, it's a popular issue so the fix is easily findable</a>.


{% highlight pl %}

SELECT ROUND(DBMS_RANDOM.VALUE(10,20)) INTO X FROM DUAL;
SELECT ROUND(DBMS_RANDOM.VALUE(10,20)) INTO Y FROM DUAL;

KEY := UTL_RAW.CAST_TO_RAW(DBMS_OBFUSCATION_TOOLKIT.MD5(INPUT_STRING => X*Y));
{% endhighlight pl %}

Now I see why the script's author couldn't decrypt his password : the key used for encryption is chosen randomly ! Fortunately the key space is not that large : x and y can take 10 differents values, so the key seed can take at most 55 uniques values. This keyseed is then feeded into a md5 hash procedure and the result is converted after into a 64-char hex string.


{% highlight pl %}
FOR I IN 1..LENGTH(INPLAINTEXT) LOOP
	RES1 := CHR(ASCII(SUBSTR(INPLAINTEXT,I,1))+I) || RES1;
END LOOP;
{% endhighlight pl %}

That's when I spent a lot of time doing stupid things, since the operation is quite twisted. The  <code>CHR(ASCII(SUBSTR(INPLAINTEXT,I,1))+I)</code> sequence indicate a linear letter shift, the first password letter being shifted by one, the second by two, etc. Of course PL/SQL indexes its array from 1 to N ...

The <code>|| RES1; </code> sequence looks like some kind of concatenation, so it means RES1 is an array of shifted letters. Oh wait, the concatenation is backward : we prepend every char to the array, resulting in reversing the order of letters ! 
Exemple of encryption :



{% highlight pl %}
RES := 
    UTL_RAW.BIT_XOR(
    	UTL_RAW.CAST_TO_RAW(RES1),
    	UTL_RAW.CAST_TO_RAW(KEY)
		);
{% endhighlight pl %}

That's standard XOR encryption here. In order to reverse the encoding algorithm, we have to generate every 55 keyseed possibles and their equivalent hex-based md5 hash, then reversing the twisted-but-deterministic scrambling part and trying to find the secret password in the haystack of non-printable char spewed by the sql script. After a lot of trials-and-errors, this is a working decoder : 

{% highlight pl %}
DROP TABLE DBMSOUTPUT
/

CREATE TABLE  "DBMSOUTPUT"
   (    "POS" NUMBER(*,0),
    "MES" VARCHAR2(4000)
   )
/


DECLARE
KEY VARCHAR(100) := '';
RES VARCHAR(100) := '';
RES1 VARCHAR(100) := '';
SEC VARCHAR(100) := '617B7E0A0870637F710E42B44A3B0647433442441B4E4F1D4B471F29475C5D62';
X number(2) := 0;
Y number(2) := 0;
C number(10) := 0;
D Char;
E number(2) := 0;
procedure put_line(p_mes in varchar2) is
v_pos int;
begin  
  select count(0) into v_pos from dbmsoutput;  
  insert into dbmsoutput (pos, mes) values (v_pos, p_mes);
end;

BEGIN
put_line('Hello World');
for X in 10..20 loop
  for Y in X..20 loop

      KEY := UTL_RAW.CAST_TO_RAW(DBMS_OBFUSCATION_TOOLKIT.MD5(INPUT_STRING => X*Y));      
  
      RES :=
        UTL_RAW.BIT_XOR(
            SEC,
            UTL_RAW.CAST_TO_RAW(KEY)
            );
      
      RES := UTL_RAW.CAST_TO_VARCHAR2(RES);
          
      RES1 := '';
      E := 0;
      FOR I IN 1..LENGTH(RES) LOOP
          C := ASCII(SUBSTR(RES,I,1)) - LENGTH(RES) - 1 + I;
          IF C < 0 THEN
              C :=0;
              E := E + 1;
          END IF;
          IF C > 255 THEN
              C :=255;
              E := E + 1;
          END IF;
          
          D := CHR(C);
          RES1 := D || RES1;
      END LOOP;

      put_line(X || '*' || Y || ' -> ' || 'E, RES1 :' || E || ',' || RES1);
  end loop;
end loop;
END;
/

SELECT mes FROM dbmsoutput order by pos
{% endhighlight pl %}


<pre><code> Results:
....
12*20 -> E, RES1 :5, gkUjs$thhg",7?+$20,5
13*13 -> E, RES1 :4,P"kRo md kdn_"(.A("/.2
13*14 -> E, RES1 :3,Zki{&isRsscgeagp2:0+-5
13*15 -> E, RES1 :4,&k{#i%rr#h!l_ a'%0/)1
13*16 -> E, RES1 :4,Zpf&n#dfgb2fr4 =-!11#1
13*17 -> E, RES1 :0,There is no place like 127.0.0.1
13*18 -> E, RES1 :4,Tbq"qPmkQdbch$22<54,'6
13*19 -> E, RES1 :4,og(mwRphfdd;0// 5
13*20 -> E, RES1 :3,SinhWs! fdg#jerd(6:+0
14*14 -> E, RES1 :5,Zhc&vRrlfb0k_q_'($"+,$1
....
</code></pre>

