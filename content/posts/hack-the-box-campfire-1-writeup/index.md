+++
title = 'Hack The Box — Campfire-1 Writeup'
date = '2026-09-13'
draft = false
tags = ["Hack The Box", "DFIR", "Windows", "Active Directory"]
summary = 'A Hack The Box DFIR Sherlock involving event logs and prefetch files from a domain controller and workstation.'
[cover]
+++

To start off, we are provided with:
- **Security logs** from the domain controller
- **PowerShell logs** from the affected workstation
- **Prefetch files** from the affected workstation

```
└───Triage
    ├───Domain Controller
    │       SECURITY-DC.evtx
    │
    └───Workstation
        │   Powershell-Operational.evtx
        │
        └───2024-05-21T033012_triage_asset
            └───C
                └───Windows
                    └───prefetch
                            ...
```

# Question 1

>Analyzing Domain Controller Security Logs, can you confirm the UTC date & time when the kerberoasting activity occurred?

The first thing we need to do is check the security logs from the domain controller, this can be done by parsing the event log file with Eric Zimmerman's EvtxECmd then loading the resulting CSV into Timeline Explorer.

```powershell
.\EvtxECmd.exe -f '.\campfire-1\Triage\Domain Controller\SECURITY-DC.evtx' --csv <OUT_DIR> --csvf <OUT_FILE>
```

The event ID we are interested in is **4769** (a Kerberos service ticket was requested). Filtering to events with the 4769 event ID, we get 16 events, however the one that we need to look at is **event record ID 6672**.

Under service information, we see the service name `MSSQLService`, with the encryption type `RC4-HMAC` (corresponding to `0x17` when viewed in Event Viewer).

{{< figure src="image_1.png" caption="" align="center" >}}

Answer: `2024-05-21 03:18:09`.

# Question 2

>What is the Service Name that was targeted?

Identified from question 1.

Answer: `MSSQLService`.

# Question 3

>It is really important to identify the Workstation from which this activity occurred. What is the IP Address of the workstation?

Identified from question 1.

Answer: `172.17.79.129`.

# Question 4

>Now that we have identified the workstation, a triage including PowerShell logs and Prefetch files are provided to you for some deeper insights so we can understand how this activity occurred on the endpoint. What is the name of the file used to Enumerate Active directory objects and possibly find Kerberoastable accounts in the network?

We will parse the workstation's PowerShell log file with EvtxECmd once again, and load it into Timeline Explorer.

```powershell
.\EvtxECmd.exe -f '.\campfire-1\Triage\Workstation\Powershell-Operational.evtx' --csv <OUT_DIR> --csvf <OUT_FILE>
```

Several events can be seen referencing a path to a PowerView PowerShell script.

{{< figure src="image_2.png" caption="" align="center" >}}

Answer: `powerview.ps1`.

# Question 5

>When was this script executed? (UTC)

Identified from question 4.

{{< figure src="image_3.png" caption="" align="center" >}}

Answer: `2024-05-21 03:16:32`.

# Question 6

>What is the full path of the tool used to perform the actual kerberoasting attack?

We will parse the workstation's prefetch files with PECmd.  

```powershell
.\PECmd.exe -d .\campfire-1\Triage\Workstation\2024-05-21T033012_triage_asset\C\Windows\prefetch\ -q --csv <OUT_DIR> --csvf <OUT_FILE>
```

Loading the resulting CSV into Timeline Explorer, we identify a prefetch file referencing a Rubeus executable.

{{< figure src="image_4.png" caption="" align="center" >}}

Checking the `Files Loaded` column we can find the full path to the executable.

{{< figure src="image_5.png" caption="" align="center" >}}

Answer: `C:\Users\Alonzo.spire\Downloads\Rubeus.exe`.

# Question 7

>When was the tool executed to dump credentials? (UTC)

The `Last Run` column from the prefetch file provides us with the timestamp.

{{< figure src="image_6.png" caption="" align="center" >}}

Answer: `2024-05-21 03:18:08`.