+++
title = 'TryHackMe — Bounty Hacker Writeup'
date = '2023-02-11'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '“You talked a big game about being the most elite hacker in the solar system. Prove it and claim your right to the status of Elite Bounty Hacker!”'
[cover]
image = 'main.png'
+++

# Enumeration

Nmap scan shows us that **ssh** and **ftp** are running.

{{< figure src="image_1.png" align="center" >}}

# Getting the first flag

Connect to the machine via **ftp** and list the files using `dir`, we see two files which we can download by using `get`.

{{< figure src="image_2.png" align="center" >}}

Back on our attack box, we can start examining the files. `task.txt`’s author is **lin**.

{{< figure src="image_3.png" align="center" >}}

Now to the juicy part, **brute-forcing ssh using Hydra** and the `locks.txt` file we got via ftp.

{{< figure src="image_4.png" align="center" >}}

Now we can login to the machine via ssh using the information we have.

First flag is in `user.txt`.

{{< figure src="image_5.png" align="center" >}}

# Getting the last flag

Checking with `sudo -l`, we see that we can use `tar`.. interesting.

{{< figure src="image_6.png" align="center" >}}

A simple [google search](https://gtfobins.github.io/gtfobins/tar/) for “*gtfobins tar privesc*” gives us this command which will spawn a shell for us.

```bash
tar -cf /dev/null /dev/null --checkpoint=1 --checkpoint-action=exec=/bin/sh
```

And sure enough, we are root.
The last flag is in `/root/root.txt`.

{{< figure src="image_7.png" align="center" >}}
