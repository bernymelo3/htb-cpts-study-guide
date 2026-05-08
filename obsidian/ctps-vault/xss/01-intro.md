## ID
400

## Module
Cross-Site Scripting (XSS)

## Kind
notes

## Title
Section 1 — Introduction

## Description
Introduces Cross‑Site Scripting (XSS), its impact, real‑world examples, and the three main types: Stored, Reflected, and DOM‑based.

## Tags
theory, xss, introduction, stored-xss, reflected-xss, dom-xss

## TL;DR — What's Important
- **XSS injects JavaScript into a page through unsanitised user input.** The code runs in other users’ browsers, not on the server.
- **Impact is client‑side but diverse:** session theft, defacement, forced actions, keylogging, and even browser exploitation.
- **Three primary types exist:**  
  - **Stored (Persistent)** – payload saved in the database and displayed to all visitors.  
  - **Reflected (Non‑Persistent)** – payload echoed immediately in a response (e.g., search results).  
  - **DOM‑based** – payload processed entirely by client‑side JavaScript without server involvement.
- **XSS is extremely common** – even major platforms (MySpace, Twitter, Google, Apache) have suffered from it.
- **Risk = High probability × Low‑to‑Medium impact** – consistently rated as a top web vulnerability by OWASP.

## Concept Overview
Cross‑Site Scripting (XSS) occurs when a web application includes untrusted data in a page without proper validation or escaping. An attacker can inject arbitrary JavaScript that executes in the browser of anyone viewing the page. Although the server itself is not directly compromised, the client‑side execution can lead to account takeover, data theft, or malware distribution. XSS is categorised into Stored, Reflected, and DOM‑based, depending on where the payload lives and how it reaches the victim.

## Key Concepts
- **Stored (Persistent) XSS** – User input is saved (e.g., in a database) and then rendered on the page for all users. Every visitor triggers the payload. This is the most critical type.
- **Reflected (Non‑Persistent) XSS** – The payload is part of a request (URL parameter, form input) and immediately reflected in the response without being stored. The attacker must trick the victim into clicking a crafted link.
- **DOM‑based XSS** – The entire attack happens in the browser. Malicious input modifies the DOM (Document Object Model) through client‑side scripts, never touching the server.
- **Impact** – Stealing cookies/sessions, performing actions on behalf of the user, defacing pages, redirecting to malicious sites, cryptojacking, and even exploiting browser vulnerabilities.

## Why It Matters
XSS is consistently ranked among the OWASP Top 10 web application security risks. It is universally applicable—any application that reflects user input without sanitisation is a candidate. Understanding the three types is essential for both finding and fixing them. Penetration testers must be able to identify each type and craft effective payloads.

## Defender Perspective
- **Detection:** WAF signatures look for common XSS vectors (e.g., `<script>`, `javascript:`). Log analysis can reveal unusual request patterns containing HTML/JS fragments.
- **Mitigations:**  
  - **Output encoding:** escape `<`, `>`, `"`, `'`, `&` depending on context (HTML body, attribute, JavaScript, CSS).  
  - **Input validation:** allow‑list acceptable characters where possible.  
  - **Content Security Policy (CSP):** restrict script sources to trusted origins.  
  - **HttpOnly/Secure cookies:** prevent JavaScript access to session tokens.
- **MITRE ATT&CK:** *T1189 – Drive‑by Compromise* (if user visits crafted URL), *T1204 – User Execution* (clicking a link), *T1539 – Steal Web Session Cookie*.

## Key Takeaways
- XSS is a **client‑side** vulnerability, but the root cause is usually insufficient server‑side input validation/output encoding.
- The three types require **different detection and exploitation approaches**: stored affects everyone, reflected needs a link, DOM‑based requires analysis of client‑side code.
- Real‑world impact goes far beyond popping `alert(1)`; automated exploitation can spread worms (Samy) or steal credentials at scale.
- Even a single XSS on a high‑traffic page can compromise an entire user base because the payload executes in the context of the vulnerable site.
- Modern frameworks often auto‑escape templates, but custom code, misconfiguration, and rich‑text handlers still introduce XSS.

## Gotchas
- Testing only with `<script>alert(1)</script>` can miss many XSS variants; context‑specific payloads (event handlers, `javascript:` URLs, CSS injection) are often needed.
- DOM‑based XSS can be invisible to server‑side logs and WAFs because the payload never reaches the server—only client‑side code review or dynamic analysis will find it.
- XSS and CSRF are often combined: XSS can bypass CSRF protections by reading the anti‑CSRF token and sending an authenticated request.
- Browser security features (e.g., XSS Auditor, CSP) can block some attacks, but they are not substitutes for proper sanitisation.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|xss]]  
[[02-stored-xss]] →
<!-- AUTO-LINKS-END -->
