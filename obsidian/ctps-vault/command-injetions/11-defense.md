## ID
760

## Module
Command Injections

## Kind
notes

## Title
Section 11 — Command Injection Prevention

## Description
A comprehensive guide to preventing OS command injection vulnerabilities through secure coding practices, robust input validation and sanitisation, and proper web server configuration.

## Tags
defence, prevention, secure-coding, input-validation, hardening

## TL;DR — What's Important
- **Avoid shell functions entirely:** Use built‑in language features or libraries instead of `system()`, `exec()`, `child_process.exec()`, etc.
- **Validate input strictly:** Use allow‑list validation (e.g., `filter_var` in PHP, regex, or dedicated libraries) to reject anything that isn’t exactly what you expect.
- **Sanitise after validation:** Strip or escape any characters that are not required for the data’s purpose; never rely on blacklists alone.
- **Lock down the server:** Run the web server as a low‑privilege user, disable dangerous PHP functions, restrict file access with `open_basedir`, and deploy a WAF.
- **Assume bypass is possible:** No single defence is foolproof; combine multiple layers and still pentest your own application.

## Concept Overview
Command injection vulnerabilities arise when user‑controlled data is passed unsafely to OS command interpreters. Prevention centres on two complementary strategies: **code‑level** (avoid using shell‑executing functions, validate and sanitise all inputs) and **infrastructure‑level** (reduce the damage a successful injection can cause). This section provides concrete examples in PHP and NodeJS for validating IP addresses, stripping dangerous characters, and escaping remaining special characters. It also outlines server‑hardening measures such as `disable_functions`, `open_basedir`, and WAF deployment.

## Key Concepts

### 1. Replace System Commands with Built‑in Functions
- Instead of `system("ping -c 1 $ip")`, use PHP’s `fsockopen()` to check a host, or use language‑native libraries for tasks like DNS lookups, file operations, and network requests.
- If no built‑in alternative exists, isolate the command execution into a tightly controlled function that accepts only pre‑validated parameters.

### 2. Input Validation
- **Allow‑list approach:** Define exactly what the input should look like and reject everything else.
- **PHP example – IP validation:** `filter_var($_GET['ip'], FILTER_VALIDATE_IP)`
- **NodeJS example – custom regex:** `if(/^\d{1,3}\.\d{1,3}\.\d{1,3}\.\d{1,3}$/.test(ip)){ ... }`
- **Use libraries:** `is-ip` (Node), `validator` (Python), or framework built‑ins.
- Validation must happen **on the server side**, regardless of whatever the client does.

### 3. Input Sanitisation
- Performed **after** validation to remove or encode any special characters that shouldn’t be there.
- **PHP:** `preg_replace('/[^A-Za-z0-9.]/', '', $_GET['ip']);` – keeps only alphanumerics and dots.
- **NodeJS:** `ip.replace(/[^A-Za-z0-9.]/g, '');` or `DOMPurify.sanitize(ip)`.
- When special characters must be allowed (e.g., free‑text comments), use escaping functions like `escapeshellcmd()` (PHP) or `escape()` (Node), but be aware that escaping alone can often be bypassed; it is a last resort.

### 4. Server Configuration Hardening
| Measure | Purpose |
|---|---|
| Use a WAF (mod_security, Cloudflare, etc.) | Block obvious injection patterns at the network/HTTP level |
| Principle of Least Privilege | Run web server as `www-data` or similar unprivileged account |
| Disable dangerous functions | PHP: `disable_functions = system,exec,shell_exec,passthru,popen,proc_open` |
| Restrict file access | PHP: `open_basedir = /var/www/html` so injected commands can’t traverse the filesystem freely |
| Reject suspicious requests | Block double‑encoded characters, non‑ASCII URLs, and overly long parameters |
| Keep libraries updated | Avoid known‑vulnerable modules (e.g., old PHP‑CGI setups) |

## Why It Matters
Command injection is one of the most destructive web vulnerabilities because it grants direct access to the underlying operating system. Implementing the prevention measures in this section drastically reduces the attack surface. Even if a developer makes a mistake, a hardened server limits what an attacker can do. For penetration testers, understanding these defences is essential—you’ll need to recognise them during engagements and make accurate, actionable recommendations to clients.

## Defender Perspective
- **Detection:** WAF logs, unusual process creation (e.g., `sh -c`, `cmd.exe` spawned by the web server process), and integrity monitoring (files written to web root when they shouldn’t be) are key indicators.
- **Mitigations:** Everything in this section applies. Additionally, implement robust logging and alerting for any time the web server spawns a child process.
- **MITRE ATT&CK:** Prevention measures directly counter T1059 (Command and Scripting Interpreter) and T1203 (Exploitation for Client Execution) at the initial access and execution stages.

## Key Takeaways
- The best command injection defence is to **never let user data touch a shell**. Refactor code to use native functions.
- Validation *what* the input is, sanitisation *removes* what it shouldn’t contain. Do both, and do them server‑side.
- Even after all code‑level fixes, lock down the server so that a hypothetical lingering bug cannot lead to full compromise.
- Security is not one‑and‑done: combine secure coding, configuration hardening, and regular penetration testing adapted to the application’s size and complexity.

## Gotchas
- `escapeshellcmd()` in PHP escapes the entire command but can still allow argument injection if you concatenate strings after escaping; prefer `escapeshellarg()` when passing individual arguments.
- Input validation regex can be too permissive; always test with known injection payloads (`;id`, `|whoami`, backticks) to ensure they are truly rejected.
- `disable_functions` can be bypassed in some PHP environments through alternative methods like `LD_PRELOAD`; it should not be the sole line of defence.
- Blacklisting malicious characters (e.g., `;`, `&`, `|`) is fragile; attackers constantly discover new delimiters and encodings. Always default to an allow‑list (only characters you *know* are safe).