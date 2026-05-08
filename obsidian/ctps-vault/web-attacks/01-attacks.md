## ID
800

## Module
Web Attacks

## Kind
notes

## Title
Section 1 — Introduction to Web Attacks

## Description
Foundational overview of three critical web attack vectors—HTTP Verb Tampering, IDOR, and XXE Injection—and the impact they can have on modern web applications.

## Tags
theory, concept, web-attacks, http-verb-tampering, idor, xxe

## TL;DR — What's Important
- **HTTP Verb Tampering** exploits web servers that accept many HTTP methods, potentially bypassing authentication or security controls by sending unexpected verbs.
- **IDOR** is the lack of proper access control: by guessing or manipulating object references (IDs), attackers can access other users’ data or resources.
- **XXE Injection** targets applications that parse XML; outdated XML libraries can be tricked into reading local files, stealing credentials, or achieving remote code execution.
- **All three attacks are common** and can lead directly to data exposure, privilege escalation, or server compromise.
- **Defence requires layered controls** — strict method handling, robust access control checks, and secure XML parsers with external entities disabled.

## Concept Overview
Web applications have become the primary attack surface for organisations. This module focuses on three specific, high-impact attack types: HTTP Verb Tampering, Insecure Direct Object References (IDOR), and XML External Entity (XXE) Injection. Each of these arises from common misconfigurations or coding weaknesses—overly permissive HTTP methods, missing access control checks, and unsafe XML processing. The introduction sets the stage by defining each attack, explaining why they matter, and highlighting the potential business impact of a successful exploit.

## Key Concepts

### HTTP Verb Tampering
- Attackers send HTTP requests using methods other than the intended ones (`GET`, `POST`) — e.g., `PUT`, `DELETE`, `OPTIONS`, or even custom verbs.
- Web servers or frameworks may not restrict which methods are accepted for a given endpoint. An untrusted method might bypass authentication or security filters that only check, for example, `GET` requests.
- Common outcomes: bypassing login forms, accessing restricted areas, or triggering unintended actions.

### Insecure Direct Object References (IDOR)
- Occurs when an application exposes internal object identifiers (like user IDs, file names, or numeric IDs) without verifying that the requesting user has permission to access them.
- By simply changing a parameter (e.g., `?user_id=2` to `?user_id=3`), an attacker can view or modify data belonging to other users.
- IDOR is consistently ranked among the top web vulnerabilities because it stems from missing access control checks, not from complex technical bugs.

### XML External Entity (XXE) Injection
- Applications that accept and parse XML input may be vulnerable if they process Document Type Definitions (DTD) containing external entities.
- An attacker can craft an XML payload that forces the server to read local files (e.g., `/etc/passwd`, configuration files), perform Server‑Side Request Forgery (SSRF), or even execute code in certain cases.
- The vulnerability often resides in outdated XML libraries or poorly configured parsers that do not disable external entity processing.

## Why It Matters
These three attacks cover a broad spectrum of web application weaknesses: configuration flaws (HTTP Verb Tampering), broken access control (IDOR), and unsafe input parsing (XXE). Mastering them equips a penetration tester to uncover high‑severity findings that can directly lead to compromised user accounts, exposed sensitive data, or full server takeover. For defenders, understanding these attacks is essential to build proper mitigation strategies and to prioritise security efforts where they matter most.

## Defender Perspective
- **HTTP Verb Tampering** — Restrict HTTP methods on web servers and in application middleware to only those required per endpoint (e.g., allow only `GET` and `POST`). Reject unsupported verbs with a `405 Method Not Allowed` response.
- **IDOR** — Never rely on client‑provided object IDs for authorisation. Always verify that the authenticated user has access to the requested object on the server side, using a stable permission model (e.g., ACLs, ownership checks).
- **XXE** — Disable external entity processing and DTD parsing entirely in XML parsers. Use safer, non‑XML formats like JSON where possible. Keep XML libraries updated and configure them securely according to vendor guidelines.
- **Detection** — Monitor for anomalous HTTP verbs in access logs, inspect object ID parameter manipulation patterns, and analyse XML payloads for `<!ENTITY>` directives or outbound connections to unexpected external hosts.

## Key Takeaways
- HTTP Verb Tampering demonstrates that security middleware is only as strong as its coverage—every HTTP method must be explicitly restricted.
- IDOR is not a single exploit technique but a symptom of missing access control logic that must be addressed throughout the application.
- XXE is a classic input‑handling vulnerability; the fix is simple (disable entity expansion) but the impact can be devastating if missed.
- All three attack types can be discovered with a combination of manual probing, automated scanning, and careful log review.
- Understanding these attacks from both the offensive and defensive angles makes you a more effective tester and advisor.

## Gotchas
- Some web servers treat `HEAD` or `OPTIONS` methods as harmless, but they can leak information about allowed methods and aid in reconnaissance.
- IDOR can be hidden behind custom encoding (e.g., hashed IDs) — always test if the hashing scheme can be reversed or if sequential IDs still work.
- Even if an XML parser disables external entities, XML bombs (billion laughs attack) may still cause denial of service if not mitigated separately.