# riscos-nantest

This is a simple test to show where the floating point behaviour is wrong on RISC OS.

We try to turn off signals and then calculate NaN by dividing by 0. This should work.
Then we show the values of what we got back.

## RISC OS Pyromaniac behaviour

On the FPEmulator 4.18, with the Select Shared C library (5.62), running under Pyromaniac, the following results are given from the test:

```
pyrodev --common --command 'help FPEmulator' --command aif32.nantest
==> Help on keyword 'FPEmulator' (Module)
Module is: FPEmulator      4.18 (14 Sep 2006) (1.07)
NaN test
nan = nan
nan = 7ff80000/e0000000
same = 1, different = 0
INF test
inf = inf
inf = 7ff00000/00000000
same = 1, different = 0
```

Going through the lines.

1. First we print the version of the FPEmulator.
2. We start the 'NaN test'.
3. `printf` is showing the NaN we got from a division of 0 by 0 as `nan`.
4. We show the representation as two 32bit words, low word then high word, which are sane.
5. We show the value of an equality and inequality check on the NaN. This says that it's equal to itself. That's wrong. NaN is never equal to itself.
6. We start the 'INF test' (which checks the behaviour of infinity).
7. `printf` is showing the infinity we got from a division of 1 by 0 is `inf`.
8. We show the representation as two 32bit words, low word then high word, which looks reasonable.
9. We show the value of an equality and an inequality check on the infinity. That's right.

So there's a bug there.


## Unix behaviour

On Unix (specifically in this case I'm using macOS) we can trust that we probably get sane answers.

```
NaN test
nan = nan
nan = 00000000/7ff80000
same = 0, different = 1
INF test
inf = inf
inf = 00000000/7ff00000
same = 1, different = 0
```

These answers are the same, except that...

- The order of the words differs in the 32bit word representation. That's a difference in the representation that ARM used in the original reference implementation of the FPE.
- NaN does not compare to itself. That's correct.

So that's the behaviour you want.



## RISC OS 5 behaviour

I can test on RISC OS 5, 'cos that's easy - run it under Docker with RPCEmu:

    docker run -it -v $PWD/:/home/riscos/rpcemu/hostfs/Shared/ --rm -p 5901:5901 gerph/rpcemu-5

This is running on RISC OS 5.27 (19 Mar 2020); the RPCEmu 'easy start' images.

```
*Help FPEmulator
==> Help on keyword 'FPEmulator'
Module is: FPEmulator      4.37 (12 Nov 2019) (1.13M)
*aif32.nantest
NaN test

Postmortem requested
fc1367f4 in function nantest
823c in function main
  Arg2: 0x00008224 33316 -> [0xe1a0c00d 0xe92dd800 0xe24cb004 0xe15d000a]
  Arg1: 0x0000b274 45684 -> [0x33666961 0x616e2e32 0x7365746e 0x79540074]
fc139b94 in shared library function
82c8 in anonymous function
```

So... that's not good. It's crashing somewhere in the ROM - and you probably guessed that it's in the FPEmulator at this point... but you (and I) got it wrong. It's actually in the SharedCLibrary.

We don't have any trace other information to say what's going wrong, so we just have to scrounge around for more information

```
*showregs
Register dump (stored at &2000D2D0) is:
R0  = FFFFFFFF R1  = FFFFFFFD R2  = 00000002 R3  = 00009DB8
R4  = 0000B282 R5  = FFFFFFFF R6  = 00000000 R7  = 00000000
R8  = 000099CC R9  = 00009BFC R10 = 0000A4B4 R11 = 0000B1F0
R12 = 0000B110 R13 = 0000B1CC R14 = 000080B8 R15 = 000080C0
Mode USR32 flags set: Nzcvqjggggeaift        PSR = 80000010
```

This shows us that it thinks it's in user space at address `&80C0`. We can show the code around
that point with a `*MemoryI`:

```
*memoryi pc-20+40
000080A0 : .ÐMâ : E24DD008 : SUB     R13,R13,#8
000080A4 : ..Xâ : E28F0F19 : ADR     R0,&00008110
000080A8 : u..ë : EB000375 : BL      &00008E84
000080AC : h.Ÿå : E59F1068 : LDR     R1,&0000811C
000080B0 : .. ã : E3A00002 : MOV     R0,#2
000080B4 : Ø..ë : EB0003D8 : BL      &0000901C
000080B8 : ˆX.î : EE008188 : MVFD    F0,#0
000080BC : ˆ.@î : EE400188 : DVFD    F0,F0,#0
000080C0 < .XXí : ED8D8100 : STFD    F0,[R13,#0]
000080C4 : ..Xâ : E28F0F15 : ADR     R0,&00008120
000080C8 : ..Xè : E89D0006 : LDMIA   R13,{R1,R2}
000080CC : ­X..ë : EB0003AD : BL      &00008F88
000080D0 : ..Xè : E89D0006 : LDMIA   R13,{R1,R2}
000080D4 : ..Xâ : E28F0F14 : ADR     R0,&0000812C
000080D8 : i..ë : EB000369 : BL      &00008E84
000080DC : .` ã : E3A06001 : MOV     R6,#1
```

This implies that we were in the middle of the division (or possibly the store). Either way we've got an address which tells use where the exception was reported in the ROM. So let's dump the code around `&fc1367f4` to see what it's trying to do:

```
*memoryi fc1367f4-20+40
FC1367D4 : .ÅŸå : E59FC518 : LDR     R12,&FC136CF4
FC1367D8 : .".å : E51A221C : LDR     R2,[R10,#-540]
FC1367DC : .ÀŒà : E08CC002 : ADD     R12,R12,R2
FC1367E0 : .  ã : E3A02001 : MOV     R2,#1
FC1367E4 : X#Ìå : E5CC239D : STRB    R2,[R12,#925]
FC1367E8 : ..-é : E92D000A : STMDB   R13!,{R1,R3}
FC1367EC : þ.‘é : E99103FE : LDMIB   R1,{R1-R9}
FC1367F0 : .>.ë : EB003E06 : BL      &FC146010
FC1367F4 : ‘..ê : EA000091 : B       &FC136A40
FC1367F8 : .@-é : E92D4001 : STMDB   R13!,{R0,R14}
FC1367FC : .. á : E1A00002 : MOV     R0,R2
FC136800 : ÍÇ.ë : EB00C7CD : BL      &FC16873C
FC136804 : .P½è : E8BD5000 : LDMIA   R13!,{R12,R14}
FC136808 : ..Œè : E88C0003 : STMIA   R12,{R0,R1}
FC13680C : .ð á : E1A0F00E : MOV     PC,R14
FC136810 : .. á : E1A00000 : MOV     R0,R0
```

Looking at the instruction 2 back from the address given (due to the pipeline), this is `LDMIB R1,{R1-R9}` - load a set of registers from R1 incrementing before each. If we look up at the registers shown by `*ShowRegs` we see that R1 is &FFFFFFFD. So something in the SharedCLibrary is breaking.


## RISC OS 3.7

So how about running it on RISC OS 3.7. That's the common parent of both RISC OS 5 and RISC OS 4.
Again, can do this with Docker:

    docker run -it -v $PWD/:/home/riscos/rpcemu/hostfs/Shared/ --rm -p 5901:5901 gerph/rpcemu-3.7

The output was...

```
*help fpemulator
==> Help on keyword FPEmulator
Module is: FPEmulator   4.08 (17 Jan 1997) (1.07z)

*aif32.nantest
NaN test

Postmortem requested
21fefa8 in function nantest
823c in function main
22122f0 in unknown procedure
82c8 in anonymous function
```

That's the same sort of failure as the RISC OS 5 test. Doing the same tests, we see...

```
*showregs
Register dump (stored at &02104FAC) is:
R0  = 00000000 R1  = 00000000 R2  = 00000000 R3  = 00000000
R4  = 00000000 R5  = 00000000 R6  = 00000000 R7  = 00000000
R8  = 00000000 R9  = 00000000 R10 = 00000000 R11 = 00000000
R12 = 00000000 R13 = 00000000 R14 = 00000000 R15 = 00000000
Mode USR flags set: nzcvif
```

So no state from the application.

```
*memoryi 21fefa8-20+40
021FEF88 : (ÅŸå : E59FC528 : LDR     R12,&021FF4B8
021FEF8C : .".å : E51A221C : LDR     R2,[R10,#-540]
021FEF90 : .ÀŒà : E08CC002 : ADD     R12,R12,R2
021FEF94 : .  ã : E3A02001 : MOV     R2,#1
021FEF98 : X#Ìå : E5CC239D : STRB    R2,[R12,#925]
021FEF9C : ..-é : E92D000A : STMDB   R13!,{R1,R3}
021FEFA0 : þ.‘é : E99103FE : LDMIB   R1,{R1-R9}
021FEFA4 : .>.ë : EB003E0D : BL      &0220E7E0
021FEFA8 : ‘..ê : EA000091 : B       &021FF1F4
021FEFAC : .@-é : E92D4001 : STMDB   R13!,{R0,R14}
021FEFB0 : .. á : E1A00002 : MOV     R0,R2
021FEFB4 : .T.ë : EB005408 : BL      &02213FDC
021FEFB8 : .P½è : E8BD5000 : LDMIA   R13!,{R12,R14}
021FEFBC : ..Œè : E88C0003 : STMIA   R12,{R0,R1}
021FEFC0 : .ð°á : E1B0F00E : MOVS    PC,R14
021FEFC4 : .. á : E1A00000 : MOV     R0,R0
```

So the failure inside SharedCLibrary is the same as well.


## RISC OS 4

What about RISC OS 4? Running the test on there...

```
*nantest
NaN test
nan = nan
nan = 7ff80000/e0000000
same = 1, different = 0
INF test
inf = inf
inf = 7ff00000/00000000
same = 1, different = 0
```

Which is unsurprising as it's using the same SharedCLibrary and FPEmulator module.


## What's wrong?

### On RISC OS 4

The FPEmulator is returning the wrong result for a comparison of NaN. Maybe there's a flag setting that I missed which is meant to address that. I'll need to look into it.

### On RISC OS 5

I don't know what's wrong. But something's broken there.
