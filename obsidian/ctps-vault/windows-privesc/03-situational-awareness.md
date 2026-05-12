# NOTE template — hands-on section (has commands)

## ID
601

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 3 — Situational Awareness

## Description
Covers initial orientation on a compromised Windows host: network enumeration (interfaces, ARP, routing), and identifying security protections (Windows Defender, AppLocker) that shape your next moves.

## Tags
situational-awareness, network-enumeration, applocker, windows-defender, arp, dual-homed

## Commands
- ipconfig /all
- arp -a
- route print
- Get-MpComputerStatus
- Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections
- Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path <EXE_PATH> -User Everyone
- xfreerdp /v:<TARGET_IP> /u:<USER> /p:<PASS> /dynamic-resolution

## What This Section Covers
After landing on a Windows host, the first priority is understanding where you are: what networks the host touches, what other hosts it has talked to, and what security controls are in place. This "situational awareness" phase determines which tools you can use, whether lateral movement paths exist (dual-homed hosts), and whether you need to bypass AppLocker or evade Defender before proceeding with privilege escalation.

## Methodology
1. Run `ipconfig /all` to identify all network interfaces, IP addresses, DNS servers, and domain membership — look for dual-homed hosts (multiple NICs on different subnets).
2. Run `arp -a` to view the ARP cache per interface — recently communicated hosts may be lateral movement targets.
3. Run `route print` to review the routing table — understand what networks are reachable and which gateway is preferred (lower metric = preferred).
4. Check Windows Defender status with `Get-MpComputerStatus` — look at `RealTimeProtectionEnabled`, `AntivirusEnabled`, and `BehaviorMonitorEnabled` to know what's active.
5. Enumerate AppLocker rules with `Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections` — look for `Action: Deny` entries to find blocked executables.
6. Test specific binaries against AppLocker with `Get-AppLockerPolicy -Local | Test-AppLockerPolicy -path C:\Windows\System32\cmd.exe -User Everyone`.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: What is the IP address of the other NIC attached to the target host? | 172.16.20.45 | RDP in → `ipconfig /all` → Ethernet1 (vmxnet3) adapter → IPv4 Address field |
| Q2: What executable other than cmd.exe is blocked by AppLocker? | powershell_ise.exe | `Get-AppLockerPolicy -Effective | select -ExpandProperty RuleCollections` → find rule with `Action: Deny` and `Name: Block PowerShell ISE` |

## Key Takeaways
- Always check for dual-homed hosts — a second NIC on a different subnet is a lateral movement opportunity that may not be obvious from initial external scanning.
- ARP cache (`arp -a`) reveals recent communications — admin boxes connecting via RDP/WinRM are high-value pivot targets once you obtain credentials.
- `RealTimeProtectionEnabled: False` in Defender status means you can run tools more freely, but don't assume — check `BehaviorMonitorEnabled` and `AntivirusEnabled` separately.
- AppLocker default rules allow everything in `%PROGRAMFILES%\*` and `%WINDIR%\*` for Everyone, but only admins get the wildcard `*` (all files) rule — non-admin users are restricted.
- AppLocker `Deny` rules override `Allow` rules, so even if cmd.exe is in `%WINDIR%\*`, an explicit deny still blocks it.
- Routing table metrics matter: lower metric = preferred route. A persistent route (`route print` → Persistent Routes) survives reboots and tells you about intentional network design.

## Gotchas
- `Get-MpComputerStatus` requires PowerShell — if AppLocker blocks PowerShell, you'll need to find an alternative or bypass.
- AppLocker rules apply per-user/group (check `UserOrGroupSid`) — a rule blocking Everyone (`S-1-1-0`) is different from one blocking non-admins. Admins (`S-1-5-32-544`) typically have "All files" allow rules.
- `Test-AppLockerPolicy` tests against the **local** policy, not necessarily the effective (domain-pushed) policy — use `-Effective` flag on `Get-AppLockerPolicy` to see what's actually enforced.
