## ID
702

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 25 — IIS Tilde Enumeration

## Description
Exploits IIS short file name (8.3 format) disclosure via tilde enumeration to discover hidden files and directories, then brute-forces the full filenames with Gobuster.

## Tags
iis, tilde, 8.3-shortname, gobuster, enumeration, windows

## Commands
- nmap -p- -sV -sC --open <TARGET_IP>
- java -jar iis_shortname_scanner.jar 0 5 http://<TARGET_IP>/
- egrep -r ^transf /usr/share/wordlists/* | sed 's/^[^:]*://' > /tmp/list.txt
- gobuster dir -u http://<TARGET_IP>/ -w /tmp/list.txt -x .aspx,.asp

## What This Section Covers
When files or folders are created on IIS, Windows auto-generates 8.3 short file names (8 chars + dot + 3 char extension). The tilde (`~`) character followed by a sequence number represents these short names in URLs. By sending crafted HTTP requests with different character combinations after the tilde, an attacker can incrementally discover hidden files and directories, then brute-force the full names.

## Methodology
1. Scan the target: `nmap -p- -sV -sC --open <TARGET_IP>` — confirm IIS is running (tilde enum works on certain IIS versions like 7.5)
2. Run the IIS Short Name Scanner: `java -jar iis_shortname_scanner.jar 0 5 http://<TARGET_IP>/` — decline proxy when prompted
3. The tool sends requests with the tilde + character combinations (e.g., `/~a`, `/~se`, `/~sec`) and uses HTTP response codes to incrementally discover short names
4. Review discovered short names (e.g., `TRANSF~1.ASP`, `ASPNET~1`, `UPLOAD~1`) — these are truncated to 6 chars + `~1`
5. Generate a targeted wordlist from the known prefix: `egrep -r ^transf /usr/share/wordlists/* | sed 's/^[^:]*://' > /tmp/list.txt`
6. Brute-force the full filename: `gobuster dir -u http://<TARGET_IP>/ -w /tmp/list.txt -x .aspx,.asp`

## Key Takeaways
- The 8.3 format means: first 6 chars of the name + `~N` (sequence number) + `.` + first 3 chars of extension
- The sequence number (`~1`, `~2`) differentiates files with similar names in the same directory (e.g., `somefile.txt` → `somefi~1.txt`, `somefile1.txt` → `somefi~2.txt`)
- The scanner uses the OPTIONS HTTP method with a magic suffix (`/~1/`) to probe — a 200 response confirms a valid short name prefix
- Short names only reveal the first 6 characters — you still need to brute-force the full name with a tool like Gobuster
- The `egrep -r ^transf | sed 's/^[^:]*://'` pipeline extracts all words starting with the known prefix from system wordlists, stripping the filename:line prefix that `egrep -r` prepends
- Gobuster's `-x .aspx,.asp` flag appends those extensions to every wordlist entry during directory brute-forcing

## Gotchas
- The scanner requires Oracle Java — if `java -version` fails, install JDK first
- When the scanner prompts for a proxy, just press Enter to decline
- The target may not allow direct GET access to the short name URL (e.g., `http://target/TRANSF~1.ASP` returns forbidden), which is why full-name brute-forcing is necessary
- Not all IIS versions are vulnerable — this works reliably on older versions like IIS 7.5

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What is the full .aspx filename that Gobuster identified? | transfer.aspx | Gobuster brute-force using wordlist filtered by `transf` prefix with `-x .aspx,.asp` |
