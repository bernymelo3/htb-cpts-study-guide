# Methodology Registry

Tracks which CPTS module folders have an exam-ready `00-METHODOLOGY.md` playbook written.

**Reference template:** `ad-enum-attacks/00-METHODOLOGY.md` (current gold standard — has all four retrieval surfaces: Signal→Counter-Move, Decision Tree, Master Cheatsheet, Top Gotchas). `file-uploads/00-METHODOLOGY.md` is the legacy seed.

## Status

| Module                     | Notes (.md count) | Methodology                                                                     |  ID | Date       |
| -------------------------- | ----------------: | ------------------------------------------------------------------------------- | --: | ---------- |
| file-uploads               |       (reference) | Done                                                                            | 600 | seed       |
| xss                        |                 9 | Done                                                                            | 700 | 2026-05-07 |
| documentation              |                 8 | Skip (reporting module — not an attack chain; doesn't fit decision-tree format) |   — | —          |
| sqlmap-fundamentals        |                11 | Done                                                                            | 702 | 2026-05-07 |
| file-inclusion             |                12 | Done                                                                            | 703 | 2026-05-07 |
| login-brute-forcing        |                12 | Done                                                                            | 704 | 2026-05-07 |
| pivoting-tunneling         |                12 | Done                                                                            | 705 | 2026-05-13 |
| common-services            |                15 | Done                                                                            | 706 | 2026-05-15 |
| sql-injection-fundamentals |                17 | Done                                                                            | 707 | 2026-05-15 |
| ad-enum-attacks            |                33 | Done                                                                            | 709 | 2026-05-12 |
| attacking-common-applications |                33 | Done                                                                            | 710 | 2026-05-15 |
| getting-started            |                 7 | Done (foundational pentest methodology + 6 topic notes)                        | 711 | 2026-05-16 |
| windows-privesc            |                32 | Done                                                                            | 712 | 2026-05-16 |
| linux-privallege-escalation |               28 | Done                                                                            | 713 | 2026-05-16 |
| web-proxies                |                17 | Pending                                                                         |   — | —          |
| password-attacks           |                 1 | Skip (too thin until more notes added)                                          |   — | —          |

## Next Up

Tier 2 high-value audits → retrofit `file-inclusion`, `sqlmap-fundamentals`, `login-brute-forcing`, `xss` for Signal→Counter-Move + Decision Tree if missing. Tier 3 lowest priority: `web-proxies/`.

> `documentation/` skipped — reporting module, not an attack chain. If a reporting playbook is wanted later, write it as a separate "report-writing checklist" doc, not in the methodology decision-tree format.

## ID Allocation (methodology files only)

To keep methodology IDs from clashing with section-note IDs, allocate from a dedicated 700+ block:

| ID | Module |
|---|---|
| 600 | file-uploads (legacy — overlaps with stored-xss section ID) |
| 700 | xss |
| 701 | documentation (reserved) |
| 702 | sqlmap-fundamentals |
| 703 | file-inclusion |
| 704 | login-brute-forcing |
| 705 | pivoting-tunneling |
| 706 | common-services |
| 707 | sql-injection-fundamentals |
| 708 | web-proxies (reserved) |
| 709 | ad-enum-attacks |
| 710 | attacking-common-applications |
| 711 | getting-started |
| 712 | windows-privesc |
| 713 | linux-privallege-escalation |

Update this table when a new methodology is written.
