## ID
204

## Module
Using Web Proxies

## Kind
notes

## Title
Section 6 — Automatic Modification

## Description
Covers setting up automatic match‑and‑replace rules in Burp Suite and ZAP to modify requests and responses on the fly, such as spoofing User-Agent or permanently altering client‑side HTML.

## Tags
burp-suite, zap, match-and-replace, automation, proxy, header-modification

## Commands
- Burp: Proxy → Proxy settings → HTTP match and replace rules → Add
- ZAP: Ctrl+R → Replacer → Add, or via Options menu → Replacer
- Match request header: `^User-Agent.*$` → replace with `User-Agent: HackTheBox Agent 1.0`
- Match response body: `type="number"` → replace with `type="text"`
- Match response body: `maxlength="3"` → replace with `maxlength="100"`
- ZAP request body match: `ip` → replace with `ip;ls;` (exercise)

## What This Section Covers
Automatic modification allows you to define match‑and‑replace rules that the proxy applies to every request or response without manual interception. This is essential for bypassing filters (e.g., spoofing User-Agent), permanently altering client‑side validation, or injecting payloads into every request automatically — making repeated tests faster and less error‑prone.

## Methodology
1. In Burp, navigate to **Proxy → Proxy settings → HTTP match and replace rules** and click **Add**.
2. For automatic User-Agent spoofing, set:
   - **Type:** Request header
   - **Match:** `^User-Agent.*$` (regex to capture the whole header line)
   - **Replace:** `User-Agent: HackTheBox Agent 1.0`
   - **Regex match:** True
3. Click **Ok** and ensure the rule is enabled. All subsequent requests will carry the custom User-Agent.
4. In ZAP, press **Ctrl+R** or go to **Options → Replacer**, click **Add**.
   - **Match Type:** Request Header String (or *Request Header* with the header name selected from the dropdown)
   - **Match String:** `User-Agent`
   - **Replacement String:** `HackTheBox Agent 1.0`
   - **Enable:** True
5. For automatic response modification (e.g., enabling any input in a numeric field), add another rule:
   - **Type:** Response body
   - **Match:** `type="number"`
   - **Replace:** `type="text"`
   - **Regex match:** False
6. Add a second response rule to extend the maximum length: match `maxlength="3"` and replace with `maxlength="100"`.
7. Force‑refresh the page (Ctrl+Shift+R) — the changes now persist across reloads without manual interception.
8. For the exercise, in ZAP create a request‑body replacer rule that matches `ip` and replaces it with `ip;ls;` to inject a command on every Ping submission.

## Multi-step Workflow (optional)

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|web-proxies]]  
← [[05-intercepting-responses]] | [[06-zap-replacer-notes]] →
<!-- AUTO-LINKS-END -->
