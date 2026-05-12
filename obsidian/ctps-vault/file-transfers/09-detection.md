# THEORY — Detection

## ID
623

## Module
File Transfers

## Kind
theory

## Title
Section 9 — Detection

## Description
Reference list of User-Agent strings emitted by common Windows HTTP transfer techniques (Invoke-WebRequest, WinHttp, Msxml2, certutil, BITS) so defenders can fingerprint them and attackers know what they're leaking.

## Tags
detection, user-agent, defender, blue-team, fingerprinting, threat-hunting, ioc

## TL;DR — What's Important
- Every Windows HTTP transfer method emits a **distinctive User-Agent** — defenders can fingerprint the *tool* without ever seeing the payload.
- Blacklisting command lines is brittle (case obfuscation defeats it). **Whitelisting known-good UAs is robust** but takes effort to build.
- Know the UAs you leak by default so you can spoof them when needed (see [[10-evading-detection]]).

## User-Agent Reference — Windows HTTP Clients

### Invoke-WebRequest / Invoke-RestMethod
```
User-Agent: Mozilla/5.0 (Windows NT; Windows NT 10.0; en-US) WindowsPowerShell/5.1.14393.0
```
**Tell:** `WindowsPowerShell/x.x.x.x` — dead giveaway.

### `WinHttp.WinHttpRequest.5.1` (COM)
```
User-Agent: Mozilla/4.0 (compatible; Win32; WinHttp.WinHttpRequest.5)
```
**Tell:** the literal string `WinHttp.WinHttpRequest.5`.

### `Msxml2.XMLHTTP` (COM)
```
User-Agent: Mozilla/4.0 (compatible; MSIE 7.0; Windows NT 10.0; Win64; x64; Trident/7.0; .NET4.0C; .NET4.0E)
```
**Tell:** `MSIE 7.0` + `Trident/7.0` — IE-like but with `.NET4.0C/E` tokens, an oddity for real users.

### `certutil.exe`
```
User-Agent: Microsoft-CryptoAPI/10.0
```
**Tell:** legit for cert revocation checks, but `Microsoft-CryptoAPI` fetching `.exe` files is a screaming anomaly.

### BITS (`Start-BitsTransfer`, `bitsadmin`)
```
User-Agent: Microsoft BITS/7.8
```
First request is a `HEAD`, then `GET` — defenders can correlate the pair.

## Detection Building Blocks

### 1. Command-line whitelisting (robust)
- Build a list of known-good command lines per host role.
- Alert on anything not on the list.
- **Beats blacklisting** because case obfuscation (`CErTuTiL`) won't slip past.

### 2. User-Agent allowlisting
- Build org's list of legitimate UAs:
  - Browsers (Chrome, Edge, Firefox versions you license).
  - Windows Update / WSUS UAs.
  - AV/EDR update UAs.
  - LOB applications.
- Anything else → SIEM alert / threat hunt.

### 3. Anomalous content-type correlation
- `Microsoft-CryptoAPI` fetching `.exe` instead of `.crl` / `.cer` → suspicious.
- `Microsoft BITS` fetching from non-Microsoft, non-corp domains → suspicious.
- `WindowsPowerShell` UA on a host that's not an admin workstation → suspicious.

### 4. Known-bad reference lists
- [useragentstring.com](http://useragentstring.com) — common legitimate UAs.
- Public IOC / threat intel feeds for known-bad UAs.

## Detection Difficulty Ranking

| Transfer Method | UA Stealth | Notes |
|-----------------|-----------|-------|
| `Invoke-WebRequest` (no spoof) | ❌ Loud | "WindowsPowerShell/x.x" in UA |
| `certutil -urlcache` | ❌ Loud | "Microsoft-CryptoAPI" fetching binaries |
| BITS | ⚠️ Medium | Looks like Windows Update if domains match |
| `WinHttpRequest.5.1` | ⚠️ Medium | Distinctive UA but lower volume than PS |
| `Msxml2.XMLHTTP` | ⚠️ Medium | IE7 UA is uncommon on modern hosts |
| Spoofed UA (Chrome/Firefox via `-UserAgent`) | ✅ Stealthy | Blends with real browser traffic |

## What Defenders Should Hunt For

1. PowerShell / cscript / certutil / bitsadmin processes spawning network connections.
2. HTTP requests with non-allowed UAs.
3. `Microsoft-CryptoAPI` UA fetching anything other than `.crl`/`.cer`.
4. BITS jobs sourcing from non-Microsoft domains.
5. Long-lived BITS jobs that never complete (persistence).

## Key Takeaways
- Every HTTP cradle has a UA fingerprint by default — assume defenders know them all.
- Command-line *blacklists* are trivially bypassed (case obfuscation, parameter substitution). **Whitelists** are the real defense.
- BITS is the most "blendy" technique only when sourcing from MS-looking domains — attacker infra on `attacker.tk` makes it loud.
- Even small details — `Accept-Language: en-us`, `UA-CPU: AMD64` (Msxml2) — make method attribution easy for blue teams.

## Gotchas
- UA spoofing only helps with HTTP — process-level telemetry (Sysmon Event ID 1, EDR process trees) still sees `powershell.exe` spawning a network connection.
- Some EDRs strip/normalize UAs before comparison — assume defenders see the raw UA.
- `Mozilla/4.0` (not 5.0) is itself a UA tell — most modern browsers are 5.0+.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[08-living-off-the-land]] | [[10-evading-detection]] →
<!-- AUTO-LINKS-END -->
