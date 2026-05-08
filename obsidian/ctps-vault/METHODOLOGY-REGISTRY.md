# Methodology Registry

Tracks which CPTS module folders have an exam-ready `00-METHODOLOGY.md` playbook written.

**Reference template:** `file-uploads/00-METHODOLOGY.md` (the original).

## Status

| Module | Notes (.md count) | Methodology | ID | Date |
|---|---:|---|---:|---|
| file-uploads | (reference) | Done | 600 | seed |
| xss | 9 | Done | 700 | 2026-05-07 |
| documentation | 8 | Skip (reporting module — not an attack chain; doesn't fit decision-tree format) | — | — |
| sqlmap-fundamentals | 11 | Done | 702 | 2026-05-07 |
| file-inclusion | 12 | Done | 703 | 2026-05-07 |
| login-brute-forcing | 12 | Done | 704 | 2026-05-07 |
| pivoting-tunneling | 12 | Pending | — | — |
| common-services | 15 | Pending | — | — |
| sql-injection-fundamentals | 17 | Pending | — | — |
| web-proxies | 17 | Pending | — | — |
| ad-enum-attacks | 33 | Pending (large — read in parallel) | — | — |
| password-attacks | 1 | Skip (too thin until more notes added) | — | — |

## Next Up

Smallest unwritten attack-chain module → `pivoting-tunneling/` (12 notes), then `common-services/` (15 notes), then `sql-injection-fundamentals/` (17 notes).

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
| 705 | pivoting-tunneling (reserved) |
| 706 | common-services (reserved) |
| 707 | sql-injection-fundamentals (reserved) |
| 708 | web-proxies (reserved) |
| 709 | ad-enum-attacks (reserved) |

Update this table when a new methodology is written.
