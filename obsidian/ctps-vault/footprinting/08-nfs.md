# NOTE — NFS

## ID
21

## Module
Footprinting

## Kind
notes

## Title
Section 8 — NFS

## Description
NFSv2/3/4 enumeration: showmount, Nmap NSE (nfs-ls/showmount/statfs), mount with `-o nolock`, then UID/GID-spoof tricks for lateral movement and SUID-shell escalation.

## Tags
nfs, rpcbind, mount, showmount, nfs-ls, uid-gid, root-squash, sun-rpc

## Commands
- `sudo nmap <IP> -p111,2049 -sV -sC` — basic enumeration
- `sudo nmap --script nfs* <IP> -sV -p111,2049` — full NFS NSE suite
- `showmount -e <IP>` — list exports
- `mkdir target-NFS && sudo mount -t nfs <IP>:/ ./target-NFS/ -o nolock`
- `ls -l mnt/nfs/` — view files with usernames
- `ls -n mnt/nfs/` — view files with **numeric** UIDs/GIDs (key for lateral move)
- `sudo umount ./target-NFS`

## Concept Overview
NFS = Sun's network file system. **v2/v3 authenticate the host** (UID/GID-based, trust-based — terrible without strong network controls). **v4 introduces user-level auth + Kerberos**, runs over TCP/2049 only (no portmapper required). Built on ONC-RPC (a.k.a. SUN-RPC) over **TCP/UDP 111** + **TCP/UDP 2049**. Uses XDR for system-independent data exchange.

### Version Quick-ref
| Version | Key |
|---------|-----|
| v2 | Old, mostly UDP, broad support |
| v3 | Variable file size, better errors, **not fully compatible with v2 clients** |
| v4 | Kerberos, firewall-friendly, no portmapper, ACLs, stateful |
| v4.1 | pNFS, session trunking (multipathing) |

## Configuration File — `/etc/exports`
Each line = `path  host(opts) host(opts)` — what's exported, to whom, with what permissions.

### Common Options
| Option | Effect |
|--------|--------|
| `rw` | Read+write |
| `ro` | Read-only |
| `sync` | Synchronous I/O (slower, safer) |
| `async` | Async (faster, riskier) |
| `secure` | Require source port < 1024 (root-only on client) |
| `insecure` | Allow source port ≥ 1024 — ⚠️ any user can mount |
| `no_subtree_check` | Disable subtree checks |
| `root_squash` | Map remote root → anonymous (default; protective) |
| `no_root_squash` | Remote root keeps UID 0 — ⚠️ trivial root file write |
| `nohide` | Mounted-below filesystems exported with own entry |

### Dangerous Combinations
- `rw` + `no_root_squash` → as root on client, write any file owned as UID 0 on server (e.g. SUID `/bin/bash`).
- `insecure` → any unprivileged user can mount.
- `nohide` → see filesystems mounted *under* the export.

## Methodology
1. **Probe RPC + NFS ports** — `sudo nmap <IP> -p111,2049 -sV -sC`. The `rpcinfo` NSE script lists every RPC program (mountd, nfs, nlockmgr, nfs_acl) and the ports they bind.
2. **List exports** — `showmount -e <IP>` or `sudo nmap --script nfs* <IP> -sV -p111,2049`.
3. **Mount it** — create empty dir, `sudo mount -t nfs <IP>:/<path> ./local -o nolock`. `nolock` disables NLM (often required against weird servers).
4. **`ls -l` then `ls -n`** — first shows usernames (mapped to your local `/etc/passwd`), second shows raw UIDs.
5. **Mirror UIDs locally** — if interesting files are owned by UID 1000 on the server, `useradd -u 1000` locally then `su` to that user → you can read/write per the file's mode.
6. **Test for `no_root_squash`** — try writing/chmod-ing a SUID binary as root. If the SUID survives on the server side, you have an SUID escalation vector for any shell on the server.

## Important NSE Scripts
| Script | Output |
|--------|--------|
| `nfs-showmount` | Same as `showmount -e` — exports + allowed clients |
| `nfs-ls` | Mounts read-only, lists files, shows access bits |
| `nfs-statfs` | Filesystem stats (size, used, free, max file size) |
| `rpcinfo` | All RPC programs + their ports |

## Lateral Movement Tricks
- **UID/GID match** — server has no idea who you "really" are; create a local user with matching UID and you become that server-side user.
- **SUID shell drop** — copy `bash` to the share, `chmod +s` it (requires `no_root_squash`), then any low-priv shell on the server can `./bash -p` for an instant root escalation.
- **Backup script tampering** — if `backup.sh` is executed by root on the server but is writable on the share (often via `no_root_squash`), inject your payload.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — `flag.txt` in the **`nfs`** share | **`HTB{hjglmvtkjhlkfuhgi734zthrie7rjmdze}`** | `showmount -e <IP>` reveals `/var/nfs`, then `mkdir NFS && sudo mount -t nfs -v <IP>:/var/nfs ./NFS/` → `cat ./NFS/flag.txt` |
| Q2 — `flag.txt` in the **`nfsshare`** share | **`HTB{8o7435zhtuih7fztdrzuhdhkfjcn7ghi4357ndcthzuc7rtfghu34}`** | Same `showmount` reveals `/mnt/nfsshare`, then `mkdir NFSShare && sudo mount -t nfs -v <IP>:/mnt/nfsshare ./NFSShare/` → `cat ./NFSShare/flag.txt` |

## Key Takeaways
- **`ls -n` after mount** is the diagnostic — it tells you whether the export is naive about identity.
- `no_root_squash` is the holy grail — full server-side root file primitive.
- Always test `nolock` if mount hangs.
- The server has no idea who you actually are; user identity is whatever client UID you send.

## Gotchas
- `root_squash` (default) maps your root → `nobody:nogroup` — even `chmod` fails. Don't waste time trying to write as root unless you've confirmed `no_root_squash`.
- Some servers refuse mounts from unprivileged source ports unless `insecure` is set.
- `umount` after every test — orphaned mounts confuse later scans.
- If a share is exported only to a specific subnet, `mount` will return permission-denied — check `showmount -e` carefully.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[07-smb]] | [[09-dns]] →
<!-- AUTO-LINKS-END -->
