# NOTE — Windows Authentication Process

## ID
511

## Module
Password Attacks

## Kind
theory

## Title
Section 10 — Windows Authentication Process

## Description
How Windows local & domain auth works: WinLogon → LogonUI → LSASS → SAM (local) or NTDS.dit (domain). Maps each component to the attack surface it exposes.

## Tags
windows, authentication, lsass, sam, ntds, winlogon, kerberos, ntlm, credential-manager, dpapi

## TL;DR — What's Important
- **LSASS** = the gatekeeper. Everything credential-related on Windows flows through it. Dumping it = jackpot.
- **SAM** = local accounts (per machine). **NTDS.dit** = domain accounts (on every DC).
- **Both files are encrypted at rest** — you also need the corresponding SYSTEM/bootkey to extract hashes.
- **Credential Manager** + **DPAPI** = persistent stored creds (browsers, RDP, Outlook).

## The Components

| Component | Path | Purpose |
|-----------|------|---------|
| **WinLogon** | `winlogon.exe` | Captures keyboard logon, calls LogonUI, hands creds to LSASS |
| **LogonUI** | `logonui.exe` | Renders the login screen, calls credential providers (DLLs) |
| **LSASS** | `lsass.exe` (`%SystemRoot%\System32\Lsass.exe`) | Enforces local security policy, performs auth, caches creds in memory |
| **SAM** | `%SystemRoot%\system32\config\SAM` (hive `HKLM\SAM`) | Local account password hashes |
| **SECURITY** | hive `HKLM\SECURITY` | LSA secrets, cached domain logons (DCC2), DPAPI keys |
| **SYSTEM** | hive `HKLM\SYSTEM` | Boot key needed to decrypt the SAM database |
| **NTDS.dit** | `%SystemRoot%\NTDS\NTDS.dit` (DCs only) | Domain user hashes, group memberships, GPOs, replication metadata |
| **Credential Manager** | `%UserProfile%\AppData\Local\Microsoft\Credentials\` | Stored web + Windows creds |

## Authentication Packages (DLLs Loaded by LSASS)

| DLL | Role |
|-----|------|
| `lsasrv.dll` | LSA Server — security policy + protocol negotiation (NTLM vs Kerberos) |
| `msv1_0.dll` | MSV1_0 — local interactive logon (NTLM) |
| `samsrv.dll` | SAM management |
| `kerberos.dll` | Kerberos authentication |
| `netlogon.dll` | Network logon |
| `ntdsa.dll` | Directory System Agent (only on DCs) — manages NTDS.dit + LDAP + replication |

## Local vs Domain Logon Flow

**Workgroup machine:**
```
keyboard → WinLogon → LSASS → msv1_0.dll → SAM database
```

**Domain-joined machine (online):**
```
keyboard → WinLogon → LSASS → kerberos.dll → DC → NTDS.dit
```

**Domain-joined machine (offline / DC unreachable):**
```
keyboard → WinLogon → LSASS → DCC2 cache (HKLM\SECURITY)
```
Cached domain logon hashes (`DCC2`/`MSCash2`) live in `HKLM\SECURITY`. They're slow to crack (PBKDF2) and *can't* be used for PtH.

## Hash Storage Forms

| Hash | Where | Notes |
|------|-------|-------|
| **LM** | SAM, NTDS (legacy) | Old, weak. Disabled by default since Vista. Always cracks quickly if present. |
| **NTLM (NT)** | SAM, NTDS, memory | The hash. Usable in Pass-the-Hash attacks. |
| **DCC2 (MS-Cache v2)** | `HKLM\SECURITY` | Cached domain logon. Cannot PtH. Slow PBKDF2. |
| **Kerberos AES256/RC4** | NTDS (DCs), memory | Kerberos pre-auth keys. RC4 is the user's NTLM hash. |

## SAM Encryption
SAM hashes are encrypted with the **boot key** (derived from `HKLM\SYSTEM`). To extract offline, you need *both* hives: `SAM` + `SYSTEM`. Microsoft added this in NT 4.0 as "SYSKEY".

## Where Each Attack Targets
| Section | Targets |
|---------|---------|
| [[11-attacking-sam-system-security]] | SAM + SYSTEM + SECURITY hives |
| [[12-attacking-lsass]] | LSASS memory dump |
| [[13-attacking-windows-credential-manager]] | Credential Manager vaults + DPAPI |
| [[14-attacking-active-directory-and-ntds]] | NTDS.dit (DCs only) |

## Key Takeaways
- Local admin on a workstation → SAM, LSASS, Credential Manager are all reachable.
- Domain admin (or DCSync rights) → NTDS.dit is the full prize.
- Cached creds in `HKLM\SECURITY` are how laptops let you log in on the plane — and how attackers get partial domain creds without DC access.
- LSASS process memory contains plaintext passwords *only* if WDigest is enabled (default on Win Server ≤2012). On modern OS you get NTLM hashes + Kerberos keys/tickets.

## Gotchas
- Credential Guard (Win10/11 enterprise) moves LSASS secrets into a VTL1-protected enclave — Mimikatz can't read it without bypasses.
- "SAM dump" requires both the SAM *and* SYSTEM hive for offline extraction.
- `ntds.dit` is locked while AD DS is running — must use VSS shadow copy or `ntdsutil ifm` to grab a usable copy.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[09-spraying-stuffing-defaults]] | [[11-attacking-sam-system-security]] →
<!-- AUTO-LINKS-END -->
