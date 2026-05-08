# NOTE — Section 27: Domain Trusts Primer

## ID
2700

## Module
Active Directory Enumeration & Attacks

## Kind
notes

## Title
Section 27 — Domain Trusts Primer

## Description
Covers what domain/forest trusts are, the different trust types and directions, and how to enumerate trust relationships using built-in tools, PowerView, netdom, and BloodHound.

## Tags
active-directory, domain-trusts, enumeration, powerview, bloodhound, netdom

## Commands
- `Import-Module activedirectory; Get-ADTrust -Filter *`
- `Get-DomainTrust`
- `Get-DomainTrustMapping`
- `Get-DomainUser -Domain <CHILD_DOMAIN> | select SamAccountName`
- `netdom query /domain:<DOMAIN> trust`
- `netdom query /domain:<DOMAIN> dc`
- `netdom query /domain:<DOMAIN> workstation`

## What This Section Covers
Domain trusts create authentication links between domains or forests, letting users access resources across boundaries. Attackers enumerate these relationships to find lateral movement paths — a weak child domain or a trusted partner can be an indirect route into a hardened principal domain.

## Methodology
1. Enumerate trusts with `Get-ADTrust -Filter *` (built-in AD module) — note the `IntraForest`, `ForestTransitive`, and `Direction` properties.
2. Use PowerView's `Get-DomainTrust` for cleaner output showing `TrustAttributes` (WITHIN_FOREST vs FOREST_TRANSITIVE) and `TrustDirection`.
3. Map the full trust graph with `Get-DomainTrustMapping` — this queries from both sides, showing the reverse trust entries too.
4. Use `netdom query /domain:<DOMAIN> trust` as a lightweight alternative (no module import needed).
5. Enumerate users across trusts with `Get-DomainUser -Domain <CHILD_DOMAIN>` to look for privileged accounts.
6. Visualize in BloodHound using the **Map Domain Trusts** pre-built query.

## Trust Types Reference

| Type         | Description                                          | Transitivity                       |
| ------------ | ---------------------------------------------------- | ---------------------------------- |
| Parent-child | Two+ domains in same forest; child ↔ parent          | Two-way transitive                 |
| Cross-link   | Between child domains (speed up auth)                | Transitive                         |
| External     | Between domains in separate forests, no forest trust | Non-transitive, uses SID filtering |
| Tree-root    | Forest root ↔ new tree root                          | Two-way transitive                 |
| Forest       | Between two forest root domains                      | Transitive                         |
| ESAE         | Bastion forest for AD management                     | —                                  |

## Trust Direction

| Direction | Meaning |
|-----------|---------|
| One-way | Trusted domain users → trusting domain resources (not reverse) |
| Bidirectional | Users from both domains can access resources in either direction |

## Transitive vs Non-Transitive

- **Transitive:** If A trusts B and B trusts C, then A trusts C. Applies to forest, tree-root, parent-child, and cross-link trusts.
- **Non-transitive:** Only the direct trust partner is trusted. Typical for external or custom trusts.

## Lab — Questions & Answers
| Q                                        | Answer                        | Found In / Method                               |
| ---------------------------------------- | ----------------------------- | ----------------------------------------------- |
| Q1: Child domain of INLANEFREIGHT.LOCAL? | LOGISTICS.INLANEFREIGHT.LOCAL | Get-ADTrust output — `IntraForest: True`        |
| Q2: Forest transitive trust domain?      | FREIGHTLOGISTICS.LOCAL        | Get-ADTrust output — `ForestTransitive: True`   |
| Q3: Direction of that trust?             | BiDirectional                 | Get-ADTrust output — `Direction: BiDirectional` |

## Key Takeaways
- Always check with the client that enumerated trust domains are **in scope** before attacking across them.
- Bidirectional trusts are the most interesting — they let you authenticate and enumerate in both directions.
- M&A scenarios often create poorly reviewed trusts; a softer acquired domain can be an indirect route into the hardened principal domain.
- `IntraForest: True` = child domain (same forest). `ForestTransitive: True` = forest/external trust (different forest).
- Kerberoasting across trusts is a real attack path — a service account in a trusted domain may have admin rights in the principal domain.

## Gotchas
- If you can't authenticate across a trust, you can't enumerate or attack it — always verify authentication works first.
- `netdom query` may show "Not found" for trust type even when the trust exists — it's a display quirk, not an error.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[25-bleeding-edge-vulnerabilities]] | [[29-domain-trusts-child-to-parent-linux]] →
<!-- AUTO-LINKS-END -->
