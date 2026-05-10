# WingData (HackTheBox) — Full Walkthrough

## Overview

* **Difficulty:** Easy
* **Platform:** Hack The Box
* **Focus Areas:** Web Enumeration, Subdomain Discovery, RCE, Credential Cracking, Privilege Escalation, Python CVE Exploitation

This writeup walks through my full approach to compromising the WingData machine. The box revolves around multiple misconfigurations and vulnerabilities across different services, including a WingFTP server, weakly protected credentials, and a Python tarfile path traversal vulnerability. By combining thorough enumeration, hash cracking, and a CVE-based exploit, I was able to escalate from initial web access to full root compromise.

---

# Enumeration & Reconnaissance

## Initial Network Scanning

I started with Nmap to map the target’s services:

![Nmap Output](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Nmap%20Output.png)

### Findings

* SSH (port 22)
* HTTP (port 80)

To make further testing easier, I added the target to my `/etc/hosts`:

![Adding Host](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Adding%20Host.png)

---

# Virtual Host Discovery

Visiting `http://wingdata.htb` initially appeared to be a standard webpage. However, inspecting the page source revealed a reference to a subdomain.

![Page Sources](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Page%20Sources.png)

Adding this to `/etc/hosts` allowed access:

![Adding Vhost](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Adding%20Vhost.png)

Navigating to `http://ftp.wingdata.htb` revealed a WingFTP Server login page with a version number clearly displayed. This version information became crucial for the next step.

![WingFTP Login Page](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/WingFTP%20Login%20Page.png)

---

# Initial Access: WingFTP RCE Exploitation

## Vulnerability Research

Googling the WingFTP version revealed a Remote Code Execution (RCE) vulnerability that allows unauthenticated command execution.

![WingFTP Version Exploit](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/WingFTP%20Version%20Exploit.png)

---

## Exploitation with Metasploit

Using the Metasploit Framework, I loaded the appropriate exploit module:

![Metasploit Search Output](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Metasploit%20Search%20Output.png)

```bash
set RHOSTS ftp.wingdata.htb
set LHOST tun0
set LPORT 4444
run
```

![Spawned Shell](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Spawned%20Shell.png)

This yielded a limited shell. The current user had no access to the user flag, indicating the need for lateral movement.

---

# Lateral Movement: Credential Cracking

## User Enumeration

Inspecting `/etc/passwd` revealed another user:

![User Enumeration](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/User%20Enumeration.png)

The goal became compromising this account.

---

## Configuration File Enumeration

Further exploration revealed configuration files containing password hashes.

![User Configuration Files](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/User%20Configuration%20Files.png)

![User Password Hash](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/User%20Password%20Hash.png)

A particularly valuable file, `settings.xml`, documented that the hashes were:

* SHA256
* Salted with `"WingFTP"`

![SHA256+Salt](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/SHA256%2BSalt.png)

---

# Hash Cracking with Hashcat

I compiled all hashes with their salt into a file formatted for Hashcat:

![All Hashes with Salts](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/All%20Hashes%20with%20Salts.png)

Then ran Hashcat:

```bash
hashcat -m 1420 -a 0 hashes.txt rockyou.txt
```

The password for the `wacky` user was successfully cracked.

![Hashcat Output](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Hashcat%20Output.png)

---

# Gaining User Access

With valid credentials, I SSH’d into the machine, this provided access to the user flag.

![User Flag](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/User%20Flag.png)

```text
User Flag: 228c965be2452ae6a28ddfc1a8f769fb
```

---

# Privilege Escalation: CVE-2025-4517

## Sudo Enumeration

I ran `linpeas.sh` as `wacky`, which revealed a sudo privilege.

![LinPEAS Output](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/LinPEAS%20Output.png)

The combination of wildcard arguments and `NOPASSWD` presented a potential vulnerability vector.

---

# Vulnerable Script Analysis

Examining `/opt/backup_clients/restore_backup_clients.py`, I found the critical section responsible for archive extraction.

![Python File Code Snippet](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Python%20File%20Code%20Snippet.png)

### Observations

* `backup_path` is user-controlled.
* The script extracts archives as root.
* `filter="data"` is intended to block dangerous paths.
* `/opt/backup_clients/backups/` is writable by `wacky`.

---

# CVE-2025-4517: Python Tarfile Path Traversal

This vulnerability affects Python versions:

* 3.8.0 – 3.13.1
* Target version: Python 3.12.3

### Root Cause

The `filter` parameter is supposed to prevent symlinks and path traversal. However:

* If a path exceeds `PATH_MAX` (4096 bytes),
* `os.path.realpath()` fails to fully resolve symlinks,
* Returning a “safe-looking” path.

This allows arbitrary filesystem writes outside the intended extraction directory.

---

# Exploitation

I leveraged a publicly available exploit for CVE-2025-4517 targeting Python’s `tarfile` module. The exploit abuses the `filter="data"` bypass in Python 3.12.3 to achieve path traversal.

Using the script, I executed the vulnerable backup restoration script as root, which allowed me to gain root access and retrieve the root flag.

![Root Flag](https://github.com/virtualISP/Writeups/blob/main/HackTheBox/WingData/images/Root%20Flag.png)

```text
Root Flag: 2b988934c03b0d28fe930af2fe74cdae
```

---

# Key Takeaways

* **Enumeration:** Always inspect page sources for subdomains; virtual host discovery is often overlooked.
* **Credential Security:** Never hardcode salts in configuration files or use weak salts like `"WingFTP"`.
* **Sudo Misconfigurations:** Wildcard arguments with `NOPASSWD` can be extremely dangerous, especially when combined with user-writable directories.
* **Library Vulnerabilities:** Stay updated on CVEs affecting core libraries. The `filter` parameter in Python’s `tarfile` module is not a silver bullet.
* **Python Version Patching:** Upgrade to Python 3.13.2+ to fully mitigate path traversal vulnerabilities in tarfile extraction.

---

# Conclusion

This machine demonstrates how multiple minor vulnerabilities and misconfigurations can be chained together into a full compromise:

1. Web and virtual host enumeration revealing a vulnerable WingFTP server
2. Remote Code Execution on WingFTP to gain initial access
3. Discovery of user hashes in configuration files
4. Cracking SHA256 salted hashes to move laterally to another user
5. Writable directories combined with wildcard sudo privileges
6. Exploitation of Python tarfile path traversal (CVE-2025-4517) for root access

By carefully enumerating each service, analyzing exposed credentials, and leveraging known library vulnerabilities, it was possible to escalate from limited web access to full root control of the machine.
