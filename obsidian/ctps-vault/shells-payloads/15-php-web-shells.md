# NOTE — PHP Web Shells

## ID
412

## Module
Shells & Payloads

## Kind
notes

## Title
Section 15 — PHP Web Shells

## Description
Deploy WhiteWinterWolf's PHP web shell against rConfig 3.9.6 by abusing the vendor-logo upload — bypass the image-only file-type filter with Burp by changing `Content-Type` from `application/x-php` to `image/gif`.

## Tags
php, web-shells, burp, content-type-bypass, rconfig, whitewinterwolf, file-upload-bypass

## Commands
- `git clone https://github.com/WhiteWinterWolf/wwwolf-php-webshell.git` — get the shell
- (Login to rConfig at `admin:admin`, Devices → Vendors → Add Vendor)
- (Burp proxy listening on `127.0.0.1:8080`, browser configured to match)
- (Intercept upload → change `Content-Type: application/x-php` → `image/gif` → Forward)
- Navigate to `/images/vendor/<shellname>.php` — execute shell in browser

## What This Section Covers
PHP powers ~78% of websites whose server language is identifiable. When you spot PHP (e.g. `.php` login page, `X-Powered-By` header), a PHP web shell is the natural choice. rConfig's "Add Vendor" form accepts a logo image — bypass the type filter with Burp's intercept and you get RCE.

## Why PHP Web Shells?
- PHP is server-side → all code executes on the host.
- File extension `.php` triggers server-side processing.
- Most PHP shells are single-file, drop-in, browser-driven.
- WhiteWinterWolf's shell ships a clean web UI with command execution, file ops, etc.

## Content-Type Bypass — The Trick
The app's upload validator only inspects the `Content-Type` header (not file magic bytes or extension). Burp intercepts the upload POST, you edit the header in place:

```
Content-Type: application/x-php   ← original
Content-Type: image/gif           ← edited
```

The server sees "looks like an image", accepts the file, saves it with the `.php` extension intact. Browser request to `/images/vendor/connect.php` executes PHP because the server still keys execution off file extension.

## Walkthrough — rConfig Vendor Logo Upload
1. **Login** to rConfig: `admin:admin` (default).
2. **Navigate**: Devices → Vendors → Add Vendor.
3. **Configure Burp**: browser proxy → `127.0.0.1:8080`. Accept Burp's CA if prompted.
4. **Upload**: pick the PHP shell as the vendor logo. Click Save.
5. **In Burp**: intercept the upload POST. Find the part with the `.php` file. Change `Content-Type: application/x-php` → `Content-Type: image/gif`. Strip in-file comments if any. Forward.
6. **Confirm**: "Added new vendor X to Database" + vendor entry shows the ripped-paper icon (means the server didn't recognize it as an image but didn't reject it).
7. **Execute**: browse to `/images/vendor/connect.php` (or whatever filename you uploaded). Shell UI loads — issue commands.

## Considerations When Using Web Shells
- Apps may auto-delete uploads after a time period.
- Limited interactivity — chaining (`whoami && hostname`) often breaks.
- Unstable — non-interactive shells can hang on long-running commands.
- Higher forensic footprint.
- Engagement-type matters: black-box evasive tests demand stealth → upload → drop reverse shell → delete payload.

## Documentation Discipline
Every method tried, what worked, what didn't, filenames, upload locations. Include sha1/MD5 hashes of payload files in the report for attribution.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — What must `Content-Type` be changed to? | **image/gif** | "Bypassing the File Type Restriction" subsection — replace `application/x-php` with `image/gif`. |
| Q2 — Filename of the gif in `/images/vendor/` on the target | **ajax-loader.gif** (from prior Linux lab listing) | After exploitation, list `/images/vendor/` via the shell — the same path also had `ajax-loader.gif`, `cisco.jpg`, `juniper.jpg` from Section 10. |

## Key Takeaways
- File-type validation by `Content-Type` is trivially bypassed in Burp; magic-byte validation is harder.
- The "broken image" icon after upload is a tell that the server stored the file but couldn't parse it as image — you got your PHP in.
- Always remove author comments from the shell before upload — `whitewinterwolf` strings are signatured.
- After getting RCE, immediately drop a proper reverse shell and *delete the web shell file*.

## Gotchas
- Firefox 90+ removed FTP — don't try to deliver shells over FTP via Firefox.
- Burp CA must be trusted before HTTPS upload pages will play nice — accept the cert when prompted.
- Some validators check magic bytes AND extension — add a fake GIF89a header to the start of your `.php` if needed.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[14-antak-webshell]] | [[16-skills-assessment-live-engagement]] →
<!-- AUTO-LINKS-END -->
