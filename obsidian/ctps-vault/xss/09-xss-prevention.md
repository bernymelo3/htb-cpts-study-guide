## ID
402

## Module
Cross-Site Scripting (XSS)

## Kind
notes

## Title
Section 9 — XSS Prevention

## Description
Describes defense techniques against XSS on both front‑end and back‑end, including input validation, sanitization, output encoding, server configuration, and secure coding practices.

## Tags
xss, prevention, defense, sanitization, input-validation, output-encoding

## TL;DR — What's Important
- **Prevention must happen on both front‑end and back‑end:** Front‑end validation alone is trivially bypassed; the back‑end must enforce security.
- **Validate input against strict patterns:** Use allow‑lists (e.g., email regex) and reject anything that doesn’t match.
- **Sanitize input by escaping or purifying:** Libraries like DOMPurify (JS) or functions like `addslashes()` (PHP) neutralize dangerous characters.
- **Encode output contextually:** Convert `<` → `&lt;` and similar using `htmlentities()` (PHP) or `html-entities` (NodeJS) before displaying user data.
- **Never put user input directly into dangerous contexts:** Avoid `<script>`, `<style>`, HTML comments, attribute fields, or innerHTML assignments.
- **Leverage server‑side protections:** HTTPS, `X-Content-Type-Options: nosniff`, `Content-Security-Policy` (e.g. `script-src 'self'`), `HttpOnly` and `Secure` cookies, and a WAF.
- **Defense‑in‑depth is key:** A single layer is never enough; combine all measures.

## Concept Overview
XSS prevention focuses on breaking the injection chain: ensuring untrusted data cannot be interpreted as code. This requires validaton, sanitization, and encoding at every point where data enters (source) and is rendered (sink). Front‑end defenses provide immediate user feedback, but the back‑end is the ultimate gatekeeper. Proper server configuration and content security policies add additional layers that limit the impact even if a vulnerability slips through.

## Key Concepts

### Front‑end Defense
- **Input Validation** – Use JavaScript regex (e.g., email pattern) to block malformed data before submission.  
  *Note:* This is a convenience layer, not a security barrier—it can be bypassed with direct requests.
- **Input Sanitization** – Use libraries like **DOMPurify** (`DOMPurify.sanitize(dirty)`) to strip or escape dangerous characters.
- **Avoid Dangerous Sinks** – Never insert user data into:
  - `<script>`, `<style>` tags
  - Tag attribute fields without quoting/encoding
  - HTML comments `<!-- -->`
  - DOM‑manipulating functions: `innerHTML`, `outerHTML`, `document.write()`, `document.writeln()`, `document.domain`
  - jQuery methods: `html()`, `parseHTML()`, `add()`, `append()`, `prepend()`, `after()`, `insertAfter()`, `before()`, `insertBefore()`, `replaceAll()`, `replaceWith()`

### Back‑end Defense
- **Input Validation** – Validate on the server side using language‑specific filters (e.g., PHP `filter_var($_GET['email'], FILTER_VALIDATE_EMAIL)`) and reject invalid input without displaying it.
- **Input Sanitization** – Escape special characters before storage or further processing:
  - PHP: `addslashes()` (note: use context‑appropriate escaping; for HTML output, combine with output encoding).
  - NodeJS: `DOMPurify.sanitize()` or equivalent libraries.
- **Output Encoding** – Before displaying any user input, convert special characters into their HTML entities so the browser renders them as text, not code:
  - PHP: `htmlspecialchars()` or `htmlentities()`.
  - NodeJS: `encode()` from `html-entities` library (e.g., `<` → `&lt;`).
- **Never Display Raw Input Directly** – Avoid patterns like `echo $_GET['param']`.

### Server Configuration
| Measure | Purpose |
|---------|---------|
| **HTTPS everywhere** | Prevents man‑in‑the‑middle script injection. |
| `X‑Content‑Type‑Options: nosniff` | Stops browser from MIME‑type sniffing, which could execute scripts in non‑script files. |
| **Content‑Security‑Policy** | Restricts script sources (e.g., `script-src 'self'`), blocking inline scripts and unauthorized sources. |
| **`HttpOnly` cookies** | Prevent client‑side JavaScript from reading cookies, significantly reducing session theft. |
| **`Secure` cookie flag** | Ensures cookies are only transmitted over HTTPS. |
| **Web Application Firewall (WAF)** | Detects and blocks common XSS payloads in HTTP traffic. |

## Why It Matters
Even a single XSS vulnerability can lead to account takeover, data theft, or site defacement. Implementing these preventive measures is essential for any production web application. Understanding the full defense chain helps penetration testers identify which layers are missing or misconfigured and provide actionable remediation advice.

## Defender Perspective
- **Detection:** Missing headers (CSP, `X-Content-Type-Options`), lack of output encoding, and reflected input that retains `<` or `>` are strong indicators of XSS risk.
- **Mitigations prioritized:** Output encoding is the most effective single defense; CSP provides a critical second layer; input validation reduces attack surface.
- **MITRE ATT&CK:** Mitigations include *M1053 – Data Backup* (not directly), *M1021 – Restrict Web‑Based Content*, *M1022 – Restrict File and Directory Permissions*, but primarily falls under *M1045 – Code Signing* and secure coding practices.

## Key Takeaways
- Front‑end validation is a convenience, not a security control; always replicate it on the back‑end.
- Context matters: escaping for HTML body differs from escaping for attributes, JavaScript, or CSS. Use context‑specific encoders.
- DOMPurify is excellent for sanitizing HTML input, but it’s not a reason to skip output encoding.
- CSP can block inline scripts entirely, making even successful stored XSS injections harmless (unless the policy is overly permissive).
- HttpOnly is a powerful anti‑session‑theft measure—without it, any XSS can easily steal cookies.
- Regularly update sanitization libraries and WAF rules to keep pace with evolving bypass techniques.

## Gotchas
- `addslashes()` alone is insufficient for output into HTML—it escapes quotes but not `<` or `>`. Use `htmlentities()` for display.
- CSP `script-src 'self'` will break legitimate inline scripts; use nonces or hashes for approved inline scripts, or move all scripts to external files.
- If the `HttpOnly` flag is set but an XSS vulnerability exists, the attacker can still make authenticated requests via `XMLHttpRequest` (session riding) because the cookies are automatically attached.
- Neither front‑end validation nor WAFs can be solely relied upon; both can be bypassed with crafted payloads.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|xss]]  
← [[07-xss-phishing]]
<!-- AUTO-LINKS-END -->
