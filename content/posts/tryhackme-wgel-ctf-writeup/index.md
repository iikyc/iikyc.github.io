+++
title = 'TryHackMe — Wgel CTF Writeup'
date = '2023-04-09'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '“Can you exfiltrate the root flag?”'
[cover]
image = 'main.png'
+++

# Enumeration

{{< figure src="image_1.png" align="center" >}}

Performing an Nmap scan shows two open ports; **80** and **22**, meaning that the machine is running a web server and ssh.

Navigating to the website gives us a default **Apache2** web page.

{{< figure src="image_2.png" align="center" >}}

Nothing interesting there, however once we check the page’s source code, we find the name **jessie** in an HTML comment, which could help us login through ssh.

{{< figure src="image_3.png" align="center" >}}

After enumerating the website using **ffuf**, we get one page; `/sitemap`.

{{< figure src="image_4.png" align="center" >}}

But after checking it out, there seems to be nothing of interest to us on the webpage.

{{< figure src="image_5.png" align="center" >}}

So, let’s enumerate this page using **dirb**.

{{< figure src="image_6.png" align="center" >}}

This gives us one very interesting directory; `/.ssh`.

{{< figure src="image_7.png" align="center" >}}

We got an **id_rsa** file, let’s download it so we can use it to ssh into the machine as the user jessie.

But before we can use it, we need to adjust its permissions to **600**.

{{< figure src="image_8.png" align="center" >}}

{{< figure src="image_9.png" align="center" >}}

We have a shell.

# First flag

The user flag is in the `/home/jessie/Documents` directory.

{{< figure src="image_10.png" align="center" >}}

# Final flag

Once we check what we can run with **sudo**, we see **wget**.

{{< figure src="image_11.png" align="center" >}}

Referring to [gtfobins](https://gtfobins.github.io/gtfobins/wget/), we can use wget with sudo to send a file to our attacker machine. In this case, we’ll use it to send the root flag.

Start a Netcat listener on the attacker machine.

{{< figure src="image_12.png" align="center" >}}

And on jessie’s machine, we can use `wget` to send the file `/root/root_flag.txt`.

{{< figure src="image_13.png" align="center" >}}

Back on our attacker machine, we get the root flag.

{{< figure src="image_14.png" align="center" >}}