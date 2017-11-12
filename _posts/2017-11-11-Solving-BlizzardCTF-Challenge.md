---
layout: post
title: "Solving Blizzard CTF Challenge"
date: 2017-11-11
---

Recently on my Twitter feed popped [a challenge for BlizzardCTF](https://twitter.com/0xeb/status/928059399150567424)  and I decided to give it a go :

![that's some lich king](/assets/WelcomeScreen.PNG)

<!--more-->


By clicking 4 times on the window, it launch a modal dialog box telling me the sequence is incorrect : 

![](/assets/Incorrect.PNG)

The executable is a nicely looking C# application, one which [DotPeek](https://www.jetbrains.com/decompiler/) likes to decompile. After some reversing, I've located the verification and `MessageBox` launch functions :


![](/assets/timer2_Tick.PNG)

The verification function pop `this.m_coll`, convert them to hex string and concatenate it into a single hex string array. Then it call `<Module>.native_verify`. `<Module>.native_verify` is not referenced anywhere in DotPeek's decompiled code which makes probably a C++/CLI code resolved by a hand-written loader. Instead of trying to locate and reverse the loader, it's easier to trace it using a debugger.

Fortunately, Windbg can debug C# managed code along with traditionnal native C/C++ code thanks to the `sos` extension. If you want to know more about it, [this blog hosts every info you need to know about it](https://blogs.msdn.microsoft.com/alejacma/tag/debugging/).

{% highlight text %}

0:009> .loadby sos clr
0:009> !DumpDomain
c0000005 Exception in C:\Windows\Microsoft.NET\Framework64\v4.0.30319\sos.DumpDomain debugger extension.
      PC: 00007ffb`f8957bf1  VA: 00000000`00000000  R/W: 0  Parameter: ffffc70c`26aef080
0:009> !DumpDomain
--------------------------------------
// * Snipped * 
Assembly:           000001aced7c0880 [F:\Reverse\blizzard ctf\BlizzardLichKing.exe]
ClassLoader:        000001aced7c09d0
SecurityDescriptor: 000001aced7bf7d0
  Module Name
00007ffbb7104110            F:\Reverse\blizzard ctf\BlizzardLichKing.exe
// * Snipped * 
--------------------------------------
0:009>  !DumpModule -mt  00007ffbb7104110 
Name:       F:\Reverse\blizzard ctf\BlizzardLichKing.exe
Attributes: PEFile SupportsUpdateableMethods
Assembly:   000001aced7c0880
LoaderHeap:              0000000000000000
TypeDefToMethodTableMap: 00007ffbb7100070
TypeRefToMethodTableMap: 00007ffbb7100bd8
MethodDefToDescMap:      00007ffbb7100ef0
FieldDefToDescMap:       00007ffbb7101308
MemberRefToDescMap:      0000000000000000
FileReferencesMap:       00007ffbb7102230
AssemblyReferencesMap:   00007ffbb7102238
MetaData start address:  00007ff78e2636c0 (46704 bytes)

Types defined in this module

              MT          TypeDef Name
------------------------------------------------------------------------------
00007ffbb71077d0 0x02000001 
00007ffbb7270098 0x02000002 BlizzardCTF.MyForm
// * Snipped * 
------------------------------------------------------------------------------
0:009> !DumpMT -md 00007ffbb7270098
EEClass:         00007ffbb7218aa8
Module:          00007ffbb7104110
Name:            BlizzardCTF.MyForm
mdToken:         0000000002000002
File:            F:\Reverse\blizzard ctf\BlizzardLichKing.exe
BaseSize:        0x210
ComponentSize:   0x0
Slots in VTable: 386
Number of IFaces in IFaceMap: 21
--------------------------------------
MethodDesc Table
           Entry       MethodDesc    JIT Name
00007ffc00449f60 00007ffc00263408 PreJIT System.Windows.Forms.Form.ToString()
// * Snipped * 
00007ffbb7220410 00007ffbb710c6e0    JIT BlizzardCTF.MyForm.timer1_Tick(System.Object, System.EventArgs)
00007ffbb7220418 00007ffbb710c6f0   NONE BlizzardCTF.MyForm.timer2_Tick(System.Object, System.EventArgs)
0:009> !DumpMD /d 00007ffbb710c6f0
Method Name:  BlizzardCTF.MyForm.timer2_Tick(System.Object, System.EventArgs)
Class:        00007ffbb7218aa8
MethodTable:  00007ffbb7270098
mdToken:      000000000600006f
Module:       00007ffbb7104110
IsJitted:     no
CodeAddr:     ffffffffffffffff
Transparency: Safe critical
{% endhighlight text %}


The issue with debugging C# code is the presence of JIT : unused parts of the application is not "present" (instruction-wise) until the execution hits it, kinda like `MEM_COMMIT` virtual memory is not actually backed by physical memory until a access memory triggers a `#PF`. To overcome this issue, just try to run the target function at least once :

{% highlight text %}
0:009> g
(654.3104): Break instruction exception - code 80000003 (first chance)
ntdll!DbgBreakPoint:
00007ffc`49d320a0 cc              int     3
0:011> !DumpMD /d 00007ffbb710c6f0
Method Name:  BlizzardCTF.MyForm.timer2_Tick(System.Object, System.EventArgs)
Class:        00007ffbb7218aa8
MethodTable:  00007ffbb7270098
mdToken:      000000000600006f
Module:       00007ffbb7104110
IsJitted:     yes
CodeAddr:     00007ffbb7223a30
Transparency: Safe critical
0:011> !U /d 00007ffbb7223a30
Normal JIT generated code
BlizzardCTF.MyForm.timer2_Tick(System.Object, System.EventArgs)
Begin 00007ffbb7223a30, size 2ef
*** ERROR: Module load completed but symbols could not be loaded for F:\Reverse\blizzard ctf\BlizzardLichKing.exe
>>> 00007ffb`b7223a30 55          push    rbp
00007ffb`b7223a31 4157            push    r15
00007ffb`b7223a33 4156            push    r14
00007ffb`b7223a35 4155            push    r13
00007ffb`b7223a37 4154            push    r12
// * Snipped * 
00007ffb`b7223bc9 48b8e86510b7fb7f0000 mov rax,7FFBB71065E8h (MD: <Module>.native_verify(Char*))
00007ffb`b7223bd3 48898570ffffff  mov     qword ptr [rbp-90h],rax
00007ffb`b7223bda 488d0519000000  lea     rax,[00007ffb`b7223bfa]
00007ffb`b7223be1 48894588        mov     qword ptr [rbp-78h],rax
00007ffb`b7223be5 488d8560ffffff  lea     rax,[rbp-0A0h]
00007ffb`b7223bec 48894710        mov     qword ptr [rdi+10h],rax
00007ffb`b7223bf0 c6470c00        mov     byte ptr [rdi+0Ch],0
00007ffb`b7223bf4 ff15962ceeff    call    qword ptr [00007ffb`b7106890] (<Module>.native_verify(Char*), mdToken: 0000000006000051)
00007ffb`b7223bfa c6470c01        mov     byte ptr [rdi+0Ch],1
// * Snipped * 
007ffb`b7223d17 5d                pop     rbp
00007ffb`b7223d18 c3              ret
00007ffb`b7223d19 e8d2f3a85f      call    clr!JIT_RngChkFail (00007ffc`16cb30f0)
00007ffb`b7223d1e cc              int     3
{% endhighlight text %}


Now we can break on `<Module>.native_verify` and examine what's going on behind the call. `<Module>.native_verify` actually calls  `clr!NDirectImportThunk` to resolve a C#/managed binding and call the following function (reversed by IDA) :

![Yeah I know we don't see shit](/assets/native_verify.PNG)

Screenshots of IDA graph view are not the best explaining tool so I've schematized the function flow graph :

![That's actually better](/assets/natives_verify_schema.png)



There are two "external" calls inside this function (as in IDA cannot resolve the address) that need to be resolved. The first one is simply a call to `wscncpy`, but the second one actually does the final verification resulting in a OK/KO return code. IDA cannot resolve the address since the function is located in a JIT page :


![tracking down jit code](/assets/jit_page.PNG)


Fortunately, IDA can import JIT pages as additionnal binaries and resolve functions calls based on base address. Inserting the function into allow me to decompile it using HexRays : 


{% highlight C %}
__int64 __fastcall __second_external_call(char *InBuffer, size_t BufferSize, size_t MagicValue)
{
  BYTE *InByteBuffer; // rax@1
  int MagicValue_1; // ecx@1
  size_t Hi32MagicValue; // r8@1
  size_t DwordCounter; // r9@2
  DWORD CurrentValue; // ecx@3
  int CurrentValue_1; // ecx@3
  int v9; // er9@4
  int Value; // ecx@10
  int v11; // er8@10
  int v12; // ecx@10
  int v13; // er8@10
  int v14; // ecx@10
  int v15; // er8@10
  int v16; // ecx@10
  int v17; // er8@10
  int v18; // ecx@10

  InByteBuffer = (BYTE *)InBuffer;
  MagicValue_1 = MagicValue;
  Hi32MagicValue = MagicValue >> 32;
  if ( BufferSize >= 4 )
  {
    DwordCounter = BufferSize >> 2;
    BufferSize += -4i64 * (BufferSize >> 2); // BufferSize = BufferSize % 4
    do
    {
      CurrentValue = (*InByteBuffer | ((InByteBuffer[1] | ((InByteBuffer[2] | (InByteBuffer[3] << 8)) << 8)) << 8))
                   + MagicValue_1;
      LODWORD(Hi32MagicValue) = CurrentValue ^ Hi32MagicValue;
      CurrentValue = __ROL4__(CurrentValue, 20);
      CurrentValue_1 = Hi32MagicValue + CurrentValue;
      LODWORD(Hi32MagicValue) = __ROL4__(Hi32MagicValue, 9);
      LODWORD(Hi32MagicValue) = CurrentValue_1 ^ Hi32MagicValue;
      CurrentValue_1 = __ROL4__(CurrentValue_1, 27);
      MagicValue_1 = Hi32MagicValue + CurrentValue_1;
      LODWORD(Hi32MagicValue) = __ROL4__(Hi32MagicValue, 19);
      InByteBuffer += 4;
      --DwordCounter;
    }
    while ( DwordCounter );
  }
  v9 = 128;
  if ( BufferSize == 1 )
    goto LABEL_9;
  if ( BufferSize == 2 )
  {
LABEL_8:
    v9 = InByteBuffer[1] | (v9 << 8);
LABEL_9:
    v9 = *InByteBuffer | (v9 << 8);
    goto LABEL_10;
  }
  // ALWAYS TRUE
  if ( BufferSize == 3 )
  {
    v9 = InByteBuffer[2] | 0x8000;
    goto LABEL_8;
  }
LABEL_10:
  Value = v9 + MagicValue_1;
  v11 = Value ^ Hi32MagicValue;
  Value = __ROL4__(Value, 20);
  v12 = v11 + Value;
  v11 = __ROL4__(v11, 9);
  v13 = v12 ^ v11;
  v12 = __ROL4__(v12, 27);
  v14 = v13 + v12;
  v13 = __ROL4__(v13, 19);
  v15 = v14 ^ v13;
  v14 = __ROL4__(v14, 20);
  v16 = v15 + v14;
  v15 = __ROL4__(v15, 9);
  v17 = v16 ^ v15;
  v16 = __ROL4__(v16, 27);
  v18 = v17 + v16;
  v17 = __ROL4__(v17, 19);
  return v18 ^ (unsigned int)v17;
}
{% endhighlight C %}


HexRays decompiled code is not really human readable, but it does not matter since the extracted code is just meant to be run as a black box. Now that we identified every block in the verification function, let's go back to the flow graph :

![That's actually better](/assets/natives_verify_schema.png)

What's interesting in the verification function is everything coming after the `malloc` instruction depends only on the 32-bit computed "seed" integer which is derived from the input strings representing the clicks' coordinates. That means we can try to bruteforce seed values that we return a correct value. After some HexRays and a ad-hoc C program, I found the following valid seed value : `0x33746715`. 

Not shown in the schema, but every click coordinates account for only one byte of the computed 32-bit seed value, so we can try to find matching coordinates for each byte of our valid seed : 

{% highlight C %}
void create_seed_table()
{
  uint64_t IntegerCode;

  // We only look at a 25 pt radius around a particular point (543, 175)
  for (size_t x = 543-25; x <= 543+25; x++)
  {
    for (size_t y = 175-25; y <= 175+25; y++)
    {
      IntegerCode = (x + 1981);
      IntegerCode <<= 32;
      IntegerCode += (y + 17);

      uint32_t HiIntegerCode = (IntegerCode >> 32);

      if ((851 * ((signed int)IntegerCode - 17) - 1885) > sizeof(_some_char_table))
      {
        /*printf("x: %x , y: %x = ERROR\n", x, y);*/
      }
      else
      {
      	// don't want to demangle this mess
        uint8_t ByteSeed = *(&_some_char_table[851 * ((signed int)IntegerCode - 17) - 1885] + HiIntegerCode) ^ _some_index_table[((HiIntegerCode - 1981 + 25) % 830)] * _some_index_table[(((signed int)IntegerCode - 17 + 50) % 830)];
        printf("x: %d , y: %d = %x\n", x, y, ByteSeed);
      }
    }
  }
}
{% endhighlight C %}

Using this generated table, I was able to find matching coordinates for the seed : `(543, 175), (568, 175), (567, 191), (535, 196)`. Since I'm lazy and I don't want to click precise locations every time I'm testing/trying to solve the challenge, I've implemented a clicker in Python : 

{% highlight Python %}
from ctypes import windll
import win32gui
import ctypes 
import re
import time

MOUSEEVENTF_MOVE = 0x0001 
MOUSEEVENTF_ABSOLUTE = 0x8000 
MOUSEEVENTF_MOVEABS = MOUSEEVENTF_MOVE + MOUSEEVENTF_ABSOLUTE

MOUSEEVENTF_LEFTDOWN = 0x0002 
MOUSEEVENTF_LEFTUP = 0x0004 
MOUSEEVENTF_CLICK = MOUSEEVENTF_LEFTDOWN + MOUSEEVENTF_LEFTUP

def click(x, y):
    x = 65536 * x // ctypes.windll.user32.GetSystemMetrics(0) + 1
    y = 65536 * y // ctypes.windll.user32.GetSystemMetrics(1) + 1
    ctypes.windll.user32.mouse_event(MOUSEEVENTF_MOVEABS, x, y, 0, 0)

    ctypes.windll.user32.mouse_event(MOUSEEVENTF_CLICK, 0, 0, 0, 0)


if __name__=='__main__':
    hProgman = windll.User32.FindWindowW( "WindowsForms10.Window.8.app.0.141b42a_r6_ad1", 0 )
    hWindow = windll.User32.FindWindowExW( hProgman, 0, "WindowsForms10.Window.8.app.0.141b42a_r6_ad1", 0 )
    if hWindow != 0 and hProgman != 0 :
        print("Found application %x !" % hProgman)
        print("Found window %x !" % hWindow)
        
        (x_tl, y_tl, x_br, y_br) = win32gui.GetWindowRect(hWindow)
        click(x_tl+543, y_tl+175)
        click(x_tl+568, y_tl+175)
        click(x_tl+567, y_tl+191)
        click(x_tl+535, y_tl+196)

{% endhighlight Python %}


When running the clicker, I got the following modal box : 


![](/assets/Success.PNG)

<br>
<br>
## <a name="Appendix"></a>  Appendix

### BlizzardCTF_Cracker.c
{% highlight C %}
#include <windows.h>
#include <intrin.h>
#include <immintrin.h>
#include <stdio.h>
#include <stdint.h>
#include <stdbool.h>
#include <string.h>

uint32_t _some_index_table[830] = {
	0x3FB, 0x3FD, 0x407, 0x409, 0x40F, 0x419, 0x41B, 0x425, 0x427,
	0x42D, 0x43F, 0x443, 0x445, 0x449, 0x44F, 0x455, 0x45D, 0x463,
	0x469, 0x47F, 0x481, 0x48B, 0x493, 0x49D, 0x4A3, 0x4A9, 0x4B1,
	0x4BD, 0x4C1, 0x4C7, 0x4CD, 0x4CF, 0x4D5, 0x4E1, 0x4EB, 0x4FD,
	/* Snipped */
	0x1C2D, 0x1C33, 0x1C3D, 0x1C45, 0x1C4B, 0x1C4F, 0x1C55, 0x1C73,
	0x1C81, 0x1C8B, 0x1C8D, 0x1C99, 0x1CA3, 0x1CA5, 0x1CB5, 0x1CB7,
	0x1CC9, 0x1CE1, 0x1CF3, 0x1CF9, 0x1D09, 0x1D1B, 0x1D21, 0x1D23,
	0x1D35, 0x1D39, 0x1D3F, 0x1D41, 0x1D4B, 0x1D53, 0x1D5D, 0x1D63,
	0x1D69, 0x1D71, 0x1D75, 0x1D7B, 0x1D7D, 0x1D87, 0x1D89, 0x1D95,
	0x1D99, 0x1D9F, 0x1DA5, 0x1DA7, 0x1DB3, 0x1DB7, 0x1DC5, 0x1DD7,
	0x1DDB, 0x1DE1, 0x1DF5, 0x1DF9, 0x1E01, 0x1E07, 0x1E0B, 0x1E13,
	0x1E17, 0x1E25, 0x1E2B, 0x1E2F, 0x1E3D, 0x1E49, 0x1E4D, 0x1E4F,
	0x1E6D, 0x1E71, 0x1E89, 0x1E8F, 0x1E95, 0x1EA1, 0x1EAD, 0x1EBB,
	0x1EC1, 0x1EC5, 0x1EC7, 0x1ECB, 0x1EDD, 0x1EE3, 0x1EEF
};

BYTE _some_char_table[460488] = {
	0xAB, 0x61, 0xAF, 0xBB, 0x6F, 0x36, 0xA1, 0x47, 0xC4, 0x7E, 0x48, 0x72, 0xEF, 0xC1, 0x98, 0xFE,
	0x4B, 0x23, 0xCA, 0x8F, 0xEC, 0x80, 0x4D, 0xCD, 0xF8, 0x8C, 0xE2, 0x88, 0x50, 0x5E, 0xFD, 0x8D,
	0x15, 0xBB, 0x7B, 0xBE, 0xC2, 0xE4, 0xA2, 0xA4, 0xCE, 0x15, 0xA8, 0xBA, 0x04, 0xB9, 0x21, 0xB6,
	0xFC, 0xDC, 0x0A, 0xE5, 0x8F, 0x2E, 0x07, 0xCC, 0x32, 0xD5, 0x88, 0xC2, 0x0C, 0xFA, 0x1D, 0x00,
	0xBB, 0x83, 0x41, 0xE5, 0x7D, 0xE2, 0x95, 0xF4, 0x08, 0xE0, 0x74, 0x33, 0xAA, 0x88, 0x2B, 0xB0,
	0x09, 0xCD, 0xB0, 0xFD, 0x77, 0x00, 0x28, 0x21, 0x8F, 0xB1, 0x50, 0x1C, 0x7C, 0x14, 0x90, 0x00,
	0x6B, 0x58, 0xF2, 0x35, 0xAB, 0x39, 0xB3, 0x0A, 0x26, 0x1A, 0x53, 0x9D, 0x4F, 0x39, 0x6B, 0x37,
	0x1D, 0x3D, 0x54, 0xDB, 0x53, 0x6D, 0x8B, 0x1F, 0x05, 0x20, 0x5A, 0x3B, 0x58, 0x74, 0xB5, 0x60,
	/* Snipped */
	0x56, 0x91, 0xAB, 0xEB, 0xB4, 0x2C, 0xF9, 0xDF, 0x78, 0x3E, 0xEB, 0xDB, 0x52, 0x88, 0xC6, 0x5B,
	0xDC, 0x0A, 0x20, 0xF1, 0x72, 0x81, 0x4F, 0xE7, 0x44, 0x2E, 0x21, 0x26, 0xA9, 0xD9, 0xCE, 0x5A,
	0x4D, 0x1F, 0x95, 0x9E, 0xB7, 0x81, 0xD1, 0x0A, 0x52, 0x57, 0x9F, 0xED, 0xFF, 0x89, 0x9C, 0x83,
	0xDE, 0x4B, 0x78, 0xC6, 0xCC, 0x95, 0x9E, 0x88, 0x2A, 0xD3, 0x17, 0xBF, 0xE1, 0xDF, 0xAB, 0x20,
	0xA2, 0x2C, 0x4A, 0xEF, 0x73, 0x44, 0xAB, 0x00,
};

uint32_t  check_bytearray(uint8_t *InBuffer, size_t BufferSize, size_t MagicValue)
{
	BYTE *InByteBuffer; // rax@1
	uint32_t MagicValue_1; // ecx@1
	size_t Hi32MagicValue; // r8@1
	size_t DwordCounter; // r9@2
	uint32_t CurrentValue; // ecx@3
	uint32_t CurrentValue_1; // ecx@3
	int v9; // er9@4
	int Value; // ecx@10
	int v11; // er8@10
	int v12; // ecx@10
	int v13; // er8@10
	int v14; // ecx@10
	int v15; // er8@10
	int v16; // ecx@10
	int v17; // er8@10
	int v18; // ecx@10

	InByteBuffer = (BYTE *)InBuffer;
	MagicValue_1 = (uint32_t) MagicValue;
	Hi32MagicValue = MagicValue >> 32;
	if (BufferSize >= 4)
	{
		DwordCounter = BufferSize >> 2;
		do
		{
			CurrentValue = (InByteBuffer[0] | ((InByteBuffer[1] | ((InByteBuffer[2] | (InByteBuffer[3] << 8)) << 8)) << 8)) + MagicValue_1;
			//printf("CurrentValue : %x\n", CurrentValue);

			Hi32MagicValue = CurrentValue ^ Hi32MagicValue;
			CurrentValue = _rotl(CurrentValue, 20);

			CurrentValue += Hi32MagicValue;
			Hi32MagicValue = _rotl(Hi32MagicValue, 9);

			Hi32MagicValue = CurrentValue  ^ Hi32MagicValue;
			CurrentValue = _rotl(CurrentValue, 27);

			MagicValue_1 = Hi32MagicValue + CurrentValue;
			Hi32MagicValue = _rotl(Hi32MagicValue, 19);

			InByteBuffer += sizeof(uint32_t);
			--DwordCounter;
		} while (DwordCounter);
	}

	// BufferSize = BufferSize % 4
	BufferSize = BufferSize % 4;

	v9 = 128;
	if (BufferSize == 1)
		goto LABEL_9;
	if (BufferSize == 2)
	{
	LABEL_8:
		v9 = InByteBuffer[1] | (v9 << 8);
	LABEL_9:
		v9 = *InByteBuffer | (v9 << 8);
		goto LABEL_10;
	}
	// ALWAYS TRUE
	if (BufferSize == 3)
	{
		v9 = InByteBuffer[2] | 0x8000;
		goto LABEL_8;
	}
LABEL_10:
	Value = v9 + MagicValue_1;
	v11 = Value ^ Hi32MagicValue;
	Value = _rotl(Value, 20);
	v12 = v11 + Value;
	v11 = _rotl(v11, 9);
	v13 = v12 ^ v11;
	v12 = _rotl(v12, 27);
	v14 = v13 + v12;
	v13 = _rotl(v13, 19);
	v15 = v14 ^ v13;
	v14 = _rotl(v14, 20);
	v16 = v15 + v14;
	v15 = _rotl(v15, 9);
	v17 = v16 ^ v15;
	v16 = _rotl(v16, 27);
	v18 = v17 + v16;
	v17 = _rotl(v17, 19);
	// printf("v18 : %x v17 : %x\n", v18, v17);
	return v18 ^ (unsigned int)v17;
}

bool __cdecl _native_verify(wchar_t *ByteCode)
{
	wchar_t *StrArray; // rbx@1
	BYTE *pComputedValue; // rdi@1
	signed __int64 CodeCounter; // rsi@1
	BYTE *AllocatedByteArray; // rax@3
	char CurrentValue; // r10@3
	signed int Increment; // er8@3
	BYTE *pCurrentByte; // r9@3
	const CHAR *v8; // rdi@3
	int Counter; // ebx@4
	char CurrentComputedIncrement; // cl@4
	int MangleActionId; // er8@4
	int MangleActionId2; // er8@5
	int MangleActionId3; // er8@6
	int MangleActionId4; // er8@7
	int MangleActionId5; // er8@8
	__int64 result; // rax@19
	__int64 v17; // [sp+0h] [bp-78h]@19
	int ComputedSeed; // [sp+30h] [bp-48h]@1
	__int64 IntegerCode; // [sp+38h] [bp-40h]@2
	wchar_t StrIntegerCode[17]; // [sp+40h] [bp-38h]@1
	__int64 cookie; // [sp+68h] [bp-10h]@19

	StrArray = ByteCode;
	ComputedSeed = 0;
	memset(&StrIntegerCode, 0, sizeof(StrIntegerCode));
	pComputedValue = (BYTE *)&ComputedSeed;
	// -------------
	// Convert each coor into integer and compute an integer out of it
	CodeCounter = 4;
	do
	{
		wcsncpy(StrIntegerCode, StrArray, 16);
		swscanf_s(StrIntegerCode, L"%llx", &IntegerCode);
		StrArray += 16;
		IntegerCode ^= 0xAAAAAAAA55555555ui64;
		uint32_t HiIntegerCode = (IntegerCode >> 32);
		*pComputedValue++ = *(&_some_char_table[851 * ((signed int)IntegerCode - 17) - 1885] + HiIntegerCode) ^ _some_index_table[((HiIntegerCode - 1981 + 25) % 830)] * _some_index_table[(((signed int)IntegerCode - 17 + 50) % 830)];
		--CodeCounter;
	} while (CodeCounter);
	// ---------------------
	AllocatedByteArray = (BYTE*) malloc(0x5F);
	CurrentValue = ComputedSeed;
	Increment = 0;
	pCurrentByte = AllocatedByteArray;
	v8 = (CHAR*) AllocatedByteArray;
	do
	{
		Counter = Increment + 1;
		CurrentComputedIncrement = *((BYTE *)&ComputedSeed + (Increment + 1) % 4);
		MangleActionId = Increment % 6;
		if (MangleActionId)
		{
			MangleActionId2 = MangleActionId - 1;
			if (MangleActionId2)
			{
				MangleActionId3 = MangleActionId2 - 1;
				if (MangleActionId3)
				{
					MangleActionId4 = MangleActionId3 - 1;
					if (MangleActionId4)
					{
						MangleActionId5 = MangleActionId4 - 1;
						if (MangleActionId5)
						{
							if (MangleActionId5 == 1)
								CurrentComputedIncrement -= 'Q';
						}
						else
						{
							CurrentComputedIncrement += 4;
						}
					}
					else
					{
						CurrentComputedIncrement ^= 0x11u;
					}
				}
				else
				{
					CurrentComputedIncrement = ~CurrentComputedIncrement;
				}
			}
			else
			{
				CurrentComputedIncrement = _rotr8(CurrentComputedIncrement, 2);
			}
		}
		else
		{
			CurrentComputedIncrement = _rotl8(CurrentComputedIncrement, 1);
		}
		CurrentValue += CurrentComputedIncrement;
		Increment = Counter;

		size_t i = pCurrentByte - AllocatedByteArray;
		uint8_t tmp = CurrentValue ^ _some_char_table[i];
		printf("AllocatedByteArray[%d] = %x \n", i, tmp);
		*pCurrentByte = tmp;

		++pCurrentByte;
	} while (Counter < 95);

	uint32_t ret = check_bytearray(AllocatedByteArray, 95, 0x00000417198100eb);
	if (ret == 0x70854A82)
	{
		// SUCCESS !
		return true;
	}
	return false;
}

uint32_t bruteforce_seed()
{
	BYTE *pComputedValue; // rdi@1
	signed __int64 CodeCounter; // rsi@1
	BYTE *AllocatedByteArray; // rax@3
	char CurrentValue; // r10@3
	signed int Increment; // er8@3
	BYTE *pCurrentByte; // r9@3
	const CHAR *v8; // rdi@3
	int Counter; // ebx@4
	char CurrentComputedIncrement; // cl@4
	int MangleActionId; // er8@4
	int MangleActionId2; // er8@5
	int MangleActionId3; // er8@6
	int MangleActionId4; // er8@7
	int MangleActionId5; // er8@8
	__int64 result; // rax@19
	__int64 v17; // [sp+0h] [bp-78h]@19
	int ComputedSeed; // [sp+30h] [bp-48h]@1
	__int64 IntegerCode; // [sp+38h] [bp-40h]@2
	wchar_t StrIntegerCode[17]; // [sp+40h] [bp-38h]@1
	__int64 cookie; // [sp+68h] [bp-10h]@19


	AllocatedByteArray = (BYTE*)malloc(0x5F);
	uint64_t  MaxComputedSeed = UINT_MAX;
	for (uint64_t ComputedSeed = 0; ComputedSeed <= UINT_MAX; ComputedSeed++)
	{
		memset(AllocatedByteArray, 0, 0x5f);
		CurrentValue = ComputedSeed;
		Increment = 0;
		pCurrentByte = AllocatedByteArray;
		v8 = (CHAR*) AllocatedByteArray;

		do
		{
			Counter = Increment + 1;
			CurrentComputedIncrement = *((BYTE *)&ComputedSeed + (Increment + 1) % 4);
			MangleActionId = Increment % 6;
			if (MangleActionId)
			{
				MangleActionId2 = MangleActionId - 1;
				if (MangleActionId2)
				{
					MangleActionId3 = MangleActionId2 - 1;
					if (MangleActionId3)
					{
						MangleActionId4 = MangleActionId3 - 1;
						if (MangleActionId4)
						{
							MangleActionId5 = MangleActionId4 - 1;
							if (MangleActionId5)
							{
								if (MangleActionId5 == 1)
									CurrentComputedIncrement -= 'Q';
							}
							else
							{
								CurrentComputedIncrement += 4;
							}
						}
						else
						{
							CurrentComputedIncrement ^= 0x11u;
						}
					}
					else
					{
						CurrentComputedIncrement = ~CurrentComputedIncrement;
					}
				}
				else
				{
					CurrentComputedIncrement = _rotr8(CurrentComputedIncrement, 2);
				}
			}
			else
			{
				CurrentComputedIncrement = _rotl8(CurrentComputedIncrement, 1);
			}
			CurrentValue += CurrentComputedIncrement;
			Increment = Counter;

			size_t i = pCurrentByte - AllocatedByteArray;
			uint8_t tmp = CurrentValue ^ _some_char_table[i];
			//printf("AllocatedByteArray[%d] = %x \n", i, tmp);
			*pCurrentByte = tmp;

			++pCurrentByte;
		} while (Counter < 95);

		uint32_t ret = check_bytearray(AllocatedByteArray, 95, 0x00000417198100eb);
		if (ret == 0x70854A82)
		{
			// SUCCESS !
			printf("Found seed : 0x%x (0x%x) \n", ComputedSeed, ret);
			printf("bytearray : ");
			for (auto i = 0; i < 95; i++) {
				printf("0x%x ", AllocatedByteArray[i]);
			}
			printf("\n");

			//return ComputedSeed;
		}
	}
	return 0;
}

bool create_seed_table()
{
	uint64_t IntegerCode;

	for (size_t x = 543-25; x <= 543+25; x++)
	{
		for (size_t y = 175-25; y <= 175+25; y++)
		{


			//IntegerCode = 0xAAAAA;
			//IntegerCode <<= 12;
			//IntegerCode += (x + 1981);
			//IntegerCode <<= 20;
			//IntegerCode += 0x55555;
			//IntegerCode <<= 12;
			//IntegerCode += (y + 17);
			////(0xAAAAA << 44) + (x << 32) + (0x55555 << 12) + y;
			//IntegerCode ^= 0xAAAAAAAA55555555;
			IntegerCode = (x + 1981);
			IntegerCode <<= 32;
			IntegerCode += (y + 17);

			uint32_t HiIntegerCode = (IntegerCode >> 32);

			if ((851 * ((signed int)IntegerCode - 17) - 1885) > sizeof(_some_char_table))
			{
				/*printf("x: %x , y: %x = ERROR\n", x, y);*/
			}
			else
			{
				uint8_t ByteSeed = *(&_some_char_table[851 * ((signed int)IntegerCode - 17) - 1885] + HiIntegerCode) ^ _some_index_table[((HiIntegerCode - 1981 + 25) % 830)] * _some_index_table[(((signed int)IntegerCode - 17 + 50) % 830)];
				printf("x: %d , y: %d = %x\n", x, y, ByteSeed);
			}
		}
	}	

	return true;
}

int main(int argc, char *argv[])
{
	//AAAAA37255555590AAAAA37355555590AAAAA37255555590AAAAA37355555590
	uint8_t array[0x5f] = {
		0x94,0xe5,0xc1,0xc9,0xe4,0x79,0xd8,0xf9,0x6c,0xd2,0x8d,0xfb,0x5c,0x39,0x7a,0x18,
		0xb4,0xe0,0x27,0xbd,0xf0,0xa0,0x74,0x30,0xdf,0xe0,0xb4,0xd2,0x23,0x69,0x9c,0x2b,
		0x85,0x2f,0xd6,0xcf,0x59,0x04,0x68,0x6a,0x29,0xbe,0x7d,0xa0,0x00,0xb1,0x00,0x53,
		0xf3,0x88,0x34,0xa7,0xd4,0x31,0x4e,0x42,0x4a,0xa9,0x1d,0x9b,0x8f,0x32,0xaf,0xb6,
		0x74,0x10,0xfc,0xe7,0x91,0x12,0x9c,0x39,0xff,0xdc,0x52,0x19,0xe9,0x8f,0x1a,0xc6,
		0x69,0xa9,0xcd,0xbc,0x1c,0xb0,0xb2,0xbf,0x38,0xca,0xf5,0xf6,0xa8,0xcc,0x61,
	};

	printf("TEST check_bytearray %x == 0xca4c04ee\n", check_bytearray((uint8_t*)array, 0x5f, 0x00000417198100eb));
	printf("TEST _native_verify %x == false\n", _native_verify(L"AAAAA37255555590AAAAA37355555590AAAAA37255555590AAAAA37355555590"));
	printf("TEST _native_verify %x == true\n", _native_verify(L"AAAAA37655555595AAAAA35F55555595AAAAA35E55555585AAAAA37E55555580"));
	bruteforce_seed();
	create_seed_table();
	return 0;
}
{% endhighlight C %}

