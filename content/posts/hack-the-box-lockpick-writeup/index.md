+++
title = 'Hack The Box — Lockpick Writeup'
date = '2025-05-04'
draft = false
tags = ["Hack The Box", "Linux", "Reversing", "Malware"]
summary = '"Forela needs your help! A whole portion of our UNIX servers have been hit with what we think is ransomware. We are refusing to pay the attackers and need you to find a way to recover the files provided."'
[cover]
image = 'main.png'
+++

# Disclaimer

>This is a warning that this Sherlock includes software that is going to interact with your computer and files. This software has been intentionally included for educational purposes and is NOT intended to be executed or used otherwise. Always handle such files in isolated, controlled, and secure environments. Once the Sherlock zip has been unzipped, you will find a DANGER.txt file. Please read this to proceed.

First off, we're given a zip file containing 3 files:
- `DANGER.txt` - Which contains important information and the password needed to unzip the malware archive
- `bescrypt.zip` - The archive containing the malware sample
- `forela-criticaldata` - Containing the locked files

{{< figure src="image_1.png" align="center" >}}

The first thing we need to do in order to answer the questions, get the encryption key and data from the locked files, is to load the malware sample (`bescrypt3.2`) into **IDA** to find the encryption key and understand how the files were encrypted.

{{< figure src="image_2.png" caption="IDA Text view - Main function" align="center" >}}

The main function of the sample appears simple, calling the function `process_directory()` with the target directory path and encryption key.

This is reflected more simply in IDA's pseudocode:

{{< figure src="image_3.png" caption="IDA pseudocode - Main function" align="center" >}}

# Task 1

With this information, we can answer task 1:

>Please confirm the encryption key string utilised for the encryption of the files provided?
**Answer:** bhUlIshutrea98liOp

Now that we've found the encryption key and target directory (`bhUlIshutrea98liOp`, `/forela-criticaldata/`), let's dig into the `process_directory()` function.

{{< figure src="image_4.png" caption="IDA pseudocode - process_directory() function" align="center" >}}

The sample appears to follow this logic:
- Opens `/forela-criticaldata/`
- Reads each entry in the directory (returned as a **dirent** struct from `readdir()`)
- If the `d_type` member is `4` (`DT_DIR`, a directory), it's passed to `process_directory()` again
- If the `d_type` member is `8` (`DT_REG`, a regular file), and its extension is one of the listed in the figure above, it's passed to `encrypt_file()`

For reference, here is the dirent struct:

{{< figure src="image_5.png" align="center" >}}

Next, we'll take a look at the `encrypt_file()` function:

{{< figure src="image_6.png" caption="IDA pseudocode - encrypt_file() function" align="center" >}}

{{< figure src="image_7.png" caption="IDA pseudocode - encrypt_file() function" align="center" >}}

{{< figure src="image_8.png" caption="IDA pseudocode - encrypt_file() function" align="center" >}}

{{< figure src="image_9.png" caption="IDA pseudocode - encrypt_file() function" align="center" >}}

The function `encrypt_file()`:
- Opens the file in binary mode, determines its size and reads its content into memory
- XOR encrypts the file with the key
- Writes the encrypted content into `<original_filename>.24bes`
- Writes a ransom note for each encrypted file as `<original_filename>.24bes_ note.txt`
- Deletes the original (unencrypted) file

# Decrypting the files

The following Python script decrypts files with the extension `.24bes` in `/forela-criticaldata/`:

```python
import os

# Configuration
KEY = b"bhUlIshutrea98liOp"
INPUT_DIR = "./forela-criticaldata/"
OUTPUT_DIR = os.path.join(INPUT_DIR, "decrypted")

# Ensure output directory exists
os.makedirs(OUTPUT_DIR, exist_ok=True)

def xor_decrypt(data: bytes, key: bytes) -> bytes:
    key_len = len(key)
    return bytes([b ^ key[i % key_len] for i, b in enumerate(data)])

# Decrypt files with the .24bes extension only
for filename in os.listdir(INPUT_DIR):
    if filename.endswith(".24bes"):
        input_path = os.path.join(INPUT_DIR, filename)
        output_path = os.path.join(OUTPUT_DIR, filename.replace(".24bes", ""))  # remove extension

        with open(input_path, "rb") as f:
            encrypted_data = f.read()

        decrypted_data = xor_decrypt(encrypted_data, KEY)

        with open(output_path, "wb") as f:
            f.write(decrypted_data)

        print(f"Decrypted: {filename} -> {output_path}")
```

# Task 2

>We have recently recieved an email from wbevansn1@cocolog-nifty[.]com demanding to know the first and last name we have him registered as. They believe they made a mistake in the application process. Please confirm the first and last name of this applicant. 
**Answer:** Walden Bevans

```bash
$ grep -r 'wbevansn1@cocolog-nifty.com'
forela_uk_applicants.sql:(830,'Walden','Bevans','wbevansn1@cocolog-nifty.com','Male','Aerospace Manufacturing','2023-02-16'),
```

# Task 3

>What is the MAC address and serial number of the laptop assigned to Hart Manifould? 
**Answer:** E8-16-DF-E7-52-48, 1316262

{{< figure src="image_10.png" caption="it_assets.xml" align="center" >}}

# Task 4

>What is the email address of the attacker? 
**Answer:** bes24@protonmail[.]com (refang to answer)

We've already found this when looking at the function `encrypt_file()` in IDA, but it can also be found in the ransom notes in `/forela-criticaldata/`:

{{< figure src="image_11.png" align="center" >}}

# Task 5

>City of London Police have suspiciouns of some insider trading taking part within our trading organisation. Please confirm the email address of the person with the highest profit percentage in a single trade alongside the profit percentage. 
**Answer:** fmosedale17a@bizjournals[.]com, 142303.1996053929628411706675436 (refang to answer)

# Task 6

>Our E-Discovery team would like to confirm the IP address detailed in the Sales Forecast log for a user who is suspected of sharing their account with a colleague. Please confirm the IP address for Karylin O'Hederscoll. 
**Answer:** 8[.]254[.]104[.]208 (refang to answer)

{{< figure src="image_12.png" caption="sales_forecast.xlsx" align="center" >}}

# Task 7

>Which of the following file extensions is not targeted by the malware? .txt, .sql,.ppt, .pdf, .docx, .xlsx, .csv, .json, .xml 
**Answer:** .ppt

We've also found this previously when looking at the function `process_directory()` in IDA.

# Task 8

>We need to confirm the integrity of the files once decrypted. Please confirm the MD5 hash of the applicants DB. 
**Answer:** f3894af4f1ffa42b3a379dddba384405

```bash
$ md5sum forela_uk_applicants.sql                
f3894af4f1ffa42b3a379dddba384405  forela_uk_applicants.sql
```

# Task 9

>We need to confirm the integrity of the files once decrypted. Please confirm the MD5 hash of the trading backup. 
**Answer:** 87baa3a12068c471c3320b7f41235669

```bash
$ md5sum trading-firebase_bkup.json
87baa3a12068c471c3320b7f41235669  trading-firebase_bkup.json
```

# Task 10

>We need to confirm the integrity of the files once decrypted. Please confirm the MD5 hash of the complaints file. 
**Answer:** c3f05980d9bd945446f8a21bafdbf4e7

```bash
$ md5sum complaints.csv                
c3f05980d9bd945446f8a21bafdbf4e7  complaints.csv
```