# NOTE — SNMP

## ID
105

## Module
Footprinting

## Kind
notes

## Title
Section 12 — SNMP

## Description
SNMPv1/v2c are unauthenticated and rely on a plaintext community string. Walk OIDs with snmpwalk, brute community strings with onesixtyone, and brute OIDs with braa to extract installed packages, processes, and system metadata.

## Tags
snmp, snmpv2c, snmpv3, mib, oid, snmpwalk, onesixtyone, braa, community-string

## Commands
- `snmpwalk -v2c -c <community> <IP>` — walk the OID tree
- `snmpwalk -v2c -c public <IP>` — try `public` first
- `sudo apt install onesixtyone`
- `onesixtyone -c /opt/useful/seclists/Discovery/SNMP/snmp.txt <IP>` — community brute
- `sudo apt install braa`
- `braa <community>@<IP>:.1.3.6.*` — brute OIDs once a community is known

## Concept Overview
SNMP = remote monitor + manage network gear (routers, switches, servers, IoT, printers, UPS). Agent listens on **UDP/161**; **traps** sent server→client on **UDP/162**. **MIBs** are tree definitions of what data is exposable; **OIDs** are dotted addresses inside that tree (e.g. `.1.3.6.1.2.1.1.1.0` = sysDescr).

### Versions
| Version | Auth | Crypto | Notes |
|---------|------|--------|-------|
| **v1** | None | None | Plaintext only |
| **v2c** | Community string only | None | "Community-based" — most common in the wild despite being insecure |
| **v3** | User + password (and authPriv) | Yes | Much harder to misuse — and much harder to configure |

**Community strings** = passwords-in-name-only, sent in cleartext. Two flavors:
- **Read-only (`ro`)** — query OIDs.
- **Read-write (`rw`)** — *change* OID values (dangerous: reconfigure devices, plant traps, etc.).

## Default Configuration — `/etc/snmp/snmpd.conf`
```
sysLocation    Sitting on the Dock of the Bay
sysContact     Me <me@example.org>
sysServices    72
master  agentx
agentaddress  127.0.0.1,[::1]
view   systemonly  included   .1.3.6.1.2.1.1
view   systemonly  included   .1.3.6.1.2.1.25.1
rocommunity  public default -V systemonly
rocommunity6 public default -V systemonly
rouser authPrivUser authpriv -V systemonly
```
Default community `public`, scoped to the `systemonly` view (sysDescr + hrSystem). Restrict views to limit blast radius.

### Dangerous Settings
| Setting | Why dangerous |
|---------|--------------|
| `rwuser noauth` | Full RW tree, **no auth** |
| `rwcommunity <str> <IPv4>` | RW tree regardless of source IP |
| `rwcommunity6 <str> <IPv6>` | Same, IPv6 |

## Methodology
1. **Try `public` first** — `snmpwalk -v2c -c public <IP>`. Returns sysDescr (kernel + distro), uptime, contact, location, all hrSystem entries.
2. **Brute the community string** — `onesixtyone -c <wordlist> <IP>` is fast (UDP). Use SecLists `snmp.txt`.
3. **Try named patterns** — companies often use hostname-based or abbreviated community names. Generate with `crunch` if needed.
4. **Once a community is found, walk it** — full `snmpwalk` for context.
5. **Brute OIDs with braa** for fast bulk reading: `braa <community>@<IP>:.1.3.6.*`.
6. **Note installed packages** — OID `.1.3.6.1.2.1.25.6.3.1.2.<n>` lists `hrSWInstalledName` — the full package list with versions. Map to known CVEs.

## Useful OIDs to Remember
| OID | What it returns |
|-----|----------------|
| `.1.3.6.1.2.1.1.1.0` | sysDescr — kernel + arch + distro |
| `.1.3.6.1.2.1.1.4.0` | sysContact — admin email |
| `.1.3.6.1.2.1.1.5.0` | sysName — hostname |
| `.1.3.6.1.2.1.1.6.0` | sysLocation |
| `.1.3.6.1.2.1.25.1.4.0` | Boot command line (kernel + UUID) |
| `.1.3.6.1.2.1.25.6.3.1.2.*` | Installed software |
| `.1.3.6.1.2.1.25.4.2.1.2.*` | Running processes |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Admin email address | **`devadmin@inlanefreight.htb`** | `snmpwalk -v2c -c public <IP> | tee SNMPWalk.txt` then `grep "@inlanefreight" SNMPWalk.txt` → `iso.3.6.1.2.1.1.4.0 = STRING: "devadmin <devadmin@inlanefreight.htb>"` (sysContact OID) |
| Q2 — Customised SNMP version | **`InFreight SNMP v0.91`** | Same snmpwalk output — `iso.3.6.1.2.1.1.6.0 = STRING: "InFreight SNMP v0.91"` (sysLocation field, abused as version banner) |
| Q3 — Output of the custom script | **`HTB{5nMp_fl4g_uidhfljnsldiuhbfsdij44738b2u763g}`** | `grep -m 1 -B 8 "HTB" SNMPWalk.txt` → `/usr/share/flag.sh` script's output is exposed via the hrSWRunPerf table at OID `iso.3.6.1.2.1.25.1.7.1.3.1.1.4.70.76.65.71` |

### Reference command
```bash
snmpwalk -v2c -c public <IP> | tee SNMPWalk.txt
grep -m 1 -B 8 "HTB" SNMPWalk.txt
```

## Key Takeaways
- **`public`/`private` are tried first** — admins still leave defaults in place often enough.
- onesixtyone is fast because UDP needs no handshake. Use a big SecLists wordlist.
- The installed-software OID alone often reveals an outdated package with a public CVE.
- An RW community is much rarer but devastating — you can re-key the device.

## Gotchas
- SNMPv3 is incompatible with v1/v2c queries; trying v2c against a v3-only daemon = silent failure.
- onesixtyone is UDP — packet loss can cause false negatives. Re-run with `-w 100` (delay) and bigger retries on lossy networks.
- Some agents scope reads to a `view` — `snmpwalk` against the right OID-prefix may be allowed, but a full `.1` walk denied.
- Some IoT devices ship with hardcoded RW communities that can't be changed.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[11-imap-pop3]] | [[13-mysql]] →
<!-- AUTO-LINKS-END -->
