# NOTE template — hands-on section (has commands)

---

## ID
533

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 20 — ACL Enumeration

## Description
Enumerate abusable ACL attack paths using PowerView and BloodHound, chaining ForceChangePassword → GenericWrite → GenericAll → DCSync through multiple users and groups.

## Tags
acl, enumeration, powerview, bloodhound, dacl, ace

## Commands
- Import-Module .\PowerView.ps1
- $sid = Convert-NameToSid <USER>
- Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
- Get-DomainObjectACL -ResolveGUIDs -Identity <TARGET> | ? {$_.SecurityIdentifier -eq $sid}
- Find-InterestingDomainAcl
- Get-DomainGroup -Identity "<GROUP>" | select memberof
- Get-ADUser -Filter * | Select-Object -ExpandProperty SamAccountName > ad_users.txt
- foreach($line in [System.IO.File]::ReadLines("ad_users.txt")) {get-acl "AD:\$(Get-ADUser $line)" | Select-Object Path -ExpandProperty Access | Where-Object {$_.IdentityReference -match 'INLANEFREIGHT\\<USER>'}}

## What This Section Covers
How to enumerate ACL-based attack paths using PowerView (`Get-DomainObjectACL -ResolveGUIDs`) and BloodHound, starting from a controlled user and chaining through multiple ACE relationships to reach DCSync privileges. Covers the critical difference between raw GUID output and human-readable `-ResolveGUIDs` output, plus a native PowerShell alternative using `Get-Acl` and `Get-ADUser`.

## Methodology
1. Get the SID of your controlled user: `$sid = Convert-NameToSid <USER>`
2. Search for all objects this user has rights over: `Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}`
3. Note the `ActiveDirectoryRights` and `ObjectAceType` for each result — these tell you what you can do.
4. For each new user/group you can compromise, repeat the search with their SID to chain further.
5. Check group nesting: `Get-DomainGroup -Identity "<GROUP>" | select memberof` — inherited rights from parent groups expand your attack surface.
6. Validate the full chain in BloodHound: set your controlled user as the starting node → Outbound Control Rights → Transitive Object Control.

## Multi-step Workflow
```
# 1. Get SID of controlled user
$sid = Convert-NameToSid wley

# 2. Find what wley has rights over
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}
# Result: ForceChangePassword over damundsen

# 3. Check what damundsen has rights over
$sid2 = Convert-NameToSid damundsen
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid2}
# Result: GenericWrite over Help Desk Level 1 group

# 4. Check group nesting
Get-DomainGroup -Identity "Help Desk Level 1" | select memberof
# Result: nested into Information Technology group

# 5. Check what Information Technology group has rights over
$itgroupsid = Convert-NameToSid "Information Technology"
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $itgroupsid}
# Result: GenericAll over adunn

# 6. Check what adunn has rights over
$adunnsid = Convert-NameToSid adunn
Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $adunnsid}
# Result: DS-Replication-Get-Changes + DS-Replication-Get-Changes-In-Filtered-Set = DCSync
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: What is the rights GUID for User-Force-Change-Password? | `00299570-246d-11d0-a768-00aa006e0529` | `Get-DomainObjectACL` output ObjectAceType field (without ResolveGUIDs) |
| Q2: What flag shows ObjectAceType in human-readable format? | `-ResolveGUIDs` | PowerView flag for Get-DomainObjectACL |
| Q3: What privileges does damundsen have over Help Desk Level 1? | `GenericWrite` | `Get-DomainObjectACL -ResolveGUIDs` with damundsen's SID |
| Q4: What ActiveDirectoryRights does forend have over dpayne? | `GenericAll` | `Get-DomainObjectACL -ResolveGUIDs -Identity dpayne \| ? {$_.SecurityIdentifier -eq $sid}` |
| Q5: ObjectAceType of first right forend has over GPO Management? | `Self-Membership` | `Get-DomainObjectACL -ResolveGUIDs -Identity "GPO Management" \| ? {$_.SecurityIdentifier -eq $sid}` |

## Key Takeaways
- `Find-InterestingDomainAcl` returns massive output — useless without filtering. Always start with a controlled user's SID and search targeted.
- Always use `-ResolveGUIDs` — without it, ObjectAceType shows raw GUIDs that require manual lookup.
- ACL enumeration is a chain: each compromised user/group opens new ACE edges. Repeat the SID-based search at every hop.
- Group nesting multiplies rights silently — a user in Group A inherits all rights of any group A is nested into.
- BloodHound's "Transitive Object Control" under Outbound Control Rights visualizes the full chain instantly — use PowerView to confirm and exploit, BloodHound to discover.
- The attack chain in this lab: `wley` → ForceChangePassword → `damundsen` → GenericWrite → Help Desk Level 1 → nested into Information Technology → GenericAll → `adunn` → DCSync rights.

## Gotchas
- `Get-DomainObjectACL -Identity *` can take 1-2+ minutes in large environments — be patient or scope to specific targets.
- If PowerView is already imported, running `Get-ADObject` for GUID reverse lookups may error — open a new PowerShell session.
- The native `Get-Acl` / `Get-ADUser` method works without PowerView but is much slower and still returns raw GUIDs.
- Self-Membership ACE (AddSelf) means the user can add themselves to the group — distinct from GenericWrite which lets you add anyone.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[19-acl-abuse-primer]] | [[21-acl-abuse-tactics]] →
<!-- AUTO-LINKS-END -->
