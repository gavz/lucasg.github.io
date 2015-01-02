---
layout: post
title: "HACKVent 2014 - Day 15 writeup"
date: 2014-12-15
---

<small>
I've sign up for the <a href = "hackvent.hacking-lab.com"> Hackvent event </a> made by the guys from <a href = "www.hacking-lab.com"> www.hacking-lab.com</a>, which is a advent-like hacking competition. Every day there is a new challenge posted at midnight which has a to solved at best in the same day, the challenge becoming increasingly more difficult every week completed. The aim in every puzzle is to find either a qr-encoded x-mas ball with lead to the validation code, or a secret human-readable string which gives you the former ball when feeding into a validator (the "Ball-O-Matic"). 
</small>

Here's the write-up for the challenge at day 15, in which we will learn about a useful trick for black hats.

<!--more-->

## Ghost in the shell Part :

- - - - - - -


For the Day 15 Hackvent challenge, we were given the following instructions :

![Riddle from hackvent.hacking-lab.com for Day 15](/assets/hackvent/15/riddle.png)


The text displayed is obviously a base64 string (look for the padding). After decoding :
<pre>
C8080000C645F877B013347E8845F96A61598D45F883C0028808EB013F2BC06A398B042483C40403C04083E10183C9018D7DFBF3AA83E00183C85583F0338D4DFC515A8802EB0175E8000000005B8A5BFA8AC386C8884DFD90CC33C005D100000083E8635083C4048A5C24FC885DFEFE45F8C9C3
</pre> 

This text looks like some hex-encoded string. Moreover, the challenge title 'Execute the inevitable' may refer to an executable code. Using the nasm disassembler :


<pre>
00000000 | C8080000   | enter 0x8,0x0
00000004 | C645F877   | mov byte [di-0x8],0x77
00000008 | B013       | mov al,0x13
0000000A | 347E       | xor al,0x7e
0000000C | 8845F9     | mov [di-0x7],al
0000000F | 6A61       | push byte +0x61
00000011 | 59         | pop cx
00000012 | 8D45F8     | lea ax,[di-0x8]
00000015 | 83C002     | add ax,byte +0x2
00000018 | 8808       | mov [bx+si],cl
0000001A | EB01       | jmp short 0x1d
0000001C | 3F         | aas
0000001D | 2BC0       | sub ax,ax
0000001F | 6A39       | push byte +0x39
00000021 | 8B04       | mov ax,[si]
00000023 | 2483       | and al,0x83
00000025 | C404       | les ax,[si]
00000027 | 03C0       | add ax,ax
00000029 | 40         | inc ax
0000002A | 83E101     | and cx,byte +0x1
0000002D | 83C901     | or cx,byte +0x1
00000030 | 8D7DFB     | lea di,[di-0x5]
00000033 | F3AA       | rep stosb
00000035 | 83E001     | and ax,byte +0x1
00000038 | 83C855     | or ax,byte +0x55
0000003B | 83F033     | xor ax,byte +0x33
0000003E | 8D4DFC     | lea cx,[di-0x4]
00000041 | 51         | push cx
00000042 | 5A         | pop dx
00000043 | 8802       | mov [bp+si],al
00000045 | EB01       | jmp short 0x48
00000047 | 75E8       | jnz 0x31
00000049 | 0000       | add [bx+si],al
0000004B | 0000       | add [bx+si],al
0000004D | 5B         | pop bx
0000004E | 8A5BFA     | mov bl,[bp+di-0x6]
00000051 | 8AC3       | mov al,bl
00000053 | 86C8       | xchg cl,al
00000055 | 884DFD     | mov [di-0x3],cl
00000058 | 90         | nop
00000059 | CC         | int3
0000005A | 33C0       | xor ax,ax
0000005C | 05D100     | add ax,0xd1
0000005F | 0000       | add [bx+si],al
00000061 | 83E863     | sub ax,byte +0x63
00000064 | 50         | push ax
00000065 | 83C404     | add sp,byte +0x4
00000068 | 8A5C24     | mov bl,[si+0x24]
0000006B | FC         | cld
0000006C | 885DFE     | mov [di-0x2],bl
0000006F | FE45F8     | inc byte [di-0x8]
00000072 | C9         | leave
00000073 | C3         | ret
</pre>

Trying to simply compile it and run it as a standalone executable don't work. Instead we have to inject it in a running program as a <a href="http://en.wikipedia.org/wiki/Shellcode"> shellcode </a>. You have to craft a little snippet in order to load the executable code :

{% highlight C %}
#include <stdio.h>
char code[] = "\xc8\x08\x00\x00\xc6\x45\xf8\x77\xb0\x13\x34\x7e\x88\x45\xf9\x6a\x61\x59\x8d\x45\xf8\x83\xc0\x02\x88\x08\xeb\x01\x3f\x2b\xc0\x6a\x39\x8b\x04\x24\x83\xc4\x04\x03\xc0\x40\x83\xe1\x01\x83\xc9\x01\x8d\x7d\xfb\xf3\xaa\x83\xe0\x01\x83\xc8\x55\x83\xf0\x33\x8d\x4d\xfc\x51\x5a\x88\x02\xeb\x01\x75\xe8\x00\x00\x00\x00\x5b\x8a\x5b\xfa\x8a\xc3\x86\xc8\x88\x4d\xfd\x90\xcc\x33\xc0\x05\xd1\x00\x00\x00\x83\xe8\x63\x50\x83\xc4\x04\x8a\x5c\x24\xfc\x88\x5d\xfe\xfe\x45\xf8\xc9\xc3";

int main(int argc, char **argv)
{
  char* (*func)();
  func = (char* (*)()) code;

  printf("%s", (char*)(*func)());
}
{% endhighlight C %}

When running this shellcode, you don't get directly the solution to this challenge since the executable is SIGTRAP'd. At address 0x59, we have an INT3 instruction, which is a debugging breaking point. When lauching a debugger, you've got this :

![Hack it until you make it](/assets/hackvent/15/immunity.png)


On the stack, we can see an address containing "xmas" and another one "fun", which is in fact the answer to this challenge.