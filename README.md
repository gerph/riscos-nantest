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

## RISC OS 5 SharedCLibrary running on Pyromaniac

Pyromaniac is for debugging, isn't it? So let's use it.

I saved took the C Library out of harddisc bundle on the website, and tried loading it with Pyromaniac. This fails because it's compressed. So I loaded it on RISC OS Classic and `*RMSave`'d it out.

```
pyrodev --common --command 'rmload clibRO5' --command 'pyromaniacdebug trace,traceregionfunc' --command aif32.nantest --config trace.stack_depth_indicator=yes 2> trace.txt
NaN test

C24
C22C28
C22C28
C25 -> [0xe1a0c00d 0xe92dd800 0xe24cb004 0xe15d000a]
C25 -> [0x69612e24 0x2e323366 0x746e616e 0x20747365]
C22C26
C22C27
```

There's no token expansion because there's no resources, and that version of the SCL assumes that there will be. It's not important though, because we've got a trace out of it. However when I trace this I find that the actual error being reported above is an error - 'Invalid Operation'.
In the trace it looks like this:

```
S[|||                     :  383bcf4: MOV     r0, sp                    ; R13 = &04107fc4
S[|||                     :  383bcf8: ADR     r1, &0383BCBC             ; -> "Resources:$.Resources.FPEmulator.Messages"
S[|||                     :  383bcfc: MOV     r2, #0
S[|||                     :  383bd00: SWI     XMessageTrans_OpenFile
S[|||                     :  383bd04: ADDVS   sp, sp, #&10              ; R13 = &04107fc4
S[|||                     :  383bd08: MOVVC   r0, r6                    ; R6 = &0383bc2c
S[|||                     :  383bd0c: MOV     pc, r7                    ; R7 = &0383bc94
S[|||                     :  383bc94: POPVS   {r2, r3, r4, r5, r6, r7, pc}
S[|||                     :  383bc98: MOV     r1, sp                    ; R13 = &04107fc4
S[|||                     :  383bc9c: MOV     r2, #0
S[|||                     :  383bca0: MOV     r5, #0
S[|||                     :  383bca4: MOV     r6, #0
S[|||                     :  383bca8: MOV     r7, #0
S[|||                     :  383bcac: SWI     XMessageTrans_ErrorLookup
S[|||                     :  383bcb0: MOV     r1, r4                    ; R4 = &00000000
S[|||                     :  383bcb4: BL      &0383BD10
S[|||                     :  383bd10: STMDB   sp!, {lr}                 ; R13 = &04107fc4
S[||||                    :  383bd14: MRS     r7, apsr
S[||||                    :  383bd18: MOV     r6, r0                    ; R0 = &0410a4e4
S[||||                    :  383bd1c: ADD     r0, sp, #4                ; R13 = &04107fc0
S[||||                    :  383bd20: SWI     XMessageTrans_CloseFile
S[||||                    :  383bd24: MSR     apsr_nzcvq, r7            ; R7 = &10000013
S[||||                    :  383bd28: LDMIA   sp!, {r7}                 ; R13 = &04107fc0
S[|||                     :  383bd2c: ADD     sp, sp, #&10              ; R13 = &04107fc4
S[||                      :  383bd30: MOV     r0, r6                    ; R6 = &0410a4e4
S[||                      :  383bd34: MOV     pc, r7                    ; R7 = &0383bcb8
S[||                      :  383bcb8: POP     {r2, r3, r4, r5, r6, r7, pc}
S[||                      :  383bc70: {DA 'ROM', module 'FPEmulator'}
S[|                       :  383bc70: SWI     OS_GenerateError
==== Begin error report ====
Error about to be reported to environment handler: [&80000200] Error 'InvalOp'
  r0  = &0410a4e4, r1  = &00000000, r2  = &00000002, r3  = &00009d80
  r4  = &0000b281, r5  = &ffffffff, r6  = &00000000, r7  = &00000000
  r8  = &000099cc, r9  = &00009bc4, r10 = &0000a4b4, r11 = &0000b1ec
  r12 = &0000b10c, sp  = &04107ff0, lr  = &0383bcb8, pc  = &0383bc74
  CPSR= &00000013 : SVC-32 ARM fi ae qvczn
  SPSR= &20000013 : SVC-32 ARM fi ae qvCzn
Locations:
  r0  -> [&80000200, &6f727245, &49272072, &6c61766e] in DA 'System heap'
  r3  -> [&00000000, &ffffffff, &fffffffd, &ffffffff] in DA 'Application Space'
  r8  -> "@@@@@@@@@AAAAA@@@@@@@@@@@@@@@@@@" in DA 'Application Space'
  r9  -> [&0000b750, &0000b764, &00000000, &00000000] in DA 'Application Space'
  r10 -> [&00000000, &00000000, &00000000, &00000000] in DA 'Application Space'
  r11 -> [&00008098, &0000b258, &0000b200, &0381ff70] in DA 'Application Space'
  r12 -> [&0000b52c, &00000001, &00009768, &0000b281] in DA 'Application Space'
  pc is DA 'ROM', module 'FPEmulator'
  lr is DA 'ROM', module 'FPEmulator'
==== End error report ====
 3831474: {DA 'ROM', module 'SharedCLibrary'}
 [                        :  3831474: SWI     XOS_EnterOS
 [                        :  3831478: {DA 'ROM', module 'SharedCLibrary'}
S[                        :  3831478: LDR     lr, [r0, #&120]           ; R0 = &00009420
S[                        :  383147c: TEQ     lr, #&80000000            ; R14 = &80000200
S[                        :  3831480: TEQNE   lr, #&80000001            ; R14 = &80000200
S[                        :  3831484: TEQNE   lr, #&80000002            ; R14 = &80000200
S[                        :  3831488: TEQNE   lr, #&80000003            ; R14 = &80000200
S[                        :  383148c: ADDNE   lr, r0, #&78              ; R0 = &00009420
S[                        :  3831490: STMNEIA lr!, {r0, r1, r2, r3, r4, r5, r6, r7, r8, r9, r10, r11, r12}  ; R14 = &00009498
```

It's as if the C Library is ignoring the attempt to change the signal handling to get to that point. From the earlier output from `*ShowRegs` we know that it's producing an invalid R1 value because the LDMIB instruction fails. So let's have a search for the LDMIB in the trace. It turns out there is only one with the right format:

```
 [|||||                   :  382c480: LDR     r2, [r10, #-&21c]         ; R10 = &0000a4b4
 [|||||                   :  382c484: ADD     r3, r2, r1                ; R2 = &fc7d65c8, R1 = &038337b8
 [|||||                   :  382c488: LDR     r1, [r3, r0, LSL #2]      ; R3 = &00009d80, R0 = &00000002
 [|||||                   :  382c48c: LDR     r2, &0382C468             ; = &ffffffff
 [|||||                   :  382c490: CMP     r1, r2                    ; R1 = &fffffffd, R2 = &ffffffff
 [|||||                   :  382c494: MOVNE   r0, #1
 [|||||                   :  382c498: MOVNE   pc, lr                    ; R14 = &0381c934
 [|||||                   :  381c934: CMP     r0, #0                    ; R0 = &00000001
 [|||||                   :  381c938: POP     {r0, r1, r3, lr}
 [|||                     :  381c93c: MOVEQ   r0, #0
 [|||                     :  381c940: MOVEQ   pc, lr                    ; R14 = &0383119c
 [|||                     :  381c944: LDR     r12, &0381CE7C            ; = &03833174
 [|||                     :  381c948: LDR     r2, [r10, #-&21c]         ; R10 = &0000a4b4
 [|||                     :  381c94c: ADD     r12, r12, r2              ; R12 = &03833174, R2 = &fc7d65c8
 [|||                     :  381c950: MOV     r2, #1
 [|||                     :  381c954: STRB    r2, [r12, #&39d]          ; R2 = &00000001, R12 = &0000973c
 [|||                     :  381c958: PUSH    {r1, r3}
 [||||                    :  381c95c: LDMIB   r1, {r1, r2, r3, r4, r5, r6, r7, r8, r9}  ; R1 = &0000b184
 [||||                    :  381c960: BL      &0382C3EC
 [||||                    :  382c3ec: MOV     r12, sp                   ; R13 = &0000b118
 [||||                    :  382c3f0: PUSH    {r11, r12, lr, pc}
```

So in Pyromaniac, it's getting r1 correct. The fact that higher up the code there's a comparison where R1 contained `&FFFFFFFD` - the value that we failed with - is interesting. I don't see how it could have reached the `LDMIB`, but that's the sequence of code when it's working.

Let's go back to the code that handled the signal.

Here's the `signal` code when executed in the RISC OS 5 SCL:

```
 [|||||||                 :     80b4: BL      &0000901C
 [|||||||                 :     901c: LDR     pc, &00009300             ; = &0382c4a4
 [|||||||                 :  382c4a4: {DA 'ROM', module 'SharedCLibrary'}
 [|||||||                 :  382c4a4: MOV     r2, r0                    ; R0 = &00000002
 [|||||||                 :  382c4a8: CMP     r0, #0                    ; R0 = &00000002
 [|||||||                 :  382c4ac: BLE     &0382C4CC
 [|||||||                 :  382c4b0: CMP     r2, #&b                   ; R2 = &00000002, #11
 [|||||||                 :  382c4b4: LDRLT   r0, &0382C464             ; = &038337b8
 [|||||||                 :  382c4b8: LDRLT   r3, [r10, #-&21c]         ; R10 = &0000a4b4
 [|||||||                 :  382c4bc: ADDLT   r3, r3, r0                ; R3 = &fc7d65c8, R0 = &038337b8
 [|||||||                 :  382c4c0: LDRLT   r0, [r3, r2, LSL #2]      ; R3 = &00009d80, R2 = &00000002
 [|||||||                 :  382c4c4: STRLT   r1, [r3, r2, LSL #2]      ; R1 = &fffffffd, R3 = &00009d80, R2 = &00000002
 [|||||||                 :  382c4c8: MOVLT   pc, lr                    ; R14 = &000080b8
 [|||||||                 :     80b8: {DA 'Application Space'}
```

And that same code on the RISC OS 4 SCL:

```
 [|||||||                 :     80b4: BL      &0000901C
 [|||||||                 :     901c: LDR     pc, &00009300             ; = &0382831c
 [|||||||                 :  382831c: {DA 'ROM', module 'SharedCLibrary': Function signal}
 [|||||||                 : Function: signal
 [|||||||                 :   r0  = &00000002, r1  = &fffffffd, r2  = &00000009, r3  = &00000001
 [|||||||                 :   r4  = &0000b281, r5  = &00000010, r6  = &ffffffff, r7  = &00000000
 [|||||||                 :   r8  = &00000000, r9  = &ffffffff, r10 = &0000a4b4, r11 = &0000b1fc
 [|||||||                 :   r12 = &fc7d755c, sp  = &0000b1d8, lr  = &000080b8, pc  = &0382831c
 [|||||||                 :   CPSR= &60000010 : USR-32 ARM fi ae qvCZn
 [|||||||                 :  382831c: MOV     r12, sp                   ; Function: signal  ; R13 = &0000b1d8
 [|||||||                 :  3828320: PUSH    {r0, r1, r4, r5, r11, r12, lr, pc}
 [||||||||                :  3828324: SUB     r11, r12, #4              ; R12 = &0000b1d8
 [||||||||                :  3828328: CMP     sp, r10                   ; R13 = &0000b1b8, R10 = &0000a4b4
 [||||||||                :  382832c: BLLT    &0381D310
 [||||||||                :  3828330: MOV     r4, r1                    ; R1 = &fffffffd
 [||||||||                :  3828334: CMP     r0, #0                    ; R0 = &00000002
 [||||||||                :  3828338: BLE     &03828344
 [||||||||                :  382833c: CMP     r0, #&b                   ; R0 = &00000002, #11
 [||||||||                :  3828340: BLT     &03828350
 [||||||||                :  3828350: LDR     r1, &0382827C             ; = &03832894
 [||||||||                :  3828354: LDR     r12, [r10, #-&21c]        ; R10 = &0000a4b4
 [||||||||                :  3828358: ADD     r1, r12, r1               ; R12 = &fc7d755c, R1 = &03832894
 [||||||||                :  382835c: LDR     r5, [r1, r0, LSL #2]      ; R1 = &00009df0, R0 = &00000002
 [||||||||                :  3828360: STR     r4, [r1, r0, LSL #2]      ; R4 = &fffffffd, R1 = &00009df0, R0 = &00000002
 [||||||||                :  3828364: TEQ     r0, #2                    ; R0 = &00000002
 [||||||||                :  3828368: BNE     &038283AC
 [||||||||                :  382836c: BL      &0382F408                 ; -> Function: _kernel_fpavailable
 [||||||||                :  382f408: {DA 'ROM', module 'SharedCLibrary': Function _kernel_fpavailable}
 [||||||||                : Function: _kernel_fpavailable
 [||||||||                :   r0  = &00000002, r1  = &00009df0, r2  = &00000009, r3  = &00000001
 [||||||||                :   r4  = &fffffffd, r5  = &ffffffff, r6  = &ffffffff, r7  = &00000000
 [||||||||                :   r8  = &00000000, r9  = &ffffffff, r10 = &0000a4b4, r11 = &0000b1d4
 [||||||||                :   r12 = &fc7d755c, sp  = &0000b1b8, lr  = &03828370, pc  = &0382f408
 [||||||||                :   CPSR= &40000010 : USR-32 ARM fi ae qvcZn
 [||||||||                :  382f408: STR     lr, [sp, #-4]!            ; Function: _kernel_fpavailable  ; R14 = &03828370, R13 = &0000b1b8
 [|||||||||               :  382f40c: MRS     lr, apsr
 [|||||||||               :  382f410: STR     lr, [sp, #-4]!            ; R14 = &40000010, R13 = &0000b1b4
 [||||||||||              :  382f414: LDR     r12, &0382F110            ; = &03831ec4
 [||||||||||              :  382f418: LDR     r0, [r10, #-&21c]         ; R10 = &0000a4b4
 [||||||||||              :  382f41c: ADD     r12, r12, r0              ; R12 = &03831ec4, R0 = &fc7d755c
 [||||||||||              :  382f420: LDRB    r0, [r12, #&118]          ; R12 = &00009420 ([&00008080, &000093e4, &0000940c, &e92d5001])
 [||||||||||              :  382f424: POP     {lr}
 [|||||||||               :  382f428: MSR     apsr_nzcvq, lr            ; R14 = &40000010
 [|||||||||               :  382f42c: POP     {pc}
 [|||||||||               :  3828370: {DA 'ROM', module 'SharedCLibrary'}
 [||||||||                :  3828370: TEQ     r0, #0                    ; R0 = &00000001
 [||||||||                :  3828374: BEQ     &038283AC
 [||||||||                :  3828378: LDR     r0, &03827F3C             ; = &fffffffd
 [||||||||                :  382837c: TEQ     r4, r0                    ; R4 = &fffffffd, R0 = &fffffffd
 [||||||||                :  3828380: BNE     &03828398
 [||||||||                :  3828384: TEQ     r5, r0                    ; R5 = &ffffffff, R0 = &fffffffd
 [||||||||                :  3828388: MOVNE   r1, #&70000               ; #458752
 [||||||||                :  382838c: MOVNE   r0, #0
 [||||||||                :  3828390: BNE     &038283A8
 [||||||||                :  38283a8: BL      &0382C1C0
 [||||||||                :  382c1c0: MRC     p1, #1, r2, c0, c0, #0
 [||||||||                :  3834108: {DA 'ROM', module 'FPEmulator'}
U[                        :  3834108: SUB     sp, sp, #&40              ; R13 = &08404000
U[|                       :  383410c: STMIA   sp!, {r0, r1, r2, r3, r4, r5, r6, r7, r8, r9, r10, r11, r12}  ; R13 = &08403fc0
U[|                       :  3834110: STMIA   sp, {sp, lr} ^            ; R13 = &08403ff4
U[|                       :  3834114: MOV     r0, r0                    ; R0 = &00000000
U[|                       :  3834118: SUB     sp, sp, #&34              ; R13 = &08403ff4
U[)|                      :  383411c: STR     lr, [sp, #&3c]            ; R14 = &0382c1c4, R13 = &08403fc0
U[)|                      :  3834120: MOV     r12, sp                   ; R13 = &08403fc0
U[)|                      :  3834124: MRS     r10, apsr
U[)|                      :  3834128: MRS     r9, spsr
U[)|                      :  383412c: PUSH    {r9, r10}
U[)||                     :  3834130: TST     r9, #&f                   ; R9 = &00000010, #15
U[)||                     :  3834134: SUB     r9, lr, #4                ; R14 = &0382c1c4
U[)||                     :  3834138: LDRTEQ  r11, [r9], #0             ; R9 = &0382c1c0
U[)||                     :  383413c: LDRNE   r11, [r9]                 ; R9 = &0382c1c0
U[)||                     :  3834140: MOV     r10, #0
U[)||                     :  3834144: LDR     r10, [r10, #&ff4]         ; R10 = &00000000
U[)||                     :  3834148: TST     r11, #&8000000            ; R11 = &ee302110, #134217728
U[)||                     :  383414c: BEQ     &03835830
U[)||                     :  3834150: AND     r9, r11, #&f00            ; R11 = &ee302110
U[)||                     :  3834154: CMP     r9, #&100                 ; R9 = &00000100, #256
U[)||                     :  3834158: BNE     &03835444
U[)||                     :  383415c: LDR     r8, [r10, #&84]           ; R10 = &07001814
U[)||                     :  3834160: TEQ     r8, #0                    ; R8 = &00000000
U[)||                     :  3834164: BNE     &0383542C
U[)||                     :  3834168: MRS     r8, apsr
U[)||                     :  383416c: BIC     r8, r8, #&80              ; R8 = &6000009b
U[)||                     :  3834170: MSR     cpsr_fsxc, r8             ; R8 = &6000001b
U[)||                     :  3834174: MOV     r0, r0                    ; R0 = &00000000
U[)||                     :  3834178: LDR     r7, [r10, #&80]           ; R10 = &07001814
U[)||                     :  383417c: TST     r11, #&2000000            ; R11 = &ee302110, #33554432
U[)||                     :  3834180: BNE     &03835140
U[)||                     :  3835140: TST     r11, #&80000              ; R11 = &ee302110, #524288
U[)||                     :  3835144: TSTNE   r11, #&80                 ; R11 = &ee302110, #128
U[)||                     :  3835148: BNE     &03835830
U[)||                     :  383514c: TST     r11, #&10                 ; R11 = &ee302110, #16
U[)||                     :  3835150: BNE     &03835230
U[)||                     :  3835230: AND     r8, r11, #&f00000         ; R11 = &ee302110
U[)||                     :  3835234: ADD     pc, pc, r8, LSR #18       ; R15 = &03835234, R8 = &00300000
U[)||                     :  3835248: B       &038352E0
U[)||                     :  38352e0: MOV     r6, r7                    ; R7 = &01070000
U[)||                     :  38352e4: AND     r3, r11, #&f000           ; R11 = &ee302110
U[)||                     :  38352e8: CMP     r3, #&d000                ; R3 = &00002000, #53248
U[)||                     :  38352ec: BHS     &038352F8
U[)||                     :  38352f0: STR     r6, [r12, r3, LSR #10]    ; R6 = &01070000, R12 = &08403fc0, R3 = &00002000
U[)||                     :  38352f4: B       &03835008
U[)||                     :  3835008: STR     r7, [r10, #&80]           ; R7 = &01070000, R10 = &07001814
U[)||                     :  383500c: LDR     r8, [r12, #-8]            ; R12 = &08403fc0
U[)||                     :  3835010: ANDS    r8, r8, #&f               ; R8 = &00000010
U[)||                     :  3835014: BNE     &03834300
U[)||                     :  3835018: LDR     r9, [r12, #&3c]           ; R12 = &08403fc0
U[)||                     :  383501c: CMP     sp, #0                    ; R13 = &08403fb8
U[)||                     :  3835020: LDRTNE  r11, [r9], #0             ; R9 = &0382c1c4
U[)||                     :  3835024: ANDNE   r8, r11, r11, LSL #1      ; R11 = &e1c23001, R11 = &e1c23001
U[)||                     :  3835028: TSTNE   r8, #&8000000             ; R8 = &c1802000, #134217728
U[)||                     :  383502c: BLNE    &03835798
U[)||                     :  3835030: LDMDB   r12, {r8, r9}             ; R12 = &08403fc0
U[)||                     :  3835034: MSR     cpsr_fsxc, r9             ; R9 = &0000009b
U[)||                     :  3835038: MSR     spsr_fsxc, r8             ; R8 = &00000010
U[)||                     :  383503c: MOV     sp, r12                   ; R12 = &08403fc0
U[)|                      :  3835040: LDMIA   r12, {r0, r1, r2, r3, r4, r5, r6, r7, r8, r9, r10, r11, r12, sp, lr} ^  ; R12 = &08403fc0
U[)|                      :  3835044: MOV     r0, r0                    ; R0 = &00000000
U[)|                      :  3835048: ADD     sp, sp, #&3c              ; R13 = &08403fc0
U[)                       :  383504c: LDMIA   sp!, {pc} ^               ; R13 = &08403ffc
U[)                       :  382c1c4: {DA 'ROM', module 'SharedCLibrary'}
 [||||||||                :  382c1c4: BIC     r3, r2, r1                ; R2 = &01070000, R1 = &00070000
 [||||||||                :  382c1c8: EOR     r3, r3, r0                ; R3 = &01000000, R0 = &00000000
 [||||||||                :  382c1cc: MCR     p1, #1, r3, c0, c0, #0
 [||||||||                :  3834108: {DA 'ROM', module 'FPEmulator'}
U[                        :  3834108: SUB     sp, sp, #&40              ; R13 = &08404000
U[|                       :  383410c: STMIA   sp!, {r0, r1, r2, r3, r4, r5, r6, r7, r8, r9, r10, r11, r12}  ; R13 = &08403fc0
U[|                       :  3834110: STMIA   sp, {sp, lr} ^            ; R13 = &08403ff4
U[|                       :  3834114: MOV     r0, r0                    ; R0 = &00000000
U[|                       :  3834118: SUB     sp, sp, #&34              ; R13 = &08403ff4
U[)|                      :  383411c: STR     lr, [sp, #&3c]            ; R14 = &0382c1d0, R13 = &08403fc0
U[)|                      :  3834120: MOV     r12, sp                   ; R13 = &08403fc0
U[)|                      :  3834124: MRS     r10, apsr
U[)|                      :  3834128: MRS     r9, spsr
U[)|                      :  383412c: PUSH    {r9, r10}
U[)||                     :  3834130: TST     r9, #&f                   ; R9 = &00000010, #15
U[)||                     :  3834134: SUB     r9, lr, #4                ; R14 = &0382c1d0
U[)||                     :  3834138: LDRTEQ  r11, [r9], #0             ; R9 = &0382c1cc
U[)||                     :  383413c: LDRNE   r11, [r9]                 ; R9 = &0382c1cc
U[)||                     :  3834140: MOV     r10, #0
U[)||                     :  3834144: LDR     r10, [r10, #&ff4]         ; R10 = &00000000
U[)||                     :  3834148: TST     r11, #&8000000            ; R11 = &ee203110, #134217728
U[)||                     :  383414c: BEQ     &03835830
U[)||                     :  3834150: AND     r9, r11, #&f00            ; R11 = &ee203110
U[)||                     :  3834154: CMP     r9, #&100                 ; R9 = &00000100, #256
U[)||                     :  3834158: BNE     &03835444
U[)||                     :  383415c: LDR     r8, [r10, #&84]           ; R10 = &07001814
U[)||                     :  3834160: TEQ     r8, #0                    ; R8 = &00000000
U[)||                     :  3834164: BNE     &0383542C
U[)||                     :  3834168: MRS     r8, apsr
U[)||                     :  383416c: BIC     r8, r8, #&80              ; R8 = &6000009b
U[)||                     :  3834170: MSR     cpsr_fsxc, r8             ; R8 = &6000001b
U[)||                     :  3834174: MOV     r0, r0                    ; R0 = &00000000
U[)||                     :  3834178: LDR     r7, [r10, #&80]           ; R10 = &07001814
U[)||                     :  383417c: TST     r11, #&2000000            ; R11 = &ee203110, #33554432
U[)||                     :  3834180: BNE     &03835140
U[)||                     :  3835140: TST     r11, #&80000              ; R11 = &ee203110, #524288
U[)||                     :  3835144: TSTNE   r11, #&80                 ; R11 = &ee203110, #128
U[)||                     :  3835148: BNE     &03835830
U[)||                     :  383514c: TST     r11, #&10                 ; R11 = &ee203110, #16
U[)||                     :  3835150: BNE     &03835230
U[)||                     :  3835230: AND     r8, r11, #&f00000         ; R11 = &ee203110
U[)||                     :  3835234: ADD     pc, pc, r8, LSR #18       ; R15 = &03835234, R8 = &00200000
U[)||                     :  3835244: B       &03835334
U[)||                     :  3835334: AND     r3, r11, #&f000           ; R11 = &ee203110
U[)||                     :  3835338: CMP     r3, #&d000                ; R3 = &00003000, #53248
U[)||                     :  383533c: BHS     &03835350
U[)||                     :  3835340: LDR     r6, [r12, r3, LSR #10]    ; R12 = &08403fc0, R3 = &00003000
U[)||                     :  3835344: BIC     r6, r6, #&ff000000        ; R6 = &01000000
U[)||                     :  3835348: ORR     r7, r6, #&1000000         ; R6 = &00000000
U[)||                     :  383534c: B       &03835008
U[)||                     :  3835008: STR     r7, [r10, #&80]           ; R7 = &01000000, R10 = &07001814
U[)||                     :  383500c: LDR     r8, [r12, #-8]            ; R12 = &08403fc0
U[)||                     :  3835010: ANDS    r8, r8, #&f               ; R8 = &00000010
U[)||                     :  3835014: BNE     &03834300
U[)||                     :  3835018: LDR     r9, [r12, #&3c]           ; R12 = &08403fc0
U[)||                     :  383501c: CMP     sp, #0                    ; R13 = &08403fb8
U[)||                     :  3835020: LDRTNE  r11, [r9], #0             ; R9 = &0382c1d0
U[)||                     :  3835024: ANDNE   r8, r11, r11, LSL #1      ; R11 = &e1a00002, R11 = &e1a00002
U[)||                     :  3835028: TSTNE   r8, #&8000000             ; R8 = &c1000000, #134217728
U[)||                     :  383502c: BLNE    &03835798
U[)||                     :  3835030: LDMDB   r12, {r8, r9}             ; R12 = &08403fc0
U[)||                     :  3835034: MSR     cpsr_fsxc, r9             ; R9 = &0000009b
U[)||                     :  3835038: MSR     spsr_fsxc, r8             ; R8 = &00000010
U[)||                     :  383503c: MOV     sp, r12                   ; R12 = &08403fc0
U[)|                      :  3835040: LDMIA   r12, {r0, r1, r2, r3, r4, r5, r6, r7, r8, r9, r10, r11, r12, sp, lr} ^  ; R12 = &08403fc0
U[)|                      :  3835044: MOV     r0, r0                    ; R0 = &00000000
U[)|                      :  3835048: ADD     sp, sp, #&3c              ; R13 = &08403fc0
U[)                       :  383504c: LDMIA   sp!, {pc} ^               ; R13 = &08403ffc
U[)                       :  382c1d0: {DA 'ROM', module 'SharedCLibrary'}
 [||||||||                :  382c1d0: MOV     r0, r2                    ; R2 = &01070000
 [||||||||                :  382c1d4: TEQ     r0, r0                    ; R0 = &01070000, R0 = &01070000
 [||||||||                :  382c1d8: TEQ     pc, pc                    ; R15 = &0382c1d8, R15 = &0382c1d8
 [||||||||                :  382c1dc: MOVSNE  pc, lr                    ; R14 = &038283ac
 [||||||||                :  382c1e0: MOV     pc, lr                    ; R14 = &038283ac
 [||||||||                :  38283ac: MOV     r0, r5                    ; R5 = &ffffffff
 [||||||||                :  38283b0: LDMDB   r11, {r4, r5, r11, sp, pc}  ; R11 = &0000b1d4
 [||||||||                :     80b8: {DA 'Application Space'}
```

So that's a lot longer - mostly because it's got the code from the undefined instruction handler (FPEmulator) in the middle where we update the FP flags. That code is executing in UND mode, indicated by the '`U`' at the start of the line.

That's why the RISC OS 5 CLibrary went into the invalid operation error code. And for some reason that invalid operation code caused exceptions. But only on actual RISC OS 5, and possibly only on RPCEmu.

## What's wrong?

### On RISC OS 4

The FPEmulator is returning the wrong result for a comparison of NaN. Maybe there's a flag setting that I missed which is meant to address that. I'll need to look into it.

### On RISC OS 5

I don't know what's wrong. At the very least it's looking like the C library isn't disabling signals when you request it. And then, whilst handling it, something is going wrong.
