# NOTE — Cracking Protected Files

## ID
507

## Module
Password Attacks

## Kind
methodology

## Title
Section 6 — Cracking Protected Files

## Description
Crack encrypted SSH private keys, Office documents (docx, xlsx), and PDFs by extracting hashes with the `2john` helper family and running them through JtR or hashcat.

## Tags
ssh2john, office2john, pdf2john, encrypted-files, ssh-key, password-protected-docs

## Commands
- `ssh2john id_rsa > ssh.hash`
- `office2john Confidential.xlsx > office.hash`
- `pdf2john PDF.pdf > pdf.hash`
- `john --wordlist=rockyou.txt <hash_file>`
- `john <hash_file> --show`
- `ssh-keygen -yf ~/.ssh/id_rsa` (test if key is passphrase-protected)

## Concept Overview
Most "encrypted" file formats embed a verification hash derived from the password. The `2john` family extracts that hash into JtR-compatible format, then standard offline cracking applies.

## Hunting Encrypted Files on Linux
```bash
for ext in $(echo ".xls .xls* .xltx .od* .doc .doc* .pdf .pot .pot* .pp*"); do
  echo -e "\nFile extension: " $ext
  find / -name *$ext 2>/dev/null | grep -v "lib\|fonts\|share\|core"
done
```

## SSH Private Keys
SSH keys start with `-----BEGIN [...] PRIVATE KEY-----`. Find them with:
```bash
grep -rnE '^\-{5}BEGIN [A-Z0-9]+ PRIVATE KEY\-{5}$' /* 2>/dev/null
```

### Detecting if a Key is Encrypted
Try to read the public key:
```bash
ssh-keygen -yf ~/.ssh/id_rsa
```
- No prompt → unencrypted (just use it)
- "Enter passphrase" → encrypted, crack it

### Cracking Procedure
```bash
ssh2john id_rsa > ssh.hash
john --wordlist=rockyou.txt ssh.hash
john ssh.hash --show
```

## Office Documents (.docx / .xlsx / .pptx)
```bash
office2john Protected.docx > office.hash
john --wordlist=rockyou.txt office.hash
```
Hashcat mode: `9400` (Office 2007), `9500` (2010), `9600` (2013).

## PDF Files
```bash
pdf2john PDF.pdf > pdf.hash
john --wordlist=rockyou.txt pdf.hash
```
Hashcat modes: `10400` (PDF 1.1–1.3), `10500` (1.4–1.6), `10600` (1.7 Level 3), `10700` (1.7 Level 8).

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Crack `Confidential.xlsx` from `cracking-protected-files.zip` | **(hidden)** | `office2john Confidential.xlsx > hash` then `john --wordlist=rockyou.txt hash` |

## Key Takeaways
- The `2john` script names match the file type: `ssh2john`, `zip2john`, `rar2john`, `office2john`, `pdf2john`, `keepass2john`, `bitlocker2john`.
- Modern Office encryption (2013+) is comparatively slow — expect minutes-to-hours of cracking even on a strong dictionary.
- Encrypted SSH keys with weak passphrases are a common quick win during internal engagements (developer workstations).

## Gotchas
- Some `2john` tools are Python scripts (`pdf2john.py`, `office2john.py`) — others are compiled binaries (`zip2john`, `bitlocker2john`). Use `locate *2john*` to find them on your distro.
- Modern OpenSSH private keys (post-2018) all look the same in header form whether encrypted or not — you can't tell by reading the file; use `ssh-keygen -yf` to test.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[05-writing-custom-wordlists-and-rules]] | [[07-cracking-protected-archives]] →
<!-- AUTO-LINKS-END -->
