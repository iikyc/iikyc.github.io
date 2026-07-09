+++
title = 'Hack The Box: Hack The Boo 2023 CTF Event — HauntMart Writeup'
date = '2023-10-28'
draft = false
tags = ["Hack The Box"]
summary = '"An eerie expedition into the world of online retail, where the most sinister and spine-tingling inventory reigns supreme. Can you take it down?"'
[cover]
image = 'main.png'
+++

This challenge is part of a series of challenges from [Hack The Boo 2023 Competition](https://ctf.hackthebox.com/event/details/hack-the-boo-2023-competition-1238) organized by Hack The Box.

{{< figure src="image_1.png" align="center" >}}

# Overview

**HauntMart** is the web category challenge revealed on the first day of the CTF, and involves:

- Code analysis
- Understanding of Docker containers
- Basic to intermediate web testing abilities

Provided as part of the challenge are:

- A dedicated Docker instance hosted by Hack The Box
- A ZIP archive containing the source code which also allows for building local instances of the website

{{< figure src="image_2.png" align="center" >}}

# Initial site review

{{< figure src="image_3.png" align="center" >}}

To access further pages on the site, a user needs to authenticate and we are given options to either login or create an account.

Initial checks to see if there are any data leaks through the **login form** are unsuccessful as the error messages simply state "*Invalid username/password!*", so enumerating for existing users is not possible.

However attempting to **sign up** with an existing username does generate an error informing us that the username is taken.

However, that is a dead end as no users exist.

# Creating an account and further site review

{{< figure src="image_4.png" align="center" >}}

As seen, the site is a Halloween-themed e-commerce style site, none of the products redirect anywhere however.. there is a second working link in the top navigation bar which redirects to `/product`.

{{< figure src="image_5.png" align="center" >}}

After submitting a product, we get a very vague message.

{{< figure src="image_6.png" align="center" >}}

So, it's time to take a look at the source code.

# Source code review

## Dockerfile

Locally, the website would be accessible through `http://localhost:1337/`.

```
# Expose port the server is reachable on
EXPOSE 1337
```

## main.py

- 2 possible URL prefixes: `/` and `/api`

{{< figure src="image_7.png" align="center" >}}

## routes.py

- 5 web routes
- 4 api routes

{{< figure src="image_8.png" align="center" >}}

Notice the decorators above each endpoint, we will get to these soon enough.

For now, the most interesting endpoint is `/api/addAdmin`, which takes in a `username` parameter to change the role of the user in the database to admin.

{{< figure src="image_9.png" align="center" >}}

## database.py

In database.py, we find the function `makeUserAdmin()`, which updates the role in the database.

{{< figure src="image_10.png" align="center" >}}

## util.py

- Contains definitions for the decorators seen in `routes.py`

The `isFromLocalhost()` function verifies that a request's remote address is `127.0.0.1`.

{{< figure src="image_11.png" align="center" >}}

# Exploit

So, we know which endpoint to call to "upgrade" our user to an admin role, however the challenge here is that the endpoint can only be called locally.

How do we that? We need to review the source code just a bit more.

On the website, submitting a product sends a POST request to the `/api/product` route, which takes in the **name, price, description and URL** of the product in JSON format.

{{< figure src="image_12.png" align="center" >}}

Taking a look at the API route, the program:

- Checks if the POST request has an authentication token via the `@isAuthenticated` decorator
- Stores the product data
- Checks if any product data is missing
- And finally, calls the `downloadManual()` function with the `manualUrl` from the product information which we inputted

{{< figure src="image_13.png" align="center" >}}

The `downloadManual()` function:

1. Checks if the URL is safe by passing it to the `isSafeUrl()` function
2. The `isSafeUrl()` function iterates through the list of blocked hosts `["127.0.0.1", "localhost", "0.0.0.0"]`, and if the product URL contains any of those strings, it deems it unsafe
3. Generates a local filename for the submitted manual from the URL. Eg. `http://myproduct.abc/productmanual` would be stored as `productmanual`
4. Sends a **GET** request to the provided product URL

{{< figure src="image_14.png" align="center" >}}

So, theoretically, we could exploit this to make the application send a GET request to `http://localhost:1337/api/addAdmin?username=<USERNAME>`.

However, we need to bypass the `isSafeUrl()` function check.

After a quick Google search for "*URL bypass*", I found [this page from HackTricks](https://book.hacktricks.xyz/pentesting-web/ssrf-server-side-request-forgery/url-format-bypass).

And just like that, all we have to do is:

- Ensure we have an account on the website
- Navigate to `/product` and add a product with the following "Manual Url": `http://127.000000000000000.1:1337/api/addAdmin?username=USERNAME`

Finally, the last thing we need to do is to logout and back in again in order to "update" our user's role on the running application.

{{< figure src="image_15.png" align="center" >}}

And just like that, we get the flag! (Fake flag generated from local instance)

{{< figure src="image_16.png" align="center" >}}