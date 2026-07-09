+++
title = 'TryHackMe — Boiler CTF Writeup'
date = '2023-10-18'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '''"Intermediate level CTF. Just enumerate, you'll get there."'''
[cover]
image = 'main.png'
+++

# Nmap

After scanning with Nmap, we discover four open ports:
- **21** — FTP
- **80** — HTTP
- **10000** — HTTP
- **55007** — SSH

{{< figure src="image_1.png" align="center" >}}

Nmap indicates that anonymous FTP login is allowed.

# Getting the user flag
After logging in via FTP, we find a text file with jumbled characters.

To save you time, they are meaningless.

{{< figure src="image_2.png" align="center" >}}

Moving on to the web app on port 80, it displays the default **Apache2** page.

{{< figure src="image_3.png" align="center" >}}

So, we’ll start enumerating the website using **ffuf**.

{{< figure src="image_4.png" align="center" >}}

As you can see, the site uses **Joomla CMS**, which can be accessed at `/joomla`.

Unfortunately though, nothing is of interest to us here.

{{< figure src="image_5.png" align="center" >}}

Next, I tried to enumerate the `/joomla` directory with using **dirb**.

As you can see, most of the discovered directories do not stand out as interesting.. except for `/_test`.

{{< figure src="image_6.png" align="center" >}}

After visiting `/_test`, I saw that “*sar2html*” is mentioned, and after googling what it was, the first indexed result was an [RCE exploit](https://www.exploit-db.com/exploits/47204).

```
# Exploit Title: sar2html Remote Code Execution
# Date: 01/08/2019
# Exploit Author: Furkan KAYAPINAR
# Vendor Homepage:https://github.com/cemtan/sar2html
# Software Link: https://sourceforge.net/projects/sar2html/
# Version: 3.2.1
# Tested on: Centos 7
In web application you will see index.php?plot url extension.
http://<ipaddr>/index.php?plot=;<command-here> will execute
the command you entered. After command injection press "select # host" then your command's
output will appear bottom side of the scroll screen.
```

As you can see, we can pass a command in the `plot` parameter that shows up after clicking **New** on the webpage.

{{< figure src="image_7.png" align="center" >}}

And just like that, we have RCE, now we can check the contents of `log.txt`.

{{< figure src="image_8.png" align="center" >}}

The file is apparently an SSH log, and we get the credentials for the user **basterd**.

And after logging in via ssh, we discover a file called `backup.sh` which contains the credentials for another user; **stoner**.

{{< figure src="image_9.png" align="center" >}}

After switching users, we find the first flag in `/home/stoner`.

{{< figure src="image_10.png" align="center" >}}

# Getting the root flag

Checking what we can run with `sudo -l`, we find nothing.

However, using `find` to check which files have the **SUID bit** set, we find… `find`.

{{< figure src="image_11.png" align="center" >}}

We can find a method to spawn a shell with `find` on [GTFOBins](https://gtfobins.github.io/gtfobins/find/).

After escalating to root, we can find the root flag in `/root`.

{{< figure src="image_12.png" align="center" >}}