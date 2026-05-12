# NOTE — IMAP / POP3

## ID
103

## Module
Footprinting

## Kind
notes

## Title
Section 11 — IMAP / POP3

## Description
Mailbox-access protocols. IMAP (143/993) for online mailbox management; POP3 (110/995) for download-only. Auth via curl/openssl s_client; auth_debug + verbose passwords are common Dovecot footguns.

## Tags
imap, pop3, dovecot, ssl, tls, curl, openssl, mailbox, mail-clients

## Commands
- `sudo nmap <IP> -sV -p110,143,993,995 -sC` — discover + cert dump
- `curl -k 'imaps://<IP>' --user <user>:<pass>` — list folders
- `curl -k 'imaps://<IP>' --user <user>:<pass> -v` — verbose (TLS + cert + protocol)
- `openssl s_client -connect <IP>:imaps` — manual IMAPS
- `openssl s_client -connect <IP>:pop3s` — manual POP3S

## Concept Overview
- **IMAP** (Internet Message Access Protocol) — server-side mailbox management; folders, multi-client sync, online or offline-with-cache. Default TCP/143, TLS on TCP/993.
- **POP3** — download-and-(optionally-)delete; no folders, no server-side state worth speaking of. TCP/110, TLS on TCP/995.
- Both use plaintext commands without TLS — sniffable.
- SMTP sends, IMAP/POP3 receive. Often the same Dovecot install handles both 110 and 143.

## IMAP Commands (sequence-tagged: `1 LOGIN ...`)
| Command | Effect |
|---------|--------|
| `1 LOGIN <user> <pass>` | Auth |
| `1 LIST "" *` | List all mailboxes/folders |
| `1 LSUB "" *` | List subscribed folders |
| `1 CREATE "<box>"` | Create mailbox |
| `1 DELETE "<box>"` | Delete mailbox |
| `1 RENAME "<old>" "<new>"` | Rename |
| `1 SELECT INBOX` | Open mailbox |
| `1 UNSELECT INBOX` | Close current |
| `1 FETCH <id> all` | Fetch a message |
| `1 CLOSE` | Expunge \Deleted msgs |
| `1 LOGOUT` | Disconnect |

## POP3 Commands
| Command | Effect |
|---------|--------|
| `USER <user>` | Username |
| `PASS <pass>` | Password |
| `STAT` | Mailbox stats |
| `LIST` | Sizes of all messages |
| `RETR <id>` | Get message |
| `DELE <id>` | Mark for deletion |
| `CAPA` | Server capabilities |
| `RSET` | Reset (un-delete) |
| `QUIT` | Disconnect |

## Dovecot Dangerous Settings
| Setting | Why dangerous |
|---------|--------------|
| `auth_debug` | Logs all auth flow |
| `auth_debug_passwords` | Logs **submitted passwords** (and scheme) — read access to log = creds |
| `auth_verbose` | Logs failed auth + reasons (helps password spraying) |
| `auth_verbose_passwords` | Failed-auth passwords logged (truncated) |
| `auth_anonymous_username` | Username used for ANONYMOUS SASL — opens an anon-login door |

If you can read `/var/log/dovecot.log` (e.g. via LFI or shell), `auth_debug_passwords` literally hands you valid creds.

## Methodology
1. **Nmap version + scripts** — banner gives Dovecot version + capabilities; cert section gives CN/email/org.
2. **`curl -k 'imaps://<IP>' --user <user>:<pass>` ** — fastest folder list when you have creds.
3. **`-v` variant** — confirms TLS version, cipher, cert, and shows the underlying CAPABILITY/AUTHENTICATE/LIST exchange.
4. **`openssl s_client -connect <IP>:imaps`** — drop into a manual TLS session, hand-type IMAP commands.
5. **Try POP3S in parallel** — same Dovecot, both ports may yield different visibility.

## Cert Dump = Bonus Recon
The `openssl s_client` cert dump usually reveals:
- `CN` = mail server hostname
- `O` = organization
- `OU` = department
- `emailAddress` = admin or department mailbox

Hostname + admin email feed back into other modules (DNS, OSINT, password spraying).

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Exact organization name | **`Inlanefreight Ltd`** | `sudo nmap -p110,143,993,995 -sC -sV <IP>` → SSL cert `organizationName=InlaneFreight Ltd` |
| Q2 — FQDN the IMAP/POP3 servers are assigned to | **`dev.inlanefreight.htb`** | Same Nmap output, SSL cert `commonName=dev.inlanefreight.htb` |
| Q3 — Flag from IMAP service | **`HTB{roncfbw7iszerd7shni7jr2343zhrj}`** | `openssl s_client -connect <IP>:imaps` → flag is **appended to the CAPABILITY line** (`* OK [CAPABILITY ... AUTH=PLAIN] HTB{...}`) |
| Q4 — Customised POP3 version | **`InFreight POP3 v9.188`** | `telnet <IP> 110` → `+OK InFreight POP3 v9.188` |
| Q5 — Admin email address | **`devadmin@inlanefreight.htb`** | `openssl s_client -connect <IP>:imaps` → `tag0 LOGIN robin robin` → `tag1 LIST "" "*"` → `tag2 SELECT "DEV.DEPARTMENT.INT"` → `tag3 FETCH 1 (BODY[])` → email `From: CTO <devadmin@inlanefreight.htb>` |
| Q6 — Flag from inside the email | **`HTB{983uzn8jmfgpd8jmof8c34n7zio}`** | Same `FETCH 1 (BODY[])` body contains the flag |

### Critical Trick
The Q3 flag is hidden **inside the unauth CAPABILITY response** — no login required. Look at the very end of `* OK [CAPABILITY ...]` line. Easy to miss when scanning past it.

## Key Takeaways
- **TLS cert is free recon** — `openssl s_client -connect <IP>:imaps` then read the subject.
- `curl -k 'imaps://...'` is the cheapest "do these creds work" test.
- Same creds across SMTP/IMAP/POP3 is the norm — re-test creds across services.
- `auth_debug_passwords = yes` + readable log = trivial cred harvest.

## Gotchas
- Capabilities exposed *before* login are the "pre-login" set; more become available *after* login. Don't rely on the unauth list to decide what's possible.
- `-k` skips cert verification (self-signed is common in lab/internal setups). Production use should not.
- POP3 `DELE` is permanent after `QUIT` — be careful in real engagements.
- Some servers force STARTTLS on TCP/143 (no implicit TLS) — `openssl s_client -starttls imap` instead of pointing at 993.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[10-smtp]] | [[12-snmp]] →
<!-- AUTO-LINKS-END -->
