+++
title = 'TryHackMe — Juicy Details Writeup'
date = '2023-03-28'
draft = false
tags = []
summary = '“A popular juice shop has been breached! Analyze the logs to see what had happened…”'
[cover]
image = 'main.png'
+++

# Reconnaissance

**What tools did the attacker use?**

When taking a look at the `access.log` file, we can observe a lot of the tools the attacker used, which are: **nmap**, **hydra**, **sqlmap**, **curl** and **feroxbuster**.

{{< figure src="image_1.png" align="center" >}}

{{< figure src="image_2.png" align="center" >}}

{{< figure src="image_3.png" align="center" >}}

{{< figure src="image_4.png" align="center" >}}

{{< figure src="image_5.png" align="center" >}}

**What endpoint was vulnerable to a brute-force attack?**

Again, manually searching for Hydra attempts yields many GET and POST requests to `/rest/user/login`.

{{< figure src="image_6.png" align="center" >}}

**What endpoint was vulnerable to SQL injection?**

You can see the SQL injection payload being passed in the search query using **sqlmap**.

**/rest/products/search**

{{< figure src="image_7.png" align="center" >}}

**What parameter was used for the SQL injection?**

Taking a look at the `/rest/products/search` URL, the search parameter is `q`.

```
::ffff:192.168.10.5 - - [11/Apr/2021:09:29:15 +0000] "GET /rest/products/search?q=1&QKqc=7074%20AND%201%3D1%20UNION%20ALL%20SELECT%201%2CNULL%2C%27%3Cscript%3Ealert%28%22XSS%22%29%3C%2Fscript%3E%27%2Ctable_name%20FROM%20information_schema.tables%20WHERE%202%3E1--%2F%2A%2A%2F%3B%20EXEC%20xp_cmdshell%28%27cat%20..%2F..%2F..%2Fetc%2Fpasswd%27%29%23 HTTP/1.1" 200 - "-" "sqlmap/1.5.2#stable (http://sqlmap.org)"
```

**What endpoint did the attacker try to use to retrieve files? (Include the /)**

The attacker is trying to get two files from ``/ftp``.

As you can see in the logs, the request returns 403/Forbidden.

{{< figure src="image_8.png" align="center" >}}

# Stolen Data

**What section of the website did the attacker use to scrape user email addresses?**

**product reviews**

{{< figure src="image_9.png" align="center" >}}

**Was their brute-force attack successful? If so, what is the timestamp of the successful login?**

Look for the login attempt with Hydra that returns **200**.

**11/Apr/2021:09:16:31 +0000**

{{< figure src="image_10.png" align="center" >}}

**What user information was the attacker able to retrieve from the endpoint vulnerable to SQL injection?**

For this question, you will need to look at the SQL injections, and try to interpret what attributes are requested from the SQL tables.

**User’s emails:**

{{< figure src="image_11.png" align="center" >}}

And **user’s passwords**:

{{< figure src="image_12.png" align="center" >}}

**What files did they try to download from the vulnerable endpoint? (endpoint from the previous task, question #5)**

Looking at the `vsftpd.log` file, we see two OK DOWNLOAD events for two files.

**www-data.bak, coupons_2013.md.bak**

{{< figure src="image_13.png" align="center" >}}

**What service and account name were used to retrieve files from the previous question?**

The attacker exploited [CVE-1999–0497](https://nvd.nist.gov/vuln/detail/CVE-1999-0497).

**ftp, anonymous**

{{< figure src="image_14.png" align="center" >}}

**What service and username were used to gain shell access to the server?**

After many failed login attempts in the file `auth.log`, the attacker successfully logged in as the user **www-data** using ssh.

{{< figure src="image_15.png" align="center" >}}