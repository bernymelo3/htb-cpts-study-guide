# THEORY — Tools of the Trade

## ID
701

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 2 — Tools of the Trade

## Description
Reference guide to the essential tools used throughout this module for enumerating and attacking Active Directory — covering BloodHound, Kerbrute, Impacket, PowerView, Rubeus, and many more.

## Tags
tools, reference, enumeration, attacks, active-directory

## TL;DR — What's Important
- **BloodHound + SharpHound** are the most powerful AD privilege path mapping tools – always run them early.
- **Impacket toolkit** (Python) is the Swiss Army knife for AD attacks from Linux – includes `secretsdump`, `GetNPUsers`, `GetUserSPNs`, `ntlmrelayx`, etc.
- **Rubeus** (C#) and **PowerView** (PowerShell) are essential for attacking AD from a Windows host.
- **CrackMapExec (CME)** automates enumeration and attacks across many hosts – very efficient for spraying and checking access.
- **Kerbrute** is the go‑to for username enumeration and password spraying via Kerberos pre‑auth.
- **Responder / Inveigh** capture NetNTLM hashes via spoofing – critical for gaining footholds.

## Concept Overview
This module uses many tools, each specializing in different phases of AD attacks: initial enumeration (BloodHound, PowerView, enum4linux), credential theft (Mimikatz, Responder, Rubeus), offline cracking (Hashcat), and exploitation (Impacket, evil‑winrm, CrackMapExec). Knowing which tool to use in which situation – and from which OS (Windows vs. Linux) – is key. This reference section provides a lookup for each tool’s purpose and typical use.

## Key Concepts

### Comprehensive Tool Table

| Tool | Platform | Primary Use |
|------|----------|--------------|
| **PowerView / SharpView** | Windows (PS / .NET) | AD situational awareness; replacement for `net` commands; find users, groups, SPNs, sessions |
| **BloodHound** | Windows / Linux (Java GUI) | Visual AD privilege path mapping with Neo4j |
| **SharpHound** | Windows (C#) | BloodHound data collector – runs on Windows |
| **BloodHound.py** | Linux (Python) | Python‑based BloodHound ingestor for Linux attack hosts |
| **Kerbrute** | Linux / Windows (Go) | Username enumeration, password spraying, brute‑force via Kerberos pre‑auth |
| **Impacket toolkit** | Linux (Python) | Multi‑protocol Swiss Army knife: `secretsdump`, `GetNPUsers`, `GetUserSPNs`, `psexec`, `wmiexec`, `ntlmrelayx`, `smbserver`, more |
| **Responder** | Linux (Python) / Windows | LLMNR, NBT‑NS, MDNS poisoning – capture NetNTLM hashes |
| **Inveigh** | Windows (PowerShell / C#) | Windows equivalent of Responder |
| **CrackMapExec (CME)** | Linux (Python) | Multi‑purpose enumeration, authentication testing, command execution (SMB, WinRM, MSSQL, etc.) |
| **Rubeus** | Windows (C#) | Kerberos abuse: Kerberoasting, ASREPRoasting, pass‑the‑ticket, Kerberos golden ticket |
| **Hashcat** | Linux / Windows | Offline password cracking (cracked hashes from Kerberoasting, Responder, etc.) |
| **enum4linux / enum4linux-ng** | Linux | SMB / RPC enumeration – user, share, OS, policy info from Windows/Samba |
| **ldapsearch** | Linux | Built‑in LDAP query tool |
| **windapsearch** | Linux (Python) | Python automated LDAP enumeration |
| **DomainPasswordSpray.ps1** | Windows (PS) | Password spraying with lockout awareness |
| **LAPSToolkit** | Windows (PS) | Audit LAPS – find local admin passwords stored in AD |
| **smbmap** | Linux / Windows | Enumerate SMB shares and permissions |
| **psexec.py / wmiexec.py** | Linux (Impacket) | Remote command execution over SMB / WMI |
| **Snaffler** | Windows (C#) | Find credentials and sensitive data in file shares |
| **smbserver.py** | Linux (Impacket) | Simple SMB server for file transfers |
| **setspn.exe** | Windows (native) | Query Service Principal Names (SPNs) |
| **Mimikatz** | Windows | Extract plaintext passwords, hashes, Kerberos tickets from memory |
| **secretsdump.py** | Linux (Impacket) | Remote dump SAM / LSA secrets (NTLM hashes) |
| **evil-winrm** | Linux (Ruby) | Interactive shell over WinRM |
| **mssqlclient.py** | Linux (Impacket) | Interact with MSSQL databases |
| **noPac.py** | Linux (Python) | Exploit CVE‑2021‑42278 / CVE‑2021‑42287 – Impersonate DA |
| **ntlmrelayx.py** | Linux (Impacket) | SMB relay attacks |
| **PetitPotam.py** | Linux (Python) | Coerce Windows authentication via MS‑EFSRPC |
| **gettgtpkinit.py / getnthash.py** | Linux (Python) | Certificate‑based Kerberos attacks |
| **adidnsdump** | Linux (Python) | Dump DNS records from AD |
| **gpp-decrypt** | Linux | Decrypt Group Policy Preferences credentials |
| **GetNPUsers.py** | Linux (Impacket) | ASREPRoasting – get hashes for users with pre‑auth disabled |
| **lookupsid.py** | Linux (Impacket) | SID brute‑forcing |
| **ticketer.py** | Linux (Impacket) | Golden / Silver ticket creation |
| **raiseChild.py** | Linux (Impacket) | Child‑to‑parent domain trust escalation |
| **Active Directory Explorer** | Windows (GUI) | AD database viewer / snapshot comparison |
| **PingCastle** | Windows / Linux | AD security assessment and risk scoring |
| **Group3r** | Windows (C#) | Audit GPO security misconfigurations |
| **ADRecon** | Windows (PS) | Extract AD data to Excel with analysis |

## Why It Matters
The sheer number of AD tools can be overwhelming. Understanding what each tool does and when to reach for it is critical for efficiency. Using the wrong tool (e.g., `enum4linux` on a hardened Windows host) wastes time. Conversely, knowing that `CrackMapExec` can test credentials, run commands, and dump SAM all from one command saves enormous effort. This reference helps you quickly recall which tool is best for each phase of an AD attack.

## Key Takeaways
- **Start with BloodHound + SharpHound** – it gives you a visual map of attack paths. Collect data early, analyze later.
- **Use Kerbrute for user enumeration** – silent, logs only successes, works against Kerberos.
- **Impacket is universal** – install it on every Linux attack host. Learn its core scripts (`secretsdump`, `GetNPUsers`, `GetUserSPNs`, `ntlmrelayx`, `psexec`).
- **CrackMapExec (CME)** automates repetitive tasks – password spraying, checking admin access, running commands across many hosts.
- **Rubeus + PowerView** are your Windows power tools – if you land on a Windows domain‑joined host, these are essential.
- **Mimikatz** is the ultimate post‑exploitation tool – but often blocked. Use `sekurlsa::logonpasswords` and `sekurlsa::tickets` first.
- **Offline cracking (Hashcat)** turns captured hashes (Kerberoast, Responder, ASREPRoast) into plaintext passwords.

## Gotchas
- Many tools require specific privileges – e.g., `secretsdump` needs admin rights on the target; `GetNPUsers` works with any domain user.
- BloodHound data can be massive – filter out noise with custom queries.
- Always check tool flags – Impacket scripts use `-dc-ip` to specify a DC (avoids DNS issues), `-k` for Kerberos auth.
- Some tools (like `enum4linux`) rely on legacy protocols – may fail on modern Windows with SMB signing enabled.
- PowerShell tools may hit execution policy – bypass with `-ExecutionPolicy Bypass -Scope Process`.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[01-intro-to-ad]] | [[03-scenario]] →
<!-- AUTO-LINKS-END -->
