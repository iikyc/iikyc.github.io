+++
title = 'Hack The Box Cyber Apocalypse 2024'
date = '2024-03-14'
draft = false
tags = ["Hack The Box", "DFIR"]
summary = '"Legionaries in an apocalypse"'
[cover]
image = 'main.png'
+++

The following challenges are a part of [Hack The Box's Cyber Apocalypse 2024](https://ctf.hackthebox.com/event/details/cyber-apocalypse-2024-hacker-royale-1386) CTF.

I was able to solve 20 out of the 67 challenges across 8 categories. In this post, I'll be covering 2 forensics challenges and 2 web challenges.

{{< figure src="image_1.png" align="center" >}}

# Challenge 1: Phreaky

The first I'll go through is "**Phreaky**", which is part of the forensics category.

We are provided with a **PCAP file**, and after scrolling through a bit to see what stands out, I noticed some SMTP packets.

{{< figure src="image_2.png" align="center" >}}

If we right click on the packet and follow the TCP stream, we can view the contents of the packet, which include **base64** encoded data along with a password.

If you look closely, the encoded data is an archive (zip file), as hinted by the `Content-Type` header.

{{< figure src="image_3.png" align="center" >}}

We can use CyberChef to decode the data and extract the zip archive from it.

{{< figure src="image_4.png" align="center" >}}

As you can see in the output tab, and as hinted by the SMTP packets, the complete PDF is split up and sent separately.

We can find the rest of the packets containing the file's data by using the `frame contains "part of the file"` filter in Wireshark. This will give us any packets containing the string `part of the file`.

{{< figure src="image_5.png" align="center" >}}

So now we know that we need to decode and extract the zip archives from each packet containing each part of the final PDF file.

The next step is to open up each of the packets -> follow TCP stream and record the base64 encoded data along with their respective password.

{{< figure src="image_6.png" align="center" >}}

After that, we can put each part of the encoded data in CyberChef, download the zip file and extract the contents using the password. This will give us a total of **15** "part" PDFs.

Taking a look at the first part of the final PDF, we can see that its an incomplete file.

{{< figure src="image_7.png" align="center" >}}

You can't open any of these incomplete files, so what we need to do is to concatenate all of these parts into a single PDF file.

This can be achieved purely through a terminal.

{{< figure src="image_8.png" align="center" >}}

The final PDF contains the flag on the second page.

{{< figure src="image_9.png" align="center" >}}

# Challenge 2: Pursue the Tracks

The following challenge is also part of the forensics series, called "**Pursue the Tracks**".

We are provided with an **MFT file (NTFS Master File Table)**, and a Docker instance which we can connect to using netcat in order to answer questions and get the flag.

{{< figure src="image_10.png" align="center" >}}

To open the MFT file, you can use [Eric Zimmerman's MFTExplorer](https://ericzimmerman.github.io/#!index.md), or [MFT Browser](http://kacos2000.github.io/MFT_Browser/) - which I used.

{{< figure src="image_11.png" align="center" >}}

We can answer the first 2 questions just from looking at the entries.

>Q1. Files are related to two years, which are those? **2023,2024**

>Q2. There are some documents, which is the name of the first document? **Final_Annual_ Report.xlsx**

Next, we're asked to find out which file was deleted.

Looking at the headers for `Marketing_Plan.xlsx`, the `Allocation Status` header is set to `0x0000` (deleted/deallocated).

{{< figure src="image_12.png" align="center" >}}

The next question asks how many files were set in hidden mode, looking at the `Attributes` for `credentials.txt`, the `Flag bit` is set to `Hidden`.

{{< figure src="image_13.png" align="center" >}}

**Next question:** What is the filename of the important TXT file: **credentials.txt**.

>Q6. A file was also copied, which is the new filename? **Financial_Statement_ draft.xlsx** (frankly, I guessed this one)

>Q7. Which file was modified after creation? **Project_Proposal.pdf**

{{< figure src="image_14.png" align="center" >}}

>Q8. What is the name of the file located at record number 45? **Annual_Report.xlsx**

{{< figure src="image_15.png" align="center" >}}

>Q9. What is the size of the file located at record number 40? **57344**

{{< figure src="image_16.png" align="center" >}}

# Challenge 3: TimeKORP

Next we have "**TimeKORP**", a web challenge.

We're provided with a website and the source code.

{{< figure src="image_17.png" align="center" >}}

The website is used to display the current time or date. Notice the `format` parameter? It specifies the display format of the time or date.

Looking at the source code, its written in **PHP** and passes the value of the `format` parameter without sanitization

The user input is directly concatenated to the command `date '+<parameter_value>' 2>&1`.

{{< figure src="image_18.png" align="center" >}}

We can verify this by running `date '+%H:%M:%S' 2>&1` in a terminal.

{{< figure src="image_19.png" align="center" >}}

How do we break this?

We can close the single quote after the `date` command, and insert our own command.

As a proof of concept, we can run the following two commands within a terminal (highlighted part would be the `format` parameter value).

{{< figure src="image_20.png" align="center" >}}

Taking this to the web application, I printed the **current user**, **directory** and a **recursive listing** of the current directory.

{{< figure src="image_21.png" align="center" >}}

As you can see, the flag is nowhere in the current directory (nor in the parent directory).

Firstly, I managed to find the flag by running two commands separately, but I summarized it below in one command.

We can use the `find` command to find any file containing the string `flag` in its name, store it in a variable and then `cat` the contents of the file.

{{< figure src="image_22.png" align="center" >}}

# Challenge 4: KORP Terminal

The final challenge I'll be covering is "**KORP Terminal**", a web challenge.

We're only provided with a web application which has a login form.

{{< figure src="image_23.png" align="center" >}}

Playing around with the username and password parameters, I found out that passing a `'` into the **username field** results in an interesting database error.

{{< figure src="image_24.png" align="center" >}}

From the error, we can infer that in the backend is utilizing a **MariaDB** database, and that the username field is vulnerable to SQLi.

So we can use `sqlmap` to try and dump the tables. We let sqlmap try random values in the username field, and it results in dumping the table `users` from the database `korp_terminal` by error-based/time-based SQLi.

{{< figure src="image_25.png" align="center" >}}

There is a single entry in the users table containing the username `admin` and the password which is a **bcrypt hash** (as denoted by the `$2b`).

{{< figure src="image_26.png" align="center" >}}

To crack the hash, we can use [John](https://www.kali.org/tools/john/) and the [rockyou.txt](https://www.kali.org/tools/wordlists/) wordlist.

We get the flag after logging in as the user `admin`.

{{< figure src="image_27.png" align="center" >}}