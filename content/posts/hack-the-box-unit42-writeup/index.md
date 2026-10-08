+++
title = 'Hack The Box — Unit42 Writeup'
date = '2026-10-09'
draft = false
tags = ["DFIR", "Windows", "Hack The Box"]
summary = 'A Hack The Box DFIR Sherlock based on a campaign using trojanized UltraVNC, with Sysmon event logs provided.'
[cover]
image = ''
+++

# Processing the Logs

To start off, we will parse the Sysmon log file we have into a CSV using Eric Zimmerman's [EvtxECmd](https://github.com/EricZimmerman/evtx).

```powershell
EvtxECmd.exe -f .\Microsoft-Windows-Sysmon-Operational.evtx --csv .\ --csvf logs.csv
```

Once parsed, EvtxECmd shows us an overview of the file which contains 169 event logs.

{{< figure src="image_1.png" caption="" align="center" >}}

Finally, we will load the CSV into [Timeline Explorer](https://www.sans.org/tools/timeline-explorer).

# Question 1

> How many Event logs are there with Event ID 11?

Although this can be answered by looking at the stats EvtxECmd prints, we can also find the answer by dragging the `Event Id` column to the top in Timeline Explorer, which groups the events by their event ID, showing that we have **56** records with event ID 11.

{{< figure src="image_2.png" caption="" align="center" >}}

# Question 2

> Whenever a process is created in memory, an event with Event ID 1 is recorded with details such as command line, hashes, process path, parent process path, etc. This information is very useful for an analyst because it allows us to see all programs executed on a system, which means we can spot any malicious processes being executed. What is the malicious process that infected the victim's system?

Grouping the events further by the `Executable Info` field, we see one event which stands out - an executable with a double-extension in the user's downloads folder, mentioning `C:\Users\CyberJunkie\Downloads\Preventivo24.02.14.exe.exe`.

{{< figure src="image_3.png" caption="" align="center" >}}

# Question 3

> Which Cloud drive was used to distribute the malware?

Looking at records with event ID 22 (DNS queries), we see DNS queries for **Dropbox**.

{{< figure src="image_4.png" caption="" align="center" >}}

# Question 4

> For many of the files it wrote to disk, the initial malicious file used a defense evasion technique called Time Stomping, where the file creation date is changed to make it appear older and blend in with other files. What was the timestamp changed to for the PDF file?

Looking at records with event ID 2 (file creation time change), we can find the new timestamp of the PDF file: `2024-01-14 08:10:06`.

{{< figure src="image_5.png" caption="" align="center" >}}

{{< figure src="image_6.png" caption="" align="center" >}}

# Question 5

> The malicious file dropped a few files on disk. Where was "once.cmd" created on disk? Please answer with the full path along with the filename.

Turning to records with event ID 11 (file create operations), we find the path of this file: `C:\Users\CyberJunkie\AppData\Roaming\Photo and Fax Vn\Photo and vn 1.1.2\install\F97891C\WindowsVolume\Games\once.cmd`.

{{< figure src="image_7.png" caption="" align="center" >}}

# Question 6

> The malicious file attempted to reach a dummy domain, most likely to check the internet connection status. What domain name did it try to connect to?

As with question 3, we need to look at records with event ID 22, where there is another DNS query for `www[.]example[.]com`.

{{< figure src="image_8.png" caption="" align="center" >}}

# Question 7

> Which IP address did the malicious process try to reach out to?

Records with event ID 3 (network connect) point to `93[.]184[.]216[.]34`.

{{< figure src="image_9.png" caption="" align="center" >}}

# Question 8

> The malicious process terminated itself after infecting the PC with a backdoored variant of UltraVNC. When did the process terminate itself?

The process was terminated on `2024-02-14 03:41:58`, as told by looking at event IDs 5 (process terminated.)

{{< figure src="image_10.png" caption="" align="center" >}}

# Further Reading

- [Unit42-timely-threat-intel](https://github.com/PaloAltoNetworks/Unit42-timely-threat-intel/blob/main/2024-01-23-IOCs-from-UltraVNC-infection.txt)