# NOTE — FTP

## ID
18

## Module
Footprinting

## Kind
notes

## Title
Section 6 — FTP

## Description
TCP/21 control + TCP/20 data (active mode), passive mode for firewalled clients. Anonymous login, recursive listing, file up/download via vsFTPd; banner grabbing via nc/telnet/openssl s_client.

## Tags
ftp, vsftpd, tftp, anonymous, ssl, openssl, nc, nmap, ftp-anon, ftp-syst

## Commands
- `ftp <IP>` — interactive client (anonymous login if allowed)
- `ftp> status` — show client/server settings
- `ftp> debug` / `ftp> trace` — verbose protocol output
- `ftp> ls -R` — recursive listing (only if `ls_recurse_enable=YES`)
- `ftp> get <file>` / `put <file>` — download/upload
- `wget -m --no-passive ftp://anonymous:anonymous@<IP>` — mirror entire share
- `sudo nmap -sV -p21 -sC -A <IP>` — version + default scripts
- `sudo nmap -sV -p21 -sC -A <IP> --script-trace` — see Nmap's NSE traffic
- `nc -nv <IP> 21` — banner grab
- `telnet <IP> 21` — manual interaction
- `openssl s_client -connect <IP>:21 -starttls ftp` — TLS-wrapped FTP, also dumps cert

## Concept Overview
FTP is one of the oldest application-layer protocols. Two channels: control (TCP/21) and data (TCP/20 active, or server-announced port in passive mode). Cleartext by default — sniffable. **TFTP** is the simpler UDP cousin (no auth, no listing — operates only on world-readable files), used in trusted local nets only.

## Key Configuration File — vsFTPd
`/etc/vsftpd.conf` — the main settings file. `/etc/ftpusers` lists users **denied** login (even if they exist on the system).

### Important Settings (Default & Anonymous)
| Setting | Meaning |
|---------|---------|
| `anonymous_enable=YES` | Allow anonymous login |
| `anon_upload_enable=YES` | Allow anonymous upload |
| `anon_mkdir_write_enable=YES` | Anon can create dirs |
| `no_anon_password=YES` | Don't prompt for anon password |
| `anon_root=/home/.../ftp` | Anon root directory |
| `write_enable=YES` | Allows STOR/DELE/RNFR/RNTO/MKD/RMD/APPE/SITE |

### Dangerous / Information-Leaking Settings
| Setting | Why it helps you |
|---------|------------------|
| `anonymous_enable=YES` | Free read access — try first |
| `ls_recurse_enable=YES` | `ls -R` dumps entire tree in one shot |
| `hide_ids=YES` | UIDs/GIDs masked as `ftp` — hides who owns what |
| `chown_uploads=YES` + `chown_username=<user>` | Anon uploads silently chowned to a real user — useful for attacks via cron/web-server pickup |
| `chroot_local_user=YES` | Locks local users to home — but exceptions list weakens this |

## Methodology
1. **Banner-grab first** — `nc -nv <IP> 21` returns `220 <banner>` with often the server type/version.
2. **Try anonymous** — `ftp <IP>`, user `anonymous`, password anything (often `anonymous`).
3. **Inside the session, run `status`** to fingerprint the server's configuration.
4. **`ls -R`** if recursive listing is allowed — single shot for the whole tree.
5. **Mirror with wget** if the tree is large: `wget -m --no-passive ftp://anonymous:anonymous@<IP>` — creates `<IP>/` locally.
6. **Test write** — `touch local.txt; put local.txt`. If allowed and the share is web-rooted, you have a webshell upload primitive.
7. **TLS-wrapped FTP** — use `openssl s_client -connect <IP>:21 -starttls ftp` to grab the SSL cert (CN/email/org → more recon).

## Important NSE Scripts
| Script | What it does |
|--------|-------------|
| `ftp-anon` | Tests anonymous login, lists root if successful |
| `ftp-syst` | Runs `STAT` — server status, version, session info |
| `ftp-vsftpd-backdoor` | Checks vsFTPd 2.3.4 backdoor |
| `ftp-vuln-cve2010-4221` | ProFTPD vuln |
| `ftp-proftpd-backdoor` | ProFTPD 1.3.3c backdoor |
| `ftp-libopie` | OPIE off-by-one |
| `ftp-bounce` | FTP bounce attack test |
| `ftp-brute` | Credential brute force |

Update DB: `sudo nmap --script-updatedb`. Scripts live at `/usr/share/nmap/scripts/ftp-*.nse`.

## Banner Grab Reference
| Code | Meaning |
|------|---------|
| 220 | Service ready (banner) |
| 230 | Login successful |
| 200 | Command OK |
| 226 | Closing data connection (transfer complete) |
| 530 | Not logged in |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Version of the FTP server (full banner) | **`InFreight FTP v1.1`** | `sudo nmap -p21 -sV --disable-arp-ping -n --packet-trace <IP>` shows banner `220 InFreight FTP v1.1`. Same banner from `ftp <IP>` on connect. |
| Q2 — Contents of `flag.txt` | **`HTB{b7skjr4c76zhsds7fzhd4k3ujg7nhdjre}`** | `ftp <IP>` → login as `anonymous` (password `anonymous`) → `get flag.txt` → `!cat flag.txt`. |

## Key Takeaways
- **Anonymous + `ls -R` + `wget -m`** answers most boxes' "what files are on this FTP" questions.
- `openssl s_client -connect <IP>:21 -starttls ftp` grabs the SSL cert (CN, email) — extra recon for free.
- `--script-trace` lets you see exactly what NSE is doing — useful when scripts return ambiguous output.
- Write-able anonymous FTP + web-rooted share = RCE primitive. Always test `put`.

## Gotchas
- TFTP doesn't list directories — you must already know the filename to `get`.
- Active-mode FTP (`PORT`) breaks behind NAT — switch to passive (`PASV`).
- `hide_ids=YES` makes everything look owned by `ftp` — don't take ownership at face value.
- `wget -m` without `--no-passive` may stall behind some firewalls; the inverse is also true. Try both.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[05-staff]] | [[07-smb]] →
<!-- AUTO-LINKS-END -->
