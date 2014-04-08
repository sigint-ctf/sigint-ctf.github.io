---
layout: post
title: "Nuit Du Hack Quals 2014 Writeups"
description: ""
category: writeups
tags: [nuitduhack2014]
---
<!--{% include JB/setup %}-->

### Carbonara

We're provided the following cryptic string:

	%96 7=28 7@C E9:D 492= :D iQx>A6C2E@C xF=:FD r26D2C s:GFDQ]

Googling a few of these fragments turns up a multitude of news articles using a "tncms" package. [This](http://www.timesdispatch.com/news/local/central-virginia/a-pow-son-s--year-quest-finally-unfurls/article_3693659c-4788-516c-8c73-1c8defc13efa.html?mode=jqm) is one such page. Examining the source we find similar gibberish in the body of the article:
	 
	 <div class="encrypted-content" style="display: none">
	   <span class="paragraph4">
	       kAm{2?5CF&gt; 6G6?EF2==J &gt;256 :E 9@&gt;6] qFE E96 7=28 96 96=A65 E@ D64C6E=J 4C62E6 2?5 =2E6C H2G65 :? 2? :4@?:4 A9@E@8C2A9 E2&lt;6? 2D 96 2?5 9:D 76==@H !~(D H6C6 =:36C2E65 @? pF8] ah[ `hcd[ G2?:D965 @G6C E:&gt;6]k^Am
	   </span>
	 </div>

We also find a link to a `decrypt.js` link in the body of the article. Near the bottom is the following function:

    tncms.unscramble = function (sInput) {
        var sOutput = '';
        for (var i = 0, c = sInput.length; i < c; i++) {
            var nChar = sInput.charCodeAt(i);
            if (nChar >= 33 && nChar <= 126) {
                sTmp = String.fromCharCode(33 + (((nChar - 33) + 47) % 94));
                sOutput += sTmp
            } else {
                sOutput += sInput.charAt(i)
            }
        }
        return sOutput
    };
    
No need to reverse-engineer this; we can simply use it in the javascript console to unscramble our string:

    tncms.unscramble('%96 7=28 7@C E9:D 492= :D iQx>A6C2E@C xF=:FD r26D2C s:GFDQ]')
    "The flag for this chal is :"Imperator Iulius Caesar Divus"."

<!--more-->

### Onion Rings

The hidden service accepts a profile picture upload, and includes the option to load from a non-TOR URL. So, we can ask it to load from our server, and capture the IP of the requestor. 

As [before](http://sigint.ru/backdoor2014/backdoor2014.html#web100-1), we can listen on port 80 on a server and submit our server's URL.

The server's IP was 212.83.153.197. Visiting [http://212.83.153.197/](http://212.83.153.197/) and searching for `flag`, we find:

    The flag.. It is '0hSh1t1r4n0ut0fn00dl35'

### Windows Forensics

We are given a 400MB Windows pagefile. A few initial attempts along the lines of `strings pagefile.sys | grep flag` turned up quite a lot of results, but no interesting ones. Noticing several Chrome-related strings, we searched the file for URLs. Still, we found nothing interesting. 

Then, we reread the problem more carefully and saw that our task was to recover a command prompt session. Searching for command prompt pagefile.sys forensics yielded [this excellent guide](http://blog.roberthaist.com/2013/12/restoring-windows-cmd-sessions-from-pagefile-sys-2/). He uses `page_brute`, which splits the pagefile into 4096-byte pages and processes each with YARA. Installation of YARA proved the most difficult part of this challenge. Once YARA was installed, I added the following rule to page_brute's default_signatures.yar:

    rule CMDscan_Optimistic_Blanklines
    {
        meta:
            author="Robert Haist | @SleuthKid"
            description="Searching for blank lines and magic bytes that occur with paged cmd usage"
        strings:
            $m1 = { 63 3A 79 26 7B }
            $m2 = { 77 00 27 00 57 00 27 00 77 00 }
            $m3 = { 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 00 20 }
        condition:
            any of them
    }

Then, ran page_brute on pagefile.sys and reviewed the results using `strings -el CMDscan_Optimistic_Blanklines/*| sed -e "s/\s\{4,\}/\\n/g"`. Inside, we find:

      Microsoft Windows XP [version 5.1.2600]
      (C) Copyright 1985-2001 Microsoft Corp.
      C:\Documents and Settings\Administrateur>ncat 192.168.130.105 1234
      Username : JackTheRipper
      Password : 200020012002
      1337 - USER VALID
      1338 - 04c0f778e6dd6c0a
      1338 - 025e48c9f5f22f87
      close: No error

Neither the password nor either of the two hex strings were the flag, so we tried concatenating the two hex strings. `04c0f778e6dd6c0a025e48c9f5f22f87` was the flag. The lowercase flag format gave us a hint for Here Kitty Kitty.

### Here Kitty Kitty

In lieu of a writeup, we offer the following two images, and leave the solution as an exercise to the reader:

![waveform](/assets/images/nuitduhack2014/kitty-waveform.png)

![morse code](/assets/images/nuitduhack2014/morse.png)

Unfortunately, `5BC925649CB0188F52E617D70929191C` was not accepted. We tried HashCat dictionary and bruteforce attacks without success. After solving Windows Forensics, we tried submitting as lowercase, which was successful. Case-sensitivity isn't fun!

### BigMomma

Though we had the server binary, and briefly attempted to reverse it, we were able to identify how it worked by playing around with it for a few minutes.

       Please enter your username:
       a
       Nope (45)


       Please enter your username:
       b
       Nope (46)

So, if `a` returns 45 and `b` returns 46, what returns 0? 

    >>> chr(ord('a')-45)
    '4'

Let's try that:
     
     Please enter your username:
     4
     Nope (-100)

     
     Please enter your username:
     4a
     Nope (-3)


     Please enter your username:
     4d
     Nope (-77)
     
Apparently, we are provided the return value of the `strcmp` between our input and the correct username. So, we can derive the correct username one character at a time by converting the return value of the comparison between the terminating zero of our string, and the nonzero character of theirs. For example,

     a = "4d\0"
     b = "4dABCDEFGH"
     strcmp(a, b) == ord('\0')-ord('A')

Though a script ultimately would have been a better idea, we figured at this point that the username wasn't very long. So, we persevered manually with netcat and a Python interpreter. By this method, we discovered that the username was `4dM1N15TR4T0R`. Then, we had to find the password, for which we received the same feedback as before. In the end, we determined that the password was `THEpasswordISreallyLONGbutYOUllGETtoTHEendOFitEVENTUALLY`. 
       
    Please enter your username:
    4dM1N15TR4T0R
    Username correct, what is the password?THEpasswordISreallyLONGbutYOUllGETtoTHEendOFitEVENTUALLY
    Well done! Here is the flag: YoMamaIsLikeHTML,SmallHeadAndHugeBody
