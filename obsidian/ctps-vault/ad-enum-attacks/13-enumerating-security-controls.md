# NOTE — Enumerating Security Controls

## ID
813  <!-- Confirm the next free number in the Active Directory Enumeration & Attacks range -->

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 13 — Enumerating Security Controls

## Description
Enumerate Windows Defender, AppLocker, PowerShell Constrained Language Mode, and LAPS from a foothold to understand the defensive landscape and inform tool selection for the rest of the assessment.

## Tags
defender, applocker, laps, powershell, enumeration, security-controls

## Commands
- Get-MpComputerStatus
- Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
- $ExecutionContext.SessionState.LanguageMode
- Find-LAPSDelegatedGroups
- Find-AdmPwdExtendedRights
- Get-LAPSComputers

## What This Section Covers
After gaining initial foothold, mapping the defensive controls in place is critical — the tools you can use, the noise you'll make, and the attack paths available all depend on what's enforced. This section covers four key controls: Windows Defender (real-time protection status), AppLocker (application whitelisting rules that may block PowerShell or cmd.exe), PowerShell Constrained Language Mode (restricts .NET types, COM objects, and scripting capabilities), and LAPS (Local Administrator Password Solution — who can read those randomized local admin passwords). Understanding these controls lets you choose the right tools and avoid triggering alerts.

## Methodology
1. **Check Windows Defender status** — verify if real-time protection is active:
   `Get-MpComputerStatus`
   Look for `RealTimeProtectionEnabled : True`.

2. **Enumerate AppLocker rules** — check which executables and paths are blocked/whitelisted:
   `Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections`
   Common blocks: `%SYSTEM32%\WINDOWSPOWERSHELL\V1.0\POWERSHELL.EXE`. Bypass by calling PowerShell from `SysWOW64` or other locations.

3. **Determine PowerShell Language Mode** — confirm whether you're in Full or Constrained mode:
   `$ExecutionContext.SessionState.LanguageMode`
   ConstrainedLanguage blocks COM objects, .NET types, and restricts scripting.

4. **Enumerate LAPS delegations** — find which groups can read LAPS passwords:
   `Find-LAPSDelegatedGroups`
   Shows OUs and the groups delegated read access (typically Domain Admins, LAPS Admins).

5. **Check for users with All Extended Rights** — these users can also read LAPS passwords:
   `Find-AdmPwdExtendedRights`
   Users who joined machines to the domain often have All Extended Rights over those hosts.

6. **Pull LAPS passwords** (if you have access) — retrieve cleartext local admin passwords and expiration dates:
   `Get-LAPSComputers`

## Key Takeaways
- **Defender is on by default** — tools like PowerView will be blocked unless you bypass or use alternatives; always check `RealTimeProtectionEnabled` before running offensive tooling.
- **AppLocker blocking `powershell.exe` is common but shallow** — `SysWOW64\WindowsPowerShell\v1.0\powershell.exe` or `powershell_ise.exe` are often overlooked.
- **Constrained Language Mode kills many offensive scripts** — if you're in it, you'll need to pivot to other execution methods or disable it (covered in later modules).
- **LAPS enumeration reveals attack paths** — a user with read access to LAPS passwords on a critical server (e.g., SQL01, DC01) is a high-value target; `Get-LAPSComputers` even outputs cleartext passwords if you have the rights.
- **Users with All Extended Rights** (e.g., the account that domain-joined a machine) may be less protected than those in explicitly delegated LAPS groups — worth hunting for.
- **Live off the land first** — use built-in PowerShell cmdlets for this enumeration before bringing external tools that may trigger Defender or AppLocker.

## Gotchas
- `Get-MpComputerStatus` is unobtrusive, but if Defender tamper protection is enabled, disabling it will generate alerts.
- AppLocker rules in `AuditOnly` mode won't block execution but will still log events (event ID 8003-8006 in AppLocker log) — you may be recorded without being stopped.
- PowerShell ISE and the 32-bit PowerShell host are frequently forgotten in AppLocker policies — try those paths before resorting to more aggressive bypasses.
- LAPS passwords are stored in AD (`ms-Mcs-AdmPwd` attribute) — reading them requires the specific extended right; `Get-LAPSComputers` will return empty if you lack permission.
- Constrained Language Mode can be enforced via `AppLocker` *or* via `Device Guard / WDAC` — the bypass method depends on how it was set.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[12-internal-password-spraying-windows]] | [[14-credentialed-enum-linux]] →
<!-- AUTO-LINKS-END -->
