---
layout: post
title:  "OverTheWire \"Leviathan\" writeup"
date:   2021-10-05 22:26:00 +0200
categories: over-the-wire
---
## DISCLAIMER

*Because it's not a real writeup if it doesn't start with a disclaimer.*\
\
This writeup is not intended to ruin the game to anyone. Passwords are omitted in the whole document, and published in a table you can find at the and of the page.\
If you haven't tried to solve the game on your own, try to follow the hints in the next paragraphs before following the easy route.\
\
*Link to the [game](https://overthewire.org/wargames/leviathan/).*

## THE RED THREAD

There is a common theme between most of the challenges: trying to understand how an executable does what it is supposed to do to exploit it and gain access to the next user to print the relative password stored under `/etc/leviathan_pass`.\
When in doubt, basically, google every single system call used to find an exploit, try something, and run often `whoami`.

## SPOILER-FREE SOLUTIONS

# Leviathan 0:

ssh into the machine.
```bash
ssh -p 2223 leviathan0@leviathan.labs.overthewire.org
```
# Leviathan 0 -> Leviathan 1:

In the home directory for leviathan0, there is a ".backup" folder. Only content is a bookmarks.html file.
Running `cat .backup/bookmarks.html | grep password` gives as output:

```html
<dt>
  <a
    href="http://leviathan.labs.overthewire.org/passwordus.html | This will be fixed later, the password for leviathan1 is XXXXXXXXXX"
    ADD_DATE="1155384634"
    LAST_CHARSET="ISO-8859-1"
    id="rdf:#$2wIU71"
    >password to leviathan1</a
  >
</dt>
```

# Leviathan 1 -> Leviathan 2:

In the home directory for leviathan1, there is only an executable file "check". Launching it will prompt for a password.
Checked the output of `strings` on the file and tried a couple of strings that made sense (namely: "love" and "pass"), with no success.
File is a Suid file. Checked the file using `strace`, but nothing interesting was found out. Tried then with `ltrace` and...

# Leviathan 2 -> Leviathan 3:

This one was a FUCKING BANGER. It took me two days of trial and error, and finally I got it.\
Basically I created a file under a tmp folder and launched `printfile` on it. It just printed the file. How smart.\
I was almost giving up, but then, there it was. `ltrace` saved the day (again).

```bash
leviathan2@leviathan:~$ ltrace ./printfile /tmp/directory/test 
__libc_start_main(0x804852b, 2, 0xffffd774, 0x8048610 <unfinished ...>
access("/tmp/directory/test", 4)                                                     = 0
snprintf("/bin/cat /tmp/directory/test", 511, "/bin/cat %s", "/tmp/directory/test")  = 24
geteuid()                                                                            = 12002
geteuid()                                                                            = 12002
setreuid(12002, 12002)                                                               = 0
system("/bin/cat /tmp/directory/test" <no return ...>
--- SIGCHLD (Child exited) ---
<... system resumed> )                                                               = 0
+++ exited (status 0) +++
```
At first I thought that I could try using a payload for the `snprintf` system call. Tried stupidly big files, binary files, code injection. Nothing worked.\
And then I googled "`access` system call vulnerability", and there it was. I am going to let you google it. Long story short, the `access` system call is so vulnerable that **even the relative man page discourages its use**.\
I created a file. It was called "file ; bash". Printed it with `printfile`, and got no such file or directory.
I thought I got another dead end, but I couldn't modify any of my files. Decided to see the output of `whoami` and I was, in fact, leviathan3...

# Leviathan 3 -> Leviathan 4:

Basically the same as 1 -> 2. Used `ltrace` on the binary file and found the password...

# Leviathan 4 -> Leviathan 5:

This one was easy. In the home of the level, there is an hidden folder: `.trash`. Inside is an executable that once ran, prints a series of binary numbers. Using our lord and saviour `ltrace` shows that it is displaying the content of a familiar face...

# Leviathan 5 -> Leviathan 6:

Same as above, I ran `ltrace` on `./leviathan5`, and found that it was doing an `fopen` on the file "/tmp/file.log". Symlinked the password in /etc/leviathan_pass/leviathan6 to /tmp/file.log, and got the password I was looking for...

# Leviathan 6 -> Leviathan 7:

This one needs programming. In the home directory there was an executable. It required a 4 digit number. You know what that means? BRUTE-FORCE TIME BABY!
Came up with this: 
```bash
#!/bin/bash

for i in {0000..9999} 
do
    ~/leviathan6 $i
done
```
It took a while (in fact, a long while) to get a shell where I was user leviathan7, but it worked...

## PASSWORDS (SPOILERS)

*You have been warned. Scroll up or admit defeat.*\
\
*You wanted to conquer the leviathan, but you only got the L.*\
\
*As you wish.*

| Level          | Password   |
| -------------- | ---------- |
| Lev 0          | leviathan0 |
| Lev 0 -> Lev 1 | rioGegei8m |
| Lev 1 -> Lev 2 | ougahZi8Ta |
| Lev 2 -> Lev 3 | Ahdiemoo1j |
| Lev 3 -> Lev 4 | vuH0coox6m |
| Lev 4 -> Lev 5 | Tith4cokei |
| Lev 5 -> Lev 6 | UgaoFee4li |
| Lev 6 -> Lev 7 | ahy7MaeBo9 |
