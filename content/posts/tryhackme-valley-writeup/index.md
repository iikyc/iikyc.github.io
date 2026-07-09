+++
title = 'TryHackMe — Valley Writeup'
date = '2023-10-16'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '"Can you find your way into the Valley?"'
[cover]
image = 'main.png'
+++

# Nmap

{{< figure src="image_1.png" align="center" >}}

Starting off with the Nmap scan, we see three open ports:
- **22** — ssh
- **80** — http
- **37370** — FTP

# Getting the first flag

{{< figure src="image_2.png" align="center" >}}

When we visit the web application port **80**, we see a pretty old looking website which has a bunch of images on it, as well as a pricing page.

The first thing that came to mind was enumerating the directories.

{{< figure src="image_3.png" align="center" >}}

Using ffuf, we discover:

```
/gallery
/static
/pricing
```

It’s always a good step to re-enumerate any directories you find, so I ran **ffuf** on `/static` again..

{{< figure src="image_4.png" align="center" >}}

As you can see, we get all the images in `/static`, however, `00` is not present in the `/gallery` webpage, as the first image is `1`.

{{< figure src="image_5.png" align="center" >}}

Checking `/static/00`, we find this note that also reveals a hidden directory.

{{< figure src="image_6.png" align="center" >}}

{{< figure src="image_7.png" align="center" >}}

The hidden page is a login portal for the site’s developers, after digging around some more, I found hardcoded credentials in a JavaScript file.

{{< figure src="image_8.png" align="center" >}}

Using those credentials, we get redirected to another helpful note on the site.

{{< figure src="image_9.png" align="center" >}}

“*Stop reusing credentials*” is our hint here.

We can use the same credentials over FTP.

From there, we can use `mget` to download all the PCAP files.

{{< figure src="image_10.png" align="center" >}}

After checking each PCAP file with **Wireshark**, we stumble upon credentials passed in a POST request.

{{< figure src="image_11.png" align="center" >}}

Now, using those credentials, we can connect to the machine via **SSH**, and get the first flag.

{{< figure src="image_12.png" align="center" >}}

# Getting the root flag

{{< figure src="image_13.png" align="center" >}}

Checking sudo privs yields nothing, but there is an interesting executable file in `/home`.

By setting up a simple HTTP server using Python on the machine, we can use `wget` to download the executable.

{{< figure src="image_14.png" align="center" >}}

Using a `strings` dump, we discover what looks like a hash.

{{< figure src="image_15.png" align="center" >}}

You can use [CrackStation](https://crackstation.net/) to crack this MD5 hash, which will give us the password for the other user on the machine, **valley**.

{{< figure src="image_16.png" align="center" >}}

After switching to the user **valley**, using `find` we can see which files are owned by the **valleyAdmin** group.
The `base64.py` file is owned by **valleyAdmin**, and it’s used in the cronjob that runs every 1 minute.

{{< figure src="image_17.png" align="center" >}}

So, all we have got to do is modify the `base64.py` file to escalate our privileges by making it set the **SUID bit** on `/bin/bash`.

{{< figure src="image_18.png" align="center" >}}

And by executing `bash -p`, we are root, as we have the effective user ID of root.

{{< figure src="image_19.png" align="center" >}}