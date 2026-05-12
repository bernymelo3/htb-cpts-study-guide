# NOTE — Cracking Protected Archives

## ID
508

## Module
Password Attacks

## Kind
methodology

## Title
Section 7 — Cracking Protected Archives

## Description
Crack ZIP, OpenSSL-encrypted GZIP, and BitLocker-encrypted VHDs. Includes mounting BitLocker volumes from Linux with `dislocker`.

## Tags
zip2john, bitlocker2john, dislocker, openssl, gzip, vhd, archive-cracking

## Commands
- `zip2john ZIP.zip > zip.hash`
- `bitlocker2john -i Backup.vhd > backup.hashes`
- `grep "bitlocker\$0" backup.hashes > backup.hash`
- `hashcat -a 0 -m 22100 backup.hash rockyou.txt`
- `sudo losetup -f -P Backup.vhd`
- `sudo dislocker /dev/loop0p2 -u<password> -- /media/bitlocker`
- `sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount`
- `for i in $(cat rockyou.txt); do openssl enc -aes-256-cbc -d -in GZIP.gzip -k $i 2>/dev/null | tar xz; done`

## Concept Overview
Archive formats embed verification data derived from the password. Some (ZIP, RAR) support native passwords; others (TAR, GZIP) are wrapped with `openssl enc` or `gpg`. Each requires a different attack approach.

## Identifying the Container
```bash
file SuspiciousArchive.gzip
# → "openssl enc'd data with salted password" (not actually gzip)
```
Always run `file` before assuming the format — extensions lie.

## ZIP Files
```bash
zip2john ZIP.zip > zip.hash
john --wordlist=rockyou.txt zip.hash
john zip.hash --show
```
Hashcat mode: `13600` (WinZip), `17200`–`17220` (PKZIP variants).

## OpenSSL-Encrypted Files
No `2john` script — brute-force by attempting decryption directly:
```bash
for i in $(cat rockyou.txt); do
  openssl enc -aes-256-cbc -d -in GZIP.gzip -k "$i" 2>/dev/null | tar xz
done
```
- Errors flooding the terminal are expected (wrong password → garbage → tar errors).
- A new file appearing in `ls` = correct password found.

## BitLocker-Encrypted VHDs

### Extract Hash
`bitlocker2john` outputs 4 hashes total:
- `$bitlocker$0$...` — password (RC4-based KDF)
- `$bitlocker$1$...` — password (AES, used for cracking)
- `$bitlocker$2$...` — recovery key (RC4)
- `$bitlocker$3$...` — recovery key (AES)

Recovery keys are 48-digit random strings — not practical to crack. Target the **password** hashes:
```bash
bitlocker2john -i Backup.vhd > backup.hashes
grep "bitlocker\$0" backup.hashes > backup.hash
```

### Crack
Hashcat mode `22100`:
```bash
hashcat -a 0 -m 22100 backup.hash /usr/share/wordlists/rockyou.txt
```
Strong AES → very slow. Expect 25 H/s or less on CPU.

### Mounting from Linux (post-crack)
```bash
sudo apt install dislocker
sudo mkdir -p /media/bitlocker /media/bitlockermount

sudo losetup -f -P Backup.vhd       # attach VHD as loop device
losetup --all                        # confirm loop device name (e.g. /dev/loop0)

sudo dislocker /dev/loop0p2 -u<password> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount
ls -la /media/bitlockermount/
```

### Unmount
```bash
sudo umount /media/bitlockermount
sudo umount /media/bitlocker
sudo losetup -d /dev/loop0
```

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Crack the BitLocker VHD password (`Private.vhd` from target download) | **(hidden — see HTB walkthrough)** | `bitlocker2john -i Private.vhd > h && grep '\$0' h > h2 && hashcat -m 22100 h2 rockyou.txt` |
| Q2 — Read `flag.txt` from inside the mounted volume | **(hidden)** | After cracking: `losetup` + `dislocker` + `mount`, then `cat /media/bitlockermount/flag.txt` |

## Key Takeaways
- `bitlocker2john -i` is required — without it the tool only reads from stdin.
- BitLocker uses 1,048,576 iterations by default. Cracking is slow even on GPU.
- The `dislocker` mount step requires *two* mount points: one for the dislocker-file overlay, one for the actual filesystem.
- Most BitLocker drives in enterprise use have an MBAM-managed recovery key — if you can read AD attributes, you may get the recovery key directly without cracking.

## Gotchas
- VHDs containing multiple partitions need the partition number (`p1`, `p2`, …) on the loop device path. Check with `losetup --all` first.
- `losetup -f` finds the *next available* loop device — it might not be `loop0`. Read the output.
- Cracking `$bitlocker$2$` (recovery RC4) is wasted effort — recovery keys are 48 random digits.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[06-cracking-protected-files]] | [[08-network-services]] →
<!-- AUTO-LINKS-END -->
