# LAB — Directory Fuzzing

## ID
3

## Module
Ffuf

## Kind
lab

## Title
Section 3 — Directory Fuzzing

## Description
Run ffuf with a wordlist + the FUZZ keyword in the URL path to enumerate top-level directories on a target webserver.

## Tags
ffuf, directory-fuzzing, seclists, lab, web-content

## Commands
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://SERVER_IP:PORT/FUZZ`
- `ffuf -s -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u 'http://STMIP:STMPO/FUZZ'`
- `ffuf -h` (full options reference)

## What This Section Covers
First real ffuf run: `-w <wordlist>:FUZZ` defines the keyword, `-u <url-with-FUZZ>` places it. Default thread count is 40 — enough to scan ~87 k entries in <10 s without DoS-ing the target. `-t 200` exists but is rude on remote targets.

## Methodology
1. Pick a wordlist and bind the `FUZZ` keyword: `-w /path/to/list:FUZZ`.
2. Place `FUZZ` where the directory name goes: `-u http://target/FUZZ`.
3. Run the scan; ffuf filters the default-allowed status codes (`200,204,301,302,307,401,403`).
4. Inspect each hit. A `301` redirect is a real directory; visit it in the browser or follow up with page fuzzing.
5. Use `-s` for silent/clean output when piping to other tools.

## Key Flags
| Flag | Meaning |
|------|---------|
| `-w <list>:KEY` | Wordlist + keyword binding |
| `-u <url>` | Target URL (with KEY placeholder) |
| `-t <n>` | Thread count (default 40) |
| `-s` | Silent mode — only print hits |
| `-ic` | Ignore wordlist comment lines |
| `-mc` | Match status codes (default `200,204,301,302,307,401,403`) |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Other directory besides `/blog` | **forum** | `ffuf -s -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u 'http://STMIP:STMPO/FUZZ'` returned both `forum` and `blog` |

## Key Takeaways
- 40 threads is the sweet spot — increasing rarely helps and risks tripping rate limits or breaking targets.
- A directory returning `Status: 301` is real — the redirect is to the directory's trailing-slash form (`/blog` → `/blog/`).
- An empty 200 page on a found directory does not mean nothing's there — drill in with page fuzzing next.

## Gotchas
- Don't assume `Size: 0` means broken. The page exists; it just has no body content. Try fuzzing files inside it.
- The first lines of `directory-list-2.3-small.txt` are copyright comments — ffuf will treat them as candidates unless you pass `-ic`.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[02-web-fuzzing]] | [[04-page-fuzzing]] →
<!-- AUTO-LINKS-END -->
