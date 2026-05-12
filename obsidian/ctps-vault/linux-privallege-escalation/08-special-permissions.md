# Special Permissions

## ID
701

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 8 — Special Permissions

## Description
Covers SUID/SGID bit enumeration, GTFOBins lookups, and leveraging misconfigured special permissions to escalate privileges on Linux.

## Tags
suid, sgid, gtfobins, privilege-escalation, find, special-permissions

## Commands
- `find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null`
- `find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null`
- `find / -uid 0 -perm -6000 -type f 2>/dev/null`
- `sudo apt-get update -o APT::Update::Pre-Invoke::=/bin/sh`

## What This Section Covers
SUID (Set User ID) and SGID (Set Group ID) are special file permissions that let a binary execute with the privileges of its owner or group rather than the calling user. When root-owned binaries have these bits set and also have escape features (shell spawning, command execution), they become direct privilege escalation vectors. This section teaches how to enumerate these binaries and cross-reference them against GTFOBins.

## Methodology
1. **Enumerate SUID binaries** — run `find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null` to list all root-owned files with the setuid bit.
2. **Enumerate SGID binaries** — run `find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null` to find files with both SUID and SGID set.
3. **Diff against known defaults** — compare results against the standard output shown in the module (or a known-good baseline) to spot unusual entries.
4. **Cross-reference GTFOBins** — for every non-standard SUID/SGID binary, check [GTFOBins](https://gtfobins.github.io/) for known abuse techniques (shell escape, file read/write, reverse shell).
5. **Exploit** — use the GTFOBins technique to escalate. If the binary isn't on GTFOBins, consider reverse engineering it for vulnerabilities (e.g., shared object hijacking, command injection).

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — SUID binary not in section output | /bin/sed | Ran `find / -user root -perm -4000 ...` and compared against the section's listed output; `/bin/sed` was the extra entry |
| Q2 — SGID binary not in section output | /usr/bin/facter | Ran `find / -user root -perm -6000 ...` and compared; `/usr/bin/facter` was the extra entry |

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

### Q1 — Find the extra SUID binary
```
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
```

```
htb-student@NIX02:~$ find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null

-rwsr-xr-x 1 root root 16728 Sep  1  2020 /home/htb-student/shared_obj_hijack/payroll
-rwsr-xr-x 1 root root 16728 Sep  1  2020 /home/mrb3n/payroll
-rwSr--r-- 1 root root 0 Aug 31  2020 /home/cliff.moore/netracer
-rwsr-xr-x 1 root root 40152 Nov 30  2017 /bin/mount
-rwsr-xr-x 1 root root 40128 May 17  2017 /bin/su
-rwsr-xr-x 1 root root 73424 Feb 12  2016 /bin/sed           # <-- NOT in section output
-rwsr-xr-x 1 root root 27608 Nov 30  2017 /bin/umount
-rwsr-xr-x 1 root root 44680 May  7  2014 /bin/ping6
-rwsr-xr-x 1 root root 30800 Jul 12  2016 /bin/fusermount
-rwsr-xr-x 1 root root 44168 May  7  2014 /bin/ping
-rwsr-xr-x 1 root root 142032 Jan 28  2017 /bin/ntfs-3g
-rwsr-xr-x 1 root root 38984 Jun 14  2017 /usr/lib/x86_64-linux-gnu/lxc/lxc-user-nic
-rwsr-xr-- 1 root messagebus 42992 Jan 12  2017 /usr/lib/dbus-1.0/dbus-daemon-launch-helper
-rwsr-xr-x 1 root root 14864 Jan 18  2016 /usr/lib/policykit-1/polkit-agent-helper-1
-rwsr-sr-x 1 root root 85832 Nov 30  2017 /usr/lib/snapd/snap-confine
-rwsr-xr-x 1 root root 428240 Jan 18  2018 /usr/lib/openssh/ssh-keysign
-rwsr-xr-x 1 root root 10232 Mar 27  2017 /usr/lib/eject/dmcrypt-get-device
-rwsr-xr-x 1 root root 23376 Jan 18  2016 /usr/bin/pkexec
-rwsr-sr-x 1 root root 240 Feb  1  2016 /usr/bin/facter
-rwsr-xr-x 1 root root 39904 May 17  2017 /usr/bin/newgrp
-rwsr-xr-x 1 root root 32944 May 17  2017 /usr/bin/newuidmap
-rwsr-xr-x 1 root root 49584 May 17  2017 /usr/bin/chfn
-rwsr-xr-x 1 root root 136808 Jul  4  2017 /usr/bin/sudo
-rwsr-xr-x 1 root root 40432 May 17  2017 /usr/bin/chsh
-rwsr-xr-x 1 root root 32944 May 17  2017 /usr/bin/newgidmap
-rwsr-xr-x 1 root root 75304 May 17  2017 /usr/bin/gpasswd
-rwsr-xr-x 1 root root 54256 May 17  2017 /usr/bin/passwd
-rwsr-xr-x 1 root root 10624 May  9  2018 /usr/bin/vmware-user-suid-wrapper
-rwsr-xr-x 1 root root 1588768 Aug 31  2020 /usr/bin/screen-4.5.0
-rwsr-xr-x 1 root root 94240 Jun  9  2020 /sbin/mount.nfs
```

Compare against the section's listed output — `/bin/sed` is the extra entry.

Answer: `/bin/sed`

### Q2 — Find the extra SGID binary
```
find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null
```

```
htb-student@NIX02:~$ find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null

-rwsr-sr-x 1 root root 85832 Nov 30  2017 /usr/lib/snapd/snap-confine
-rwsr-sr-x 1 root root 240 Feb  1  2016 /usr/bin/facter      # <-- NOT in section output
```

The section only showed `/usr/lib/snapd/snap-confine`. The lab box also returns `/usr/bin/facter`.

Answer: `/usr/bin/facter`

## Key Takeaways
- The `s` in `rwsr-xr-x` replaces the execute bit — lowercase `s` means the execute bit is also set (functional SUID), uppercase `S` means only the SUID bit is set without execute (non-functional, like `netracer` in the example).
- `-perm -4000` finds SUID, `-perm -2000` finds SGID, `-perm -6000` finds both SUID+SGID set simultaneously.
- GTFOBins is the go-to reference during exams — bookmark it. Any SUID binary listed there with a "SUID" section is an instant privesc.
- `sed` with SUID is dangerous — it can read/write arbitrary files as root (e.g., inject a root-level user into `/etc/passwd`).
- `facter` (a Puppet utility) with SGID is unusual and worth investigating — non-standard SUID/SGID binaries are almost always the intended attack vector on CTFs and exams.
- Always diff your `find` output against a known baseline — the extra binary is the one you exploit.

## Gotchas
- Don't confuse `-perm -4000` (SUID only) with `-perm -6000` (both SUID and SGID). The questions ask for each separately — use the right flag.
- The uppercase `S` in `-rwSr--r--` means the SUID bit is set but the file is **not executable** — it won't run as root. Only lowercase `s` is exploitable.
