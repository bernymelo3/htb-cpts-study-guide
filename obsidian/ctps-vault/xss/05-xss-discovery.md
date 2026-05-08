## ID
530

## Module
Cross-Site Scripting (XSS)

## Kind
notes

## Title
Section 5 — XSS Discovery

## Description
Covers automated and manual methods for discovering XSS vulnerabilities, including XSStrike, payload lists, and manual code review, with a hands-on exercise identifying a Reflected XSS in an email parameter.

## Tags
xss, reflected-xss, xsstrike, discovery, recon, web-attacks

## Commands
- git clone https://github.com/s0md3v/XSStrike.git
- pip install -r requirements.txt
- python xsstrike.py -u "http://<TARGET_IP>:<PORT>/index.php?<PARAM>=test"
- curl -s http://<TARGET_IP>:<PORT>/ | grep -i "input\|name=\|action="

## What This Section Covers
XSS discovery techniques ranging from automated scanners (Nessus, Burp Pro, ZAP, XSStrike) to manual payload testing and code review. Automated tools work by identifying input fields, injecting payloads, and comparing the rendered page source — but always require manual verification. Manual code review (both front-end and back-end) remains the most reliable method for finding XSS in mature applications.

## Methodology
1. Identify all input parameters — inspect the page source with `curl -s <URL> | grep -i "input\|name=\|action="` to find form fields and parameter names.
2. **Submit the form first** to get a fully populated URL (e.g. `/?fullname=test&username=test&password=123&email=test@email.com`) — testing params individually may fail if the app requires all fields.
3. Run XSStrike against each parameter using the full URL: `python xsstrike.py -u "<FULL_URL>"`.
4. Look for "Reflections found" in XSStrike output — this indicates the parameter value is rendered back into the page.
5. Manually verify by injecting a basic payload like `<script>alert(1)</script>` into the reflected parameter via the URL.
6. Determine XSS type: if the payload reflects in an error/response message after GET submission → **Reflected**; if it persists on reload → **Stored**; if it only executes via client-side JS sink → **DOM**.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Name of the vulnerable parameter | email | Submitted the registration form to get full URL, then tested `email` parameter — value reflects back in the page |
| Q2 — What type of XSS? | Reflected | Input reflects in an error message after being processed via GET, does not persist |

## Key Takeaways
- **Submit the form before scanning** — XSStrike (and similar tools) may report "No reflection found" if the app requires all parameters to be present for processing. Always get a valid URL from a real submission first.
- Most payloads from public lists (PayloadAllTheThings, Payload-Box) won't work on a given target because they're designed for specific injection contexts (after quotes, inside attributes, bypassing filters). Context matters.
- XSS injection isn't limited to form input fields — HTTP headers like `Cookie` and `User-Agent` can also be injection vectors when their values are rendered on the page.
- Automated tools give false negatives frequently; a tool saying "no reflection" doesn't mean the parameter is safe.
- For mature/public web apps, manual code review is typically the only way to find XSS — developers already run automated scanners before release.

## Gotchas
- Testing parameters one at a time with XSStrike gave "No reflection found" on all four params — the app needed all fields populated together to process the request and reflect the `email` value.
- XSStrike's `-u` flag expects the parameter to already be in the URL query string — it won't discover parameters on its own from form HTML.


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|xss]]  
← [[04-dom-xss]] | [[06-defacing]] →
<!-- AUTO-LINKS-END -->
