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
| password-attacks           |                26 | Done                                                                            | 714 | 2026-05-16 |
| web-proxies                |                17 | Done                                                                            | 708 | 2026-05-16 |
| web-attacks                |                17 | Done                                                                            | 722 | 2026-05-16 |
| web-recon                  |                20 | Done                                                                            | 723 | 2026-05-16 |
| footprinting               |    21 (has 02-enum) | Done (retrofit → standard 00; 02-enum kept as theory note)                     | 718 | 2026-05-16 |
| nmap                       |                13 | Done                                                                            | 719 | 2026-05-16 |
| ffuf                       |                14 | Done                                                                            | 716 | 2026-05-16 |
| command-injetions          |                12 | Done                                                                            | 715 | 2026-05-16 |
| file-transfers             |                11 | Done                                                                            | 717 | 2026-05-16 |
| shells-payloads            |                18 | Done                                                                            | 720 | 2026-05-16 |
| using-the-metasploit       |                16 | Done                                                                            | 721 | 2026-05-16 |
| vulnerability-assessment   |                 0 | Skip (no notes; process/theory module — not an attack chain)                   |   — | —          |
| attacking-enterprise-networks |              0 | Skip (no notes; capstone — covered by individual module methodologies)         |   — | —          |
| penetration-testing-process |                0 | Skip (process theory — covered by getting-started methodology)                 |   — | —          |
| report                     |                 0 | Skip (reporting — same rationale as documentation; PDF only)                   |   — | —          |

## Next Up

Tier 2 high-value audits → retrofit `file-inclusion`, `sqlmap-fundamentals`, `login-brute-forcing`, `xss` for Signal→Counter-Move + Decision Tree if missing.

New tracked Pending modules (suggested order by exam value): ~~`web-attacks`~~ (done 722) → ~~`command-injetions`~~ (done 715) → ~~`web-recon`~~ (done 723) → ~~`shells-payloads`~~ (done 720) → ~~`file-transfers`~~ (done 717) → ~~`footprinting`~~ (done 718, retrofit) → ~~`ffuf`~~ (done 716) → ~~`nmap`~~ (done 719) → ~~`using-the-metasploit`~~ (done 721) → ~~`web-proxies`~~ (done 708). **All tracked modules complete** — only Skip-classified folders remain.

> `documentation/` skipped — reporting module, not an attack chain. If a reporting playbook is wanted later, write it as a separate "report-writing checklist" doc, not in the methodology decision-tree format.
> `vulnerability-assessment/`, `attacking-enterprise-networks/`, `penetration-testing-process/`, `report/` skipped — empty folders / process-theory or capstone material, not standalone attack chains.

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
| 708 | web-proxies |
| 709 | ad-enum-attacks |
| 710 | attacking-common-applications |
| 711 | getting-started |
| 712 | windows-privesc |
| 713 | linux-privallege-escalation |
| 714 | password-attacks |
| 715 | command-injetions |
| 716 | ffuf |
| 717 | file-transfers |
| 718 | footprinting |
| 719 | nmap |
| 720 | shells-payloads |
| 721 | using-the-metasploit |
| 722 | web-attacks |
| 723 | web-recon |

Update this table when a new methodology is written.
