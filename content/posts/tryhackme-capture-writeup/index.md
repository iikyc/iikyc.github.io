+++
title = 'TryHackMe — Capture! Writeup'
date = '2023-05-07'
draft = false
tags = ["TryHackMe", "Linux"]
summary = '“Can you bypass the login form?”'
[cover]
image = 'main.png'
+++

{{< figure src="image_1.png" caption="Login form in original state" align="center" >}}

Once the machine is booted up, we’re presented with the login form that we’re supposed to bypass using the two provided files; **usernames.txt** and **passwords.txt**.

For now, let’s attempt some logins and look at what happens.

{{< figure src="image_2.png" caption="Incorrect username error" align="center" >}}

Now we know that the site tells us if a user exists or not, we can use this later.

Let’s try some more usernames.

{{< figure src="image_3.png" caption="Login form with arithmetic captcha" align="center" >}}

After a couple of failed login attempts, the website attempts to rate limit us using an arithmetic captcha.

So let’s fill in the captcha and take a look at what’s happening behind the scenes.

{{< figure src="image_4.png" caption="POST Request to the login form" align="center" >}}

The POST request is pretty simple — we provide a **username**, **password** and the **captcha solution**.

After attempting more logins, I noticed that there is no pattern to the captcha at all, and there is no way around the rate limitation for now.

# Finding the username & evading the captcha

The way around the rate limitation is to use **Python** to send POST requests to the server, and to solve the captcha. To check which username(s) exist, we can simply check for any **response** that doesn’t contain “Error: *The user \<user\> does not exist*”.

Here’s a logic flow of what we’ll do:
- Define the endpoint for the POST request, and regular expressions to match the captcha equation and the user does not exist error
- Send incorrect credentials to the site to trigger the rate limiting
- Use the usernames file to send a POST request for each username
- Get the captcha equation using RegEx and solve it using `eval()`
- Resend the username with the captcha solution

If the RegEx for the invalid username does not match, we print the entire response, otherwise, we keep looping through the usernames file repeating each step.

{{< figure src="image_5.png" align="center" >}}

{{< figure src="image_6.png" caption="Response from supplying the correct username" align="center" >}}

As you can see, when we attempt to login with an existing username, we see something different in the response.

Now the error says “*Invalid password for user \<user\>*”.

# Finding the password

So, now that we have the username, we repeat the same process replacing the file with the `passwords.txt` file, and modifying our POST request.

Using the new error we got from the response, we can modify our RegEx and start brute-forcing the password.

{{< figure src="image_7.png" align="center" >}}

{{< figure src="image_8.png" caption="Final flag" align="center" >}}