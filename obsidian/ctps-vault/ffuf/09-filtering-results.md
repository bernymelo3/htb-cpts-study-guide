# LAB — Filtering Results

## ID
9

## Module
Ffuf

## Kind
lab

## Title
Section 9 — Filtering Results

## Description
Use `-fs` (filter size), `-fw` (filter words), `-fl` (filter lines), or `-ac` (auto-calibrate) to remove the baseline-response noise that floods vhost / parameter scans.

## Tags
ffuf, filtering, vhost, lab, calibration

## Commands
- `ffuf -w wordlist:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -fs 900`
- `ffuf -w wordlist:FUZZ -u http://academy.htb:PORT/ -H 'Host: FUZZ.academy.htb' -ac`

## What This Section Covers
ffuf's default filter is the status code (drops 404). When every fuzz miss returns the same 200 OK (vhost fuzzing, parameter fuzzing on a static endpoint), you need a second filter axis. Response size is the most reliable, since the baseline page is always the same byte count. `-ac` auto-calibrates by sending a few junk requests, observing the baseline, and filtering it for you.

## Methodology
1. Run the scan once without filtering — every request returns 200, same size.
2. Read the `Size:` column; that's the baseline.
3. Re-run with `-fs <baseline-size>` to suppress noise.
4. **Or** run with `-ac` and let ffuf figure it out automatically.
5. Surviving hits are real targets — verify by visiting (after `/etc/hosts`).

## Matcher / Filter Reference
| Flag | Meaning |
|------|---------|
| `-mc` | Match status codes (default: `200,204,301,302,307,401,403`) |
| `-ml` | Match line count |
| `-mr` | Match regex on response body |
| `-ms` | Match response size |
| `-mw` | Match word count |
| `-fc` | **Filter** status codes |
| `-fl` | Filter line count |
| `-fr` | Filter regex |
| `-fs` | Filter response size |
| `-fw` | Filter word count |
| `-ac` | Auto-calibrate filters from junk-input baseline |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Other vhost(s) under `academy.htb` (besides `admin`) | **test** | `ffuf -s -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://academy.htb:STMPO/ -H 'Host: FUZZ.academy.htb' -fs 986` returned `admin` and `test`. The module section already revealed `admin`, so the answer is `test` |

## Key Takeaways
- **Filter the baseline by size.** Status code alone is useless when the server returns 200 to everything.
- `-ac` is the no-brain option — it calibrates per-target on the fly. Use it unless you have a specific size in mind.
- **Multi-axis filtering** (`-fs X -fw Y`) catches the rare case where a hit accidentally matches the baseline size but differs in word count.

## Gotchas
- If the baseline page is dynamic (timestamps, randomly-ordered nav), size filtering breaks. Switch to `-fl` (lines) or `-fw` (words), which are more stable.
- `-ac` re-calibrates every run — slight changes in the target page can shift the filter and miss results. Compare runs.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[08-vhost-fuzzing]] | [[10-parameter-fuzzing-get]] →
<!-- AUTO-LINKS-END -->
