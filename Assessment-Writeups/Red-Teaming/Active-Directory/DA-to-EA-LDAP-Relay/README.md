
# From Domain Admin to Enterprise Admin: Exploiting Weak LDAP Signing via NTLM Relay

> **Note:** To protect the confidentiality and integrity of the actual penetration test, this write-up uses sanitized values. The original domain name, Domain Controller IP address, and credentials have been replaced with assumed/example values.

## Introduction

Active Directory privilege escalation is rarely caused by a single vulnerability. In real-world environments, security issues often appear when multiple legitimate technologies interact with insecure configurations.

Authentication protocols, directory services, permissions, and delegation mechanisms are designed to work together. However, when one security control is weakened, trusted functionality can become an unintended attack path.

The assessment began with authorized credentials belonging to a member of the **Domain Admins** group in the `domain.test` domain. This provided a highly privileged domain-level administrative context from the beginning of the assessment.

The assessment focused on understanding the impact of an improperly configured LDAP signing policy.

The goal was not only to achieve higher privileges but to understand:

- Why LDAP signing exists
- How missing LDAP integrity protection enables NTLM relay attacks
- How Active Directory permissions can transform a domain-level compromise into forest-level impact
- Which defensive controls can prevent this attack path


## Active Directory Authentication: The Required Foundation

Before discussing LDAP signing and NTLM relay, it is important to understand how authentication works in a Windows domain environment.

Active Directory relies primarily on authentication protocols such as:

- **NTLM**
- **Kerberos**

These protocols are responsible for proving the identity of users and systems inside a Windows environment.

A detailed breakdown of Windows authentication protocols, including NTLM and Kerberos internals, is covered in my previous article:

> **[Windows Authentication Protocols Simplified!](https://medium.com/@zer0vuln/windows-active-directory-authentication-protocols-c98f568a15cc)**


For this assessment, we only need to understand the security concepts behind these protocols.



## Assessment Scope and Initial Access Context

The assessment began with authorized credentials belonging to a member of the **Domain Admins** group in the `domain.test` domain.

A member of the Domain Admins group has extensive administrative authority within its domain and typically has control over:

- User accounts
- Computer accounts
- Group memberships
- Group Policy Objects
- Domain-level security settings

However, Active Directory environments are organized into a larger security boundary known as an **Active Directory forest**.

### Domain and Forest Relationship

A member of the **Domain Admins** group has extensive administrative authority within its domain, whereas membership in the **Enterprise Admins** group provides administrative authority across the Active Directory forest.

![Domain-Forest Relationship](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/Domain-Forest_Relationship.png)

This distinction became important during the assessment because the attack path demonstrated how an already privileged domain-level identity could be used to obtain forest-level administrative privileges.

---

## Initial Active Directory Configuration Review

After validating the provided access, the next phase focused on reviewing the Active Directory configuration and identifying potential security weaknesses.

During this review, one configuration stood out:

> **LDAP signing was not being enforced on the Domain Controller.**

![Registry Lookup](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/Registry-Lookup.png)

At first glance, this may appear to be a minor hardening issue. However, when combined with NTLM authentication and an appropriately privileged account, insufficient LDAP signing can create an avenue for NTLM relay attacks.

To understand why, it is first necessary to understand the role of LDAP and LDAP signing.

---

## LDAP and Active Directory

**LDAP (Lightweight Directory Access Protocol)** is used by applications and systems to communicate with Active Directory Domain Services.

It allows clients to perform operations such as:

- Querying directory objects
- Reading object attributes
- Modifying directory objects
- Managing users and groups
- Changing directory permissions

Because LDAP provides access to sensitive identity and directory information, protecting LDAP communication is an important part of Active Directory security.

---

## LDAP Signing

LDAP signing provides integrity protection for LDAP communication between clients and Domain Controllers.

It does not encrypt LDAP traffic. Instead, it helps protect LDAP communications against tampering and, when required, prevents unsigned NTLM authentication from being successfully relayed to the LDAP service.

### LDAPServerIntegrity

The `LDAPServerIntegrity` registry setting controls the LDAP signing requirement on a Domain Controller.

The relevant configuration is:

![LDAP Signing Settings](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/LDAP-Signing-Settings.png)

A value of `2` requires LDAP signing on the Domain Controller. Lower values do not require signing, meaning LDAP clients may be able to establish unsigned LDAP sessions depending on the client, authentication protocol, and configuration.

In this assessment, LDAP signing was not enforced. This configuration allowed the Domain Controller to accept the relayed NTLM authentication over LDAP.

---

## Understanding NTLM Relay

NTLM relay is not a password-cracking attack.

Instead, an attacker attempts to forward a legitimate NTLM authentication exchange from one service to another service.

The important point is that the attacker does not need to know the user's password.

The attack abuses the trust placed in the authentication exchange when the target service does not have adequate protections against relay attacks.

---

## Practical NTLM Relay

To validate the finding, I used **Impacket's `ntlmrelayx`** during the authorized assessment.

When started, `ntlmrelayx` initialized several protocol listeners capable of receiving authentication requests.

![Impacket's NTLMRelay](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/Impacket-NTLMRelay.png)

For this assessment, the HTTP listener was used to receive an NTLM authentication attempt. The listener itself did not provide the user's credentials; it received the authentication exchange and forwarded it to the LDAP service for relay.

![Performing NTLM Authentication](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/Performing-NTLM-Authentication.png)

Once the authentication request was received, `ntlmrelayx` successfully relayed the authentication connection to the LDAP service on the Domain Controller.

![NTLMRelay Output](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/NTLMRelay-Output.png)

The relayed LDAP session was then evaluated to determine whether the authenticated identity had sufficient Active Directory permissions to perform privileged directory modifications.

The assessment confirmed that the relayed identity possessed the required privileges for the observed changes.

---

## Active Directory Changes

Following the successful relay, the assessment resulted in several privileged changes within Active Directory.

The observed changes included:

- Adding the provided Domain Administrator account to the **Enterprise Admins** group
- Granting the provided Domain Administrator account **Replication-Get-Changes-All** privileges on the domain
- Creating a new machine account
- Assigning a known password to the newly created machine account

The machine-account operation was separately documented as part of the assessed attack activity and was not, by itself, the condition that granted Enterprise Admins membership.

---

## Verification

After the relay operation completed, the resulting Active Directory changes were independently verified.

To confirm that the changes reported during the relay operation were actually applied to Active Directory, the relevant directory objects, group memberships, and permissions were independently queried from a privileged PowerShell session on the Domain Controller.

![Verified Active Directory Group Membership and Permissions](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/Verified-Active-Directory-Group-Membership-and-Permissions.png)

This step was important because tool output alone should not be treated as the final proof of a successful privilege modification.

The verification confirmed that the expected Active Directory changes were present.

---

## Why LDAP Relay Was Possible

The issue was not that LDAP authentication itself was broken.

LDAP was functioning normally. The problem was the combination of several conditions:

![Performed NTLM Relay Attack Chain](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/NTLM-Relay-Attack-Chain.png)

Because LDAP signing was not enforced, the Domain Controller could accept an LDAP authentication session without the integrity protection required to prevent this form of NTLM relay.

The missing signing requirement therefore created the relay opportunity.

The subsequent impact depended on the permissions held by the relayed identity. In this assessment, those permissions were sufficient to perform privileged Active Directory modifications, including adding the account to the **Enterprise Admins** group and modifying domain-level replication permissions.

---

## Assessment Impact

The primary security impact was the ability to move from an already privileged **Domain Admin** context to **forest-level administrative privileges** through an NTLM relay attack against LDAP.

The assessment demonstrated that insufficient LDAP signing protection could be combined with the privileges of the relayed identity to perform changes affecting the wider Active Directory forest.

![Performed Attack Path](https://github.com/virtualISP/Writeups/blob/main/Assessment-Writeups/Red-Teaming/Active-Directory/DA-to-EA-LDAP-Relay/images/Performed-Attack-Path.png)

This demonstrates why LDAP signing should be treated as an important Active Directory security control rather than merely a protocol-hardening recommendation.

---

## Mitigations

Organizations should consider the following defensive measures:

### 1. Enforce LDAP Signing

Require signed LDAP authentication on Domain Controllers.

### 2. Enable EPA

Use **Extended Protection for Authentication (EPA)** where supported.

### 3. Reduce NTLM Usage

Identify and minimize unnecessary NTLM authentication across the environment.

### 4. Review Privileged Groups

Regularly audit:

- Domain Admins
- Enterprise Admins
- Other highly privileged groups

Review unexpected membership changes and excessive delegated permissions.

### 5. Monitor Directory Changes

Alert on unexpected:

- Privileged group membership changes
- ACL modifications
- Replication permission changes
- Computer account creation
- Other sensitive Active Directory object modifications

---

## Conclusion

This assessment demonstrates that Active Directory security weaknesses often arise from the interaction of multiple legitimate technologies rather than from a single vulnerability.

LDAP itself was functioning as intended. The security weakness was the failure to enforce LDAP signing, which created an opportunity for NTLM relay.

The impact was determined by the privileges available to the relayed identity. In this case, those privileges allowed privileged Active Directory modifications that resulted in forest-level administrative impact.

Enforcing LDAP signing, reducing NTLM usage, implementing EPA where appropriate, and continuously monitoring privileged Active Directory changes can significantly reduce the risk of this attack path.