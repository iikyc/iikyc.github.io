+++
title = 'Hack The Box — Lockpick 2.0 Writeup'
date = '2025-05-17'
draft = false
tags = ["Hack The Box", "Linux", "Reversing", "Malware"]
summary = '''
"We've been hit by Ransomware again, but this time the threat actor seems to have upped their skillset. Once again a they've managed to encrypt a large set of our files. It is our policy NOT to negotiate with criminals. Please recover the files they have encrypted - we have no other option! Unfortunately our CEO is on a no-tech retreat and so can't be reached."
'''
[cover]
image = 'main.png'
+++

# Disclaimer

>This is a warning that this Sherlock includes software that is going to interact with your computer and files. This software has been intentionally included for educational purposes and is NOT intended to be executed or used otherwise. Always handle such files in isolated, controlled, and secure environments. Once the Sherlock zip has been unzipped, you will find a DANGER.txt file. Please read this to proceed.

Firstly, we're given a zip file containing 2 files and 1 folder:
- `DANGER.txt` - Which contains important information and the password needed to unzip the malware archive
- `share/` - Containing the encrypted files
- `malware` - The archive containing the malware sample

{{< figure src="image_1.png" align="center" >}}

To start off, we'll perform initial triage on the malware sample (`update`):
- **Type:** ELF64
- **Size:** 9.32 KiB
- **SHA1:** f5194fff7081b9d25eb4acebbb46f752e562db4c

Additionally, we note that loading the sample into **DiE (Detect It Easy)** reveals that it is packed using **UPX**:

{{< figure src="image_2.png" caption="DiE" align="center" >}}

# Task 8

Knowing this, we can answer task 8:

>What was used to pack the malware? 
**Answer:** upx

# Unpacking the sample

To unpack the sample, we need to download [UPX](https://upx.github.io/) and run:

```powershell
.\upx.exe -d <PATH>\lockpick2\lockpick2.0\malware\malware\update -o update_unpacked
```

# Analyzing the sample

Now that we have the unpacked sample, let's load it into **IDA** and take a look:

{{< figure src="image_3.png" caption="main function" align="center" >}}

The main function in summary:
- Calls `get_key_from_url()` to obtain a key from a URL
- Immediately quits in case this fails
- Stores the key in `file_encryption_key`
- Calls `handle_directory()` to process the directory `/share/`
- Runs junk code to fake an update

{{< figure src="image_4.png" caption="main function - junk code" align="center" >}}

Let's look at the `get_key_from_url()` function next:

{{< figure src="image_5.png" caption="get_key_from_url function" align="center" >}}

Firstly, it sets up curl to eventually obtain the file encryption key from the URL, which in this case is XOR-encrypted using the key stored in `HESB`.

The hardcoded XOR-key `HESB` is `b7894532snsmajuys6`:

{{< figure src="image_6.png" caption=".data section" align="center" >}}

As the `get_key_from_url()` function is now clear, and we have the XOR-key, we can decrypt the URL:

```
xor_encrypted_key_url: 0A 43 4C 49 47 0F 1C 1D 01 0C 5D 0A 18 45 46 1F 1F 45 1B
```

{{< figure src="image_7.png" align="center" >}}

Visiting the URL gives us another file:
- **Name:** updater
- **MD5:** 950efb05238d9893366a816e6609500f
- **Type:** Binary data

{{< figure src="image_8.png" caption="HxD - updater" align="center" >}}

# Tasks 4 and 5

As we now have the file encryption key, we can answer tasks 4 and 5:

>What is the file name of the key utlised by the attacker? 
**Answer:** updater

>What is the file hash of the key utilised by the attacker? 
**Answer:** 950efb05238d9893366a816e6609500f

We can save the contents of `updater`, and take a look at the `handle_directory()` function:

{{< figure src="image_9.png" caption="handle_directory function" align="center" >}}

This function firstly creates a ransom note at `/share/countdown.txt`, then calls `download_lyrics()` to download and write the ransom note content to it:

{{< figure src="image_10.png" caption="download_lyrics function" align="center" >}}

It then continues to read directory entries, checking if each file's extension is one of the targeted ones by calling `is_target_extension()`:

{{< figure src="image_11.png" caption="is_target_extension function" align="center" >}}

And then finally calling the file encryption function which reads the original file's content, creates a placeholder `<original_filename>.24bes` file, encrypts the content using **AES** and writes it to the placeholder file.

Finally ending with deleting the original file:

{{< figure src="image_12.png" caption="encrypt_file function" align="center" >}}

# Task 1

With this information, we can answer task 1:

>What type of encryption has been utilised to encrypt the files provided? 
**Answer:** AES

# Decrypting the files

With the sample functionality covered, we know:
- It uses XOR-encryption for hardcoded strings
- Files are encrypted with AES
- We have the AES key, downloaded as the file `updater`

We can proceed with decrypting the files now:

```python
from Crypto.Cipher import AES

def decrypt_file(enc_path, dec_path, key, iv):
    with open(enc_path, 'rb') as f_in:
        ciphertext = f_in.read()

    cipher = AES.new(key, AES.MODE_CBC, iv)
    plaintext = cipher.decrypt(ciphertext)

    pad_len = plaintext[-1]
    if all(p == pad_len for p in plaintext[-pad_len:]):
        plaintext = plaintext[:-pad_len]
    
    with open(dec_path, 'wb') as f_out:
        f_out.write(plaintext)

# Hex values from updater
key = bytes.fromhex("f3fc056da1185eae370d76d47c4cf9db9f4efd1c1585cde3a7bcc6cb5889f6db")
iv  = bytes.fromhex("01448c7909939e13ce359710b9f0dc2e")

decrypt_file("takeover.docx.24bes", "takeover.docx", key, iv)
decrypt_file("expanding-horizons.pdf.24bes", "expanding-horizons.pdf", key, iv)
```

# Task 2

>Which market is our CEO planning on expanding into? (Please answer with the wording utilised in the PDF) 
**Answer:** Australian Market

{{< figure src="image_13.png" caption="expanding-horizons.pdf" align="center" >}}

# Task 3

>Please confirm the name of the bank our CEO would like to takeover? 
**Answer:** Notionwide

{{< figure src="image_14.png" caption="takeover.docx" align="center" >}}

# Task 6

>What is the BTC wallet address the TA is asking for payment to? 
**Answer:** 1BvBMSEYstWetqTFn5Au4m4GFg7xJaNVN2

{{< figure src="image_15.png" caption="countdown.txt" align="center" >}}

# Task 7

>How much is the TA asking for? 
**Answer:** £1000000

{{< figure src="image_16.png" caption="countdown.txt" align="center" >}}