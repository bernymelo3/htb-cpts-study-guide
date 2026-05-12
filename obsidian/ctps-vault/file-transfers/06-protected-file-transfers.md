# THEORY — Protected File Transfers

## ID
620

## Module
File Transfers

## Kind
theory

## Title
Section 6 — Protected File Transfers

## Description
Encrypting sensitive loot before transfer — Invoke-AESEncryption.ps1 for Windows, openssl with PBKDF2 for Linux — so exfilled data (NTDS.dit, hashes, configs) is safe in transit even over plaintext channels.

## Tags
encryption, aes, openssl, pbkdf2, exfil, opsec, invoke-aesencryption, data-protection

## Commands
- `Import-Module .\Invoke-AESEncryption.ps1`
- `Invoke-AESEncryption -Mode Encrypt -Key "<pwd>" -Path .\scan-results.txt`
- `Invoke-AESEncryption -Mode Decrypt -Key "<pwd>" -Path .\scan-results.txt.aes`
- `Invoke-AESEncryption -Mode Encrypt -Key "<pwd>" -Text "<plaintext>"`
- `openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc`
- `openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd`

## TL;DR — What's Important
- Loot (NTDS.dit, hashes, configs, creds) **must be encrypted before exfil** — defenders' IDS can read plaintext and clients will (rightly) ask why creds went over the wire in the clear.
- Use **unique strong passwords per engagement** — one cracked archive doesn't expose every other client's data.
- Don't exfil real PII / financial data unless explicitly scoped — use dummy data for DLP testing.

## What This Section Covers
Pentesters routinely handle highly sensitive data (NTDS.dit, hashes, config files with creds). Even when the transfer channel is encrypted (HTTPS/SFTP/SSH), there are times you must use a plaintext channel — at which point you need to encrypt the *file* itself before transfer.

## Windows — `Invoke-AESEncryption.ps1`

A small PowerShell script that does AES-256-CBC with SHA-256 key derivation. Encrypts strings or files; output file gets `.aes` extension.

### Usage
```powershell
# Import once
Import-Module .\Invoke-AESEncryption.ps1

# Encrypt a string → base64 ciphertext
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Text "Secret Text"

# Decrypt a base64 ciphertext string
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Text "LtxcRelxrDLrDB9rBD6JrfX/czKjZ2CUJkrg++kAMfs="

# Encrypt a file → file.aes
Invoke-AESEncryption -Mode Encrypt -Key "p@ssw0rd" -Path file.bin

# Decrypt a file
Invoke-AESEncryption -Mode Decrypt -Key "p@ssw0rd" -Path file.bin.aes
```

### Crypto Parameters (from the script)
| Parameter | Value |
|-----------|-------|
| Algorithm | AES-256 CBC |
| Key derivation | SHA-256 of UTF-8(`-Key`) |
| Block size | 128 |
| Padding | Zeros |
| IV | Prepended to ciphertext (first 16 bytes) |

## Linux — `openssl enc`

`openssl` is on virtually every Linux box (sysadmins use it for certs). Recommended flags:

```bash
# Encrypt
openssl enc -aes256 -iter 100000 -pbkdf2 -in /etc/passwd -out passwd.enc

# Decrypt
openssl enc -d -aes256 -iter 100000 -pbkdf2 -in passwd.enc -out passwd
```

### Why these specific flags
| Flag | Why |
|------|-----|
| `-aes256` | Strong cipher; widely supported. |
| `-pbkdf2` | Modern KDF — far better than the default MD5-based KDF (`EVP_BytesToKey`). |
| `-iter 100000` | Slow brute-force; override the (low) default iteration count. |
| `-d` | Decrypt mode (omit for encrypt). |

> Without `-pbkdf2`, openssl uses an MD5-based KDF that has been cryptographically broken for password-based encryption — **always pass `-pbkdf2`**.

## OPSEC Rules

1. **One password per engagement, randomly generated.** If a client's archive leaks and the password's cracked, no other client's data is exposed.
2. **Don't exfil real PII unless scoped.** For DLP testing, create dummy data that *looks* like the real data (e.g., a fake `customers.csv` with synthetic SSNs).
3. **Prefer encrypted transport too** — SFTP/SSH/HTTPS — file encryption is a second layer, not a substitute.
4. **Delete loot from compromised hosts** after exfil completes and verify deletion.

## Key Takeaways
- Encrypting the *file* protects loot even when the *channel* is plaintext (and protects you from IDS triggering on plaintext hashes).
- `openssl enc` without `-pbkdf2` uses a broken KDF — silently produces weak encryption. Always include the flag.
- `Invoke-AESEncryption.ps1` is small and easy to transfer over base64 paste — useful when you're already on a restricted host.
- Anything exfilled must be deletable / accounted for — clients ask about this in the report.

## Gotchas
- Different openssl versions changed the default KDF (1.1.0 → 1.1.1) — **always pass `-pbkdf2`** rather than rely on defaults.
- AES-CBC with zero padding (used by `Invoke-AESEncryption.ps1`) can leave trailing zero bytes on decrypted files — fine for text but can break exact binary diffs. Use a different padding mode if hash equality is required.
- A leaked / weak engagement password = full loot exposure if the encrypted blob is intercepted. Treat the engagement password like a master key.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|file-transfers]]
← [[05-miscellaneous-file-transfer-methods]] | [[07-catching-files-over-https]] →
<!-- AUTO-LINKS-END -->
