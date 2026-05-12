# NOTE — Credential Hunting in Network Traffic

## ID
524

## Module
Password Attacks

## Kind
methodology

## Title
Section 18 — Credential Hunting in Network Traffic

## Description
Extract credentials from packet captures and live traffic. Wireshark display filters for HTTP/FTP/SNMP/SMTP, plus Pcredz for automated multi-protocol credential & hash carving (incl. NTLMv1/v2, Kerberos).

## Tags
wireshark, pcredz, pcap, ftp, http, snmp, smtp, ntlmv2, kerberos, credit-cards, plaintext-protocols

## Commands
- `wireshark <pcap_file>`
- `./Pcredz -f demo.pcapng -t -v`
- `./Pcredz -i <interface> -t -v` (live)

## Concept Overview
Cleartext protocols + missed TLS upgrades = passwords visible in transit. Encrypted variants (HTTPS, SFTP, SMTPS, IMAPS, LDAPS, SNMPv3) prevent this, but they're not universally enforced — legacy systems, internal services, dev environments, and IoT routinely fall back to plaintext.

## Plaintext vs Encrypted Equivalents

| Plaintext | Encrypted | Default Port (Plain → Enc) |
|-----------|-----------|----------------------------|
| HTTP | HTTPS | 80 → 443 |
| FTP | FTPS / SFTP | 21 → 990 / 22 |
| Telnet | SSH | 23 → 22 |
| SMTP | SMTPS / STARTTLS | 25 → 465 |
| POP3 | POP3S | 110 → 995 |
| IMAP | IMAPS | 143 → 993 |
| LDAP | LDAPS | 389 → 636 |
| SNMPv1/v2c | SNMPv3 | 161 (both) |
| SMB (no signing) | SMB 3 + signing | 445 |
| VNC (no TLS) | VNC + TLS | 5900 |
| RDP (no TLS) | RDP + TLS | 3389 |

## Wireshark — Display Filters

### Core Filters
| Filter | Use |
|--------|-----|
| `ip.addr == 10.0.0.1` | Traffic to/from an IP |
| `tcp.port == 80` | Port match |
| `http` | HTTP only |
| `http.request.method == "POST"` | POSTs (where creds usually are) |
| `http contains "passw"` | HTTP packets with the literal string |
| `dns` | DNS queries/responses |
| `ftp` | FTP control channel |
| `snmp` | SNMP packets |
| `tcp.flags.syn == 1 && tcp.flags.ack == 0` | SYN scans / connection attempts |
| `tcp.stream eq 53` | One TCP stream |
| `frame contains "password"` | Any frame containing the literal string |

### Finding Specific Bytes
`Edit → Find Packet` (or Ctrl-F):
- Search type: **Packet bytes** for raw binary, **Packet details** for protocol fields, **String** for ASCII
- Useful for non-HTTP plaintext (mail, custom protocols)

### Following a Stream
Right-click a packet → **Follow → TCP Stream** to see the full conversation in one window. Best for reading FTP/SMTP/HTTP flows.

## Pcredz — Automated Carving
[Pcredz](https://github.com/lgandx/PCredz) parses pcaps (and live interfaces) for credentials across many protocols.

Extracts:
- Credit card numbers (Luhn-validated)
- POP / SMTP / IMAP / FTP credentials
- SNMP community strings
- HTTP NTLM Basic auth + HTML form data
- NTLMv1/v2 hashes from DCE-RPC, SMB, LDAP, MSSQL, HTTP
- Kerberos pre-auth (AS-REQ etype 23) hashes

### Run against a pcap
```bash
./Pcredz -f demo.pcapng -t -v
```

### Live capture
```bash
sudo ./Pcredz -i eth0 -t -v
```

NTLMv2 / Kerberos hashes can be fed directly to hashcat:
- NetNTLMv2 → `-m 5600`
- Kerberos AS-REQ pre-auth → `-m 18200`

## Quick Triage Workflow for a New PCAP
1. **Open in Wireshark**, *Statistics → Protocol Hierarchy* — see what protocols are present.
2. **Filter `http.request.method == "POST"`** — log in forms first.
3. **Filter `ftp`** — look for `USER` + `PASS` commands.
4. **Filter `snmp`** — community strings are field-visible.
5. **Run Pcredz `-f`** in parallel — catches the long tail (NTLM, Kerberos, mail).
6. **Search for `password`/`passwd` via Find Packet** — covers custom protocols.

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Credit card number transmitted | **(hidden — see HTB walkthrough)** | Filter `http`, find POST to `/process_payment`, expand HTML Form URL Encoded |
| Q2 — SNMPv2 community string used | **(hidden)** | Filter `snmp`, expand any packet, read `community` field |
| Q3 — Password of the user who logged into FTP | **(hidden)** | Filter `ftp`, find `Request: PASS` line, expand FTP block |
| Q4 — File downloaded over FTP | **(hidden)** | Filter `ftp`, find `Request: RETR` packet |

## Key Takeaways
- HTTP POST bodies in Wireshark = pre-formatted form-data block at the bottom of the packet detail. Pop it open and read.
- Always run `Pcredz -f` against unknown pcaps even if you skim them in Wireshark — it catches NTLM/Kerberos hashes you'd otherwise miss.
- SNMPv1/v2c community strings are essentially passwords in plaintext. Treat their presence as a finding.
- Captured NTLMv2 hashes are *crackable* (`-m 5600` in hashcat) — relaying them via NTLMRelayx works only if SMB signing isn't enforced.

## Gotchas
- TLS-encrypted traffic looks identical to noise. You need the server's private key (or SSLKEYLOGFILE for client-side decryption) to peek inside.
- Some "encrypted" protocols still leak metadata (SNI in TLS hello, domain names in DNS, even with DoH disabled).
- Pcredz produces both hash strings *and* a single one-line summary — make sure to copy the actual `Found ...` line, not just the summary.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[17-credential-hunting-in-linux]] | [[19-credential-hunting-in-network-shares]] →
<!-- AUTO-LINKS-END -->
