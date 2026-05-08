## ID
640

## Module
File Upload Attacks

## Kind
notes

## Title
Section 9 — Other Upload Attacks

## Description
Examines file upload attacks beyond code execution, including filename injection (command injection, XSS, SQLi), upload directory disclosure via error forcing, Windows‑specific quirks (reserved names, 8.3 convention), and exploiting automatic file processing libraries.

## Tags
file-upload, filename-injection, directory-disclosure, windows-quirks, advanced-attacks

## Commands
- `touch 'file$(whoami).jpg'` (command injection in filename)
- `touch 'file`whoami`.png'` (command injection via backticks)
- `touch '<script>alert(1)</script>.svg'` (XSS in filename)
- `touch "file';select+sleep(5);--.jpg"` (SQL injection in filename)
- upload a file named `CON.jpg`, `NUL.png` (Windows reserved name)
- upload `HAC~1.TXT` to overwrite `hackthebox.txt` (8.3 short name)

## What This Section Covers
This section goes beyond the core goal of uploading a web shell. It explains how an attacker can abuse the **filename itself** as an injection vector if the filename is used unsafely in OS commands, displayed on a page, or inserted into SQL. It also covers tricks to **leak the upload directory** by provoking informative errors (duplicate files, overly long names, Windows reserved characters/names) and highlights legacy **Windows 8.3 short‑filename** behaviour that can overwrite existing files. A brief look at **advanced processing exploits** (e.g., ffmpeg XXE) shows how much deeper the attack surface can go when uploaded files are automatically converted or resized.

## Methodology
1. **Filename injection tests:**  
   - Create a file with command substitution: `file$(whoami).jpg` and upload. If the server uses the name in a shell command, the `whoami` output may appear in the response or an error.  
   - Try backticks (`file\`id\`.png`) and pipe/OR (`file.jpg||ping -c 10 127.0.0.1`).  
   - Test XSS by naming a file `<svg onload=alert(1)>.svg` and check if the name is reflected unsanitized.  
   - Test SQLi by embedding `sleep` or boolean operators in the filename when it may be stored in a database.

2. **Forcing upload directory disclosure:**  
   - Upload the same file twice simultaneously (concurrent requests) to trigger a write conflict error that often prints the target path.  
   - Use an extremely long filename (5000+ characters) – a poorly handled overflow may leak directory info.  
   - If the upload link isn’t shown, fuzz the web root with common directories (`/uploads/`, `/files/`, `/user_content/`), or leverage IDOR/LFI to locate where files land.

3. **Windows‑specific attacks:**  
   - Include reserved characters in the filename (`|`, `<`, `>`, `*`, `?`). Without proper quoting, the server might interpret them as wildcards and error out, disclosing the path.  
   - Use reserved device names (`CON`, `NUL`, `COM1`, `LPT1`). Windows forbids creating files with these names; the failure can reveal the directory.  
   - Exploit 8.3 short filenames: if you can upload `WEB~1.CON` and the server writes it literally, it may overwrite `web.conf`. Test by trying to access the original file afterwards.

4. **Advanced automatic processing:**  
   - Identify any libraries that handle the uploaded file after arrival (e.g., ffmpeg, ImageMagick, unzip). Search for known CVEs affecting those versions.  
   - Test for XXE by crafting an SVG or AVI file that triggers out-of-band interactions if parsed by a vulnerable version.

## Key Takeaways
- Filenames are just as much user input as file content – never trust them without sanitisation.
- Error messages from file handling are gold for mapping the server’s filesystem; provoke them systematically.
- Windows legacy behaviours (reserved names, 8.3) create opportunities that don’t exist on Linux, making cross‑platform testing essential.
- A file upload vulnerability isn’t limited to gaining a shell; filename injection can lead to command execution, XSS, or SQL injection, and directory disclosure enables further attacks like LFI.
- Always investigate any automatic processing that happens after upload; it often imports third‑party libraries with known, exploitable bugs.

## Gotchas
- Some frameworks automatically rename uploaded files to a random string – this neutralises filename‑based injections, but check if the original name is still stored or reflected elsewhere.
- The 8.3 short‑name convention may be disabled on modern Windows servers (especially for non‑system drives), so test its availability before relying on it.
- Overly long filename attacks can crash the server or cause a DoS instead of leaking information, so use them cautiously and only with permission.
- Windows reserved device names might be blocked by the browser or by client‑side validation before the upload even reaches the server – bypass by sending the request directly with a tool like Burp.