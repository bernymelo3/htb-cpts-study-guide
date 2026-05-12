# NOTE — SMTP

## ID
27

## Module
Footprinting

## Kind
notes

## Title
Section 10 — SMTP

## Description
SMTP/ESMTP enumeration and exploitation: VRFY user enumeration, manual mail crafting via telnet, open-relay testing, and Postfix dangerous settings (especially `mynetworks = 0.0.0.0/0`).

## Tags
smtp, esmtp, postfix, vrfy, mail-spoofing, open-relay, mta, mua, msa, telnet

## Commands
- `telnet <IP> 25` — TCP/25 control session
- `HELO <hostname>` / `EHLO <hostname>` — start session (EHLO = extended)
- `VRFY <user>` — username probe (status 252 ≠ exists; depends on config)
- `MAIL FROM: <sender@domain>` / `RCPT TO: <recipient@domain> NOTIFY=success,failure`
- `DATA` → `From: ...` / `To: ...` / `Subject: ...` / `<body>` / `.`
- `RSET`, `QUIT`
- `sudo nmap <IP> -sC -sV -p25` — banner + commands list
- `sudo nmap <IP> -p25 --script smtp-open-relay -v` — 16-test open-relay check
- Through web proxy: `CONNECT <IP>:25 HTTP/1.0`

## Concept Overview
SMTP = email submission/transport. Default TCP/25 (server-to-server, often). TCP/587 = submission with STARTTLS for authenticated clients. TCP/465 = legacy implicit-TLS. ESMTP adds STARTTLS, AUTH PLAIN, etc.

### Mail Path
**Client (MUA)** → **Submission Agent (MSA)** → **Open Relay (MTA)** → **Mail Delivery Agent (MDA)** → **Mailbox (POP3/IMAP)**

The MSA validates origin; the MTA forwards; the MDA delivers locally. **Open-relay attacks** exploit MTAs that don't restrict who can submit mail to be relayed onward.

### Built-in Weaknesses
1. No usable delivery confirmation by default.
2. No connection-level sender authentication — sender field is whatever you say it is. Anti-spoofing is bolted-on (SPF/DKIM/DMARC).

## Postfix — Default Config Pointers
`/etc/postfix/main.cf` — key fields:
| Field | Meaning |
|-------|---------|
| `smtpd_banner = ESMTP Server` | Banner string returned on connect |
| `myhostname = mail1.<domain>` | Server hostname |
| `mydestination` | Domains accepted for local delivery |
| `mynetworks = 127.0.0.0/8 10.x/16` | Source networks allowed to relay |
| `smtpd_helo_restrictions = reject_invalid_hostname` | EHLO sanity check |

### Dangerous Setting (open-relay)
```
mynetworks = 0.0.0.0/0
```
Anything on the internet can submit mail to this MTA → spoof internal employees, full open-relay abuse.

## SMTP Command Reference
| Command | Purpose |
|---------|---------|
| `HELO`/`EHLO` | Start session (EHLO advertises extensions) |
| `AUTH PLAIN` | Authenticate (after STARTTLS) |
| `MAIL FROM` | Sender |
| `RCPT TO` | Recipient (one per command — repeat for multiple) |
| `DATA` | Begin message body — terminate with `.` on a line by itself |
| `RSET` | Abort current transaction (keep connection) |
| `VRFY <user>` | Verify a user exists |
| `EXPN <user>` | Expand a mailing list / alias |
| `NOOP` | Keepalive |
| `QUIT` | Close |

### Status Codes (subset)
| Code | Meaning |
|------|---------|
| 220 | Service ready |
| 250 | OK |
| 252 | Cannot VRFY but will accept message — **NOT a user-exists confirmation** |
| 354 | Start mail input |
| 421 | Service unavailable |
| 530 | Auth required |

## VRFY for User Enumeration
```
VRFY root
252 2.0.0 root
VRFY aaaaaaaaaaaaaa
252 2.0.0 aaaaaaaaaaaaaa
```
**Status 252 means "I'll accept the message, but I'm not telling you whether the user exists."** Many Postfix configs default here. Don't blindly trust 252 = exists. Test against a known-bad name first to baseline.

## Send a Spoofed Mail Manually (Open Relay)
```
EHLO inlanefreight.htb
MAIL FROM: <attacker@spoofed.htb>
RCPT TO:   <victim@target.htb> NOTIFY=success,failure
DATA
From: <attacker@spoofed.htb>
To:   <victim@target.htb>
Subject: <subject>
Date: Tue, 28 Sept 2021 16:32:51 +0200

<body>
.
QUIT
```

## Mail Header Forensics
Headers are mandatory or optional. **RFC 5322** defines structure. They reveal sender/recipient, timestamps, every relay hop the mail passed through (`Received:` chain), content type, and signing (DKIM-Signature). Both ends can read them, though most MUAs hide them by default.

## Important NSE Scripts
| Script | Purpose |
|--------|---------|
| `smtp-commands` | Lists supported commands via EHLO |
| `smtp-open-relay` | 16 different relay-bypass tests |
| `smtp-enum-users` | VRFY/EXPN/RCPT user enum |
| `smtp-vuln-cve2010-4344` / `cve2011-1764` / `cve2011-1720` | Known CVE checks |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Banner including version | **`InFreight ESMTP v2.11`** | `telnet <IP> 25` → server responds `220 InFreight ESMTP v2.11` |
| Q2 — Username that exists on the system | **`robin`** | Download HTB Academy `Footprinting-wordlist.zip` (Resources tab), unzip, then `smtp-user-enum -M VRFY -U ./footprinting-wordlist.txt -t <IP> -m 60 -w 20` → `10.x.x.x: robin exists` |

### Reference command
```bash
smtp-user-enum -M VRFY -U ./footprinting-wordlist.txt -t <IP> -m 60 -w 20
```

## Key Takeaways
- VRFY status 252 ≠ user exists with Postfix defaults. Always baseline with garbage input.
- `mynetworks = 0.0.0.0/0` is the open-relay smoking gun — Nmap `smtp-open-relay` confirms with 16 tests.
- Manual EHLO is faster than scripting for one-off probes — and you see exactly what the server sent back.
- The banner is the OS hint — Postfix on Ubuntu often shows `(Ubuntu)`.

## Gotchas
- Some servers only return useful info **after** EHLO (not HELO).
- SMTPS (TCP/465) is implicit TLS — you need `openssl s_client -connect <IP>:465` to talk to it.
- STARTTLS on TCP/25 or 587 is "upgrade in place" — `openssl s_client -connect <IP>:25 -starttls smtp`.
- VRFY/EXPN are often disabled or rate-limited on hardened servers — don't burn the connection on them.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[09-dns]] | [[11-imap-pop3]] →
<!-- AUTO-LINKS-END -->
