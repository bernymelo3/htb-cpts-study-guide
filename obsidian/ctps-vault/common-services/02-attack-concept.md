## ID
102

## Module
Attacking Common Services

## Kind
notes

## Title
Section 2 — The Concept of Attacks

## Description
A reusable mental model (Source → Process → Privileges → Destination) that deconstructs any vulnerability – using Log4j as the canonical example – so you can systematically find and exploit flaws in any service.

## Tags
theory, methodology, vulnerability-analysis, log4j, concept

## TL;DR — What's Important
- **Four‑category pattern:** Every vulnerability involves a **Source** (where input comes from), a **Process** (what the program does with it), **Privileges** (what rights the process runs with), and a **Destination** (where the result goes).
- **Source is where you inject:** Code, libraries, configs, APIs, or user input. Manipulating any of these can trigger the flaw.
- **Privileges determine impact:** A bug in a low‑privilege process (e.g., user‑land service) yields less damage than the same bug in a SYSTEM/root process.
- **Destination is your payload’s landing zone:** Local (file write, registry change) or network (reverse shell, data exfiltration). The cycle often repeats – the Destination of one step becomes the Source of the next.
- **Log4j example:** Source = manipulated User‑Agent header → Process = JNDI lookup misinterpretation → Privileges = admin (logging context) → Destination = attacker’s LDAP server → then the cycle repeats for RCE.

## Concept Overview
The “Concept of Attacks” is a structured way to think about vulnerabilities across any service, protocol, or application. Instead of memorizing hundreds of specific exploits, you learn to break an attack into four universal categories: **Source** (where the attacker’s input enters), **Process** (how the software handles that input), **Privileges** (the security context of the process), and **Destination** (where the effect lands). This pattern applies equally to buffer overflows, SQL injection, Log4shell, and misconfigured SMB shares. The framework helps you reason about unknown services, debug failed exploits, and even spot vulnerabilities during code review.

## Key Concepts

### Definitions
- **Source** – The origin of information that the process consumes. Can be code (already executed), libraries (external functions), config files, API responses, or direct user input.
- **Process** – The running instance (PID) and its internal logic: input parsing, data processing, variable assignment, logging. Most vulnerabilities live in this stage.
- **Privileges** – The permissions assigned to the process (SYSTEM, root, user, group, policy, or application‑level rules). The same bug yields different impact depending on privileges.
- **Destination** – Where the result of the process is stored (local filesystem) or forwarded (network socket). Often the attacker’s target for remote access or data theft.

### Categories / Types of Sources
| Source Type | Example in Log4j | Example in SMB |
|-------------|------------------|----------------|
| Code | – (not applicable) | Hardcoded credentials in binary |
| Libraries | Log4j library itself | libsmbclient parsing routine |
| Config | log4j.properties | smb.conf share definition |
| API | JNDI lookup interface | Named pipe RPC call |
| User Input | HTTP User‑Agent header | SMB session setup username |

### The Attack Cycle
1. **Initial trigger** – Attacker sends malicious input (Source).
2. **Processing** – Vulnerable function mishandles it (Process).
3. **Execution** – The process has sufficient privileges (Privileges) to perform the attacker’s desired action.
4. **Effect** – Result reaches a Destination (e.g., remote server, file write, command execution).
5. **Recursion** – The Destination may become a new Source for a second stage (e.g., downloaded malicious Java class → executed in same privileged process).

## Why It Matters
When you encounter a new service or a proprietary protocol, there is no ready‑made exploit. Using this conceptual model, you can systematically ask:  
- *Where can I inject input?* (Source)  
- *What does the software do with that input?* (Process)  
- *What rights does that process have?* (Privileges)  
- *What can I affect if I succeed?* (Destination)  

This turns an unknown service into a structured checklist. For example, during a penetration test you find a custom UDP‑based logging service. You notice it logs the “client version” string. You manipulate that string (Source) to include a format specifier. The logging function (Process) interprets it, leading to a format string vulnerability. The service runs as LocalSystem (Privileges). The Destination allows you to write to `C:\Windows\Temp\` (local). You now have a file write primitive – the first step toward RCE.

## Defender Perspective
- **Detection:** This model also helps defenders. Audit Sources (do you accept user input in dangerous places?), review Process logic (is there unsafe deserialization?), check Privileges (does the process really need admin rights?), and monitor Destinations (are outbound connections to unexpected IPs logged?).
- **Mitigation:** Apply the principle of least privilege – even if a vulnerability exists, low‑privilege processes limit damage. Use input validation at the Source boundary. Implement allow‑listing for Destinations (e.g., egress firewalls).
- **MITRE ATT&CK:** This concept underpins many techniques: T1203 (Exploitation for Client Execution), T1059 (Command and Scripting Interpreter – when Destination is a shell), T1574 (Hijack Execution Flow – when Source is a library).

## Key Takeaways
- The same four categories explain both simple (anonymous SMB share → file read) and complex (Log4j RCE) attacks. The model scales.
- Privileges are often the most overlooked category. Two identical bugs – one in a user‑land service, one in a kernel driver – have wildly different risk. Always check `whoami` / `sc qc` before investing time in exploitation.
- The Destination step is where persistence and pivoting happen. A vulnerability that only writes a file locally (Destination = local) is less valuable than one that can reach out to a remote server (Destination = network).
- When an exploit fails, trace it through the cycle. Did your input not reach the vulnerable Process (Source issue)? Did the Process reject it due to validation (Process issue)? Did the command execute but with insufficient rights (Privileges issue)? Did the reverse shell never come back (Destination issue, e.g., firewall)?
- For source code review, search for `Process.Start`, `Runtime.exec`, `eval`, `deserialize` – then trace back to where their arguments come from (Source) and what permissions the calling thread has (Privileges).

## Gotchas
- The Destination is **not always** the attacker’s server. In blind SQL injection, the Destination is the database server’s response page. In file inclusion, the Destination is the local filesystem.
- Privileges can be context‑dependent. A process running as SYSTEM may still be constrained by namespace isolation (Docker, AppContainer) or by system policies (Windows User Rights Assignments).
- The cycle can loop many times. Log4j’s JNDI lookup to an attacker’s LDAP server is a Destination → Source transition. Always consider second‑order effects.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|common-services]]  
← [[01-interacting-with-services]] | [[03-service-misconfigurations]] →
<!-- AUTO-LINKS-END -->
