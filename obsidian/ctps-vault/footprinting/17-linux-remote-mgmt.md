# NOTE — Linux Remote Management Protocols

## ID
601

## Module
Footprinting

## Kind
notes

## Title
Section 17 — Linux Remote Management Protocols

## Description
SSH (TCP/22) auth methods + dangerous configs + version recon via ssh-audit; Rsync (TCP/873) anonymous share enumeration; legacy R-services (rcp/rsh/rexec/rlogin on TCP/512–514) and the `.rhosts` / `/etc/hosts.equiv` trust-relationship abuse.

## Tags
ssh, rsync, r-services, rlogin, rsh, rexec, rcp, rwho, rusers, openssh, ssh-audit, hosts.equiv, rhosts

## Commands
### SSH
- `ssh-audit <IP>` — full server audit (algorithms, host keys, known-vuln versions)
- `ssh -v <user>@<IP>` — verbose connection — reveals server's allowed auth methods
- `ssh -v <user>@<IP> -o PreferredAuthentications=password` — force password auth (e.g. for brute attempts)

### Rsync
- `sudo nmap -sV -p 873 <IP>` — fingerprint
- `nc -nv <IP> 873` then `@RSYNCD: <ver>` then `#list` — list shares
- `rsync -av --list-only rsync://<IP>/<share>` — list a share
- `rsync -av rsync://<IP>/<share>` — pull entire share
- `rsync -av rsync://<IP>/<share> -e ssh` — over SSH
- `rsync -av rsync://<IP>/<share> -e "ssh -p2222"` — over SSH on non-std port

### R-services
- `sudo nmap -sV -p 512,513,514 <IP>`
- `rlogin <IP> -l <user>` — log in (no password if `.rhosts`/`hosts.equiv` allows)
- `rwho` — list logged-in users on the local LAN (UDP/513 broadcasts)
- `rusers -al <IP>` — detailed list of authenticated users on a host

## Concept Overview
Three categories of Linux remote management protocols. **SSH is the secure default**; Rsync is a common file-sync utility (often left exposed); R-services are legacy and almost always indicate weak posture if you find them.

---

## SSH (Secure Shell)

Replaces telnet/rsh. Encrypted, integrity-checked, with multiple auth methods. **TCP/22** by default. **OpenSSH** is the dominant implementation.

### SSH-1 vs SSH-2
- **SSH-1** — vulnerable to MitM, deprecated.
- **SSH-2** — current, mandatory. Banner `SSH-2.0-OpenSSH_8.2p1` = SSH-2 only. Banner `SSH-1.99-OpenSSH_3.9p1` = both v1 and v2 accepted (bad).

### Auth Methods (six total)
- Password
- **Public-key** (most common)
- Host-based
- Keyboard-interactive
- Challenge-response
- GSSAPI (Kerberos)

### Public-Key Auth Flow
1. Server sends its host key → client verifies (TOFU on first use).
2. Server sends a challenge encrypted with the user's *public* key (from `~/.ssh/authorized_keys`).
3. Client decrypts with the *private* key (passphrase-protected) and returns proof.
4. No password ever transits the wire.

### Default Config — `/etc/ssh/sshd_config`
Most settings are commented out — defaults rule. Notable defaults:
- `Include /etc/ssh/sshd_config.d/*.conf`
- `ChallengeResponseAuthentication no`
- `UsePAM yes`
- `X11Forwarding yes` ⚠️ (CVE-2016-3115 — command injection in OpenSSH 7.2p1 X11 forwarding)
- `PrintMotd no`
- `Subsystem sftp /usr/lib/openssh/sftp-server`

### Dangerous Settings
| Setting | Why dangerous |
|---------|--------------|
| `PasswordAuthentication yes` | Enables brute force against known users |
| `PermitEmptyPasswords yes` | Anyone can `ssh user@host` if user has no password |
| `PermitRootLogin yes` | Direct root login → catastrophic on compromise |
| `Protocol 1` | SSH-1 — broken crypto |
| `X11Forwarding yes` | Command injection in vulnerable versions |
| `AllowTcpForwarding yes` | Enables port-forward pivoting once authed |
| `PermitTunnel yes` | Allows tun/tap tunneling |
| `DebianBanner yes` | Leaks distro version in banner |

### ssh-audit
```
git clone https://github.com/jtesta/ssh-audit.git && cd ssh-audit
./ssh-audit.py <IP>
```
Output flags weak crypto (`[fail]`, `[warn]`), legacy algorithms, version, and banner — feeds OS fingerprinting and CVE lookup (e.g. CVE-2020-14145 → MitM in initial connection attempt).

### Banner Decoding
- `SSH-2.0-OpenSSH_8.2p1 Ubuntu-4ubuntu0.3` → SSH-2 only, OpenSSH 8.2p1, on Ubuntu (specific Ubuntu package version → distro version pin)
- `SSH-1.99-OpenSSH_3.9p1` → accepts both v1 and v2 (legacy install)

---

## Rsync

Fast file copy + delta-transfer (only sends changed bytes). Used for backups + mirroring. **TCP/873** by default. Can ride on top of SSH (`-e ssh`).

### Methodology
1. **Fingerprint** — `sudo nmap -sV -p 873 <IP>` returns `protocol version 31`.
2. **List shares** — `nc -nv <IP> 873` → server greets with `@RSYNCD: 31.0`. Send `#list` and read share names.
3. **Inspect a share** — `rsync -av --list-only rsync://<IP>/<share>` shows contents + perms + sizes.
4. **Pull files** — `rsync -av rsync://<IP>/<share>` mirrors locally.
5. **If creds required** — `rsync -av rsync://<user>@<IP>/<share>` and provide password.

### What to Look For
- `.ssh/` directories — SSH keys → instant lateral move via SSH.
- `secrets.yaml` / `*.env` / `config.*` — credentials.
- Backup tarballs — historical secrets.
- `build.sh` etc. — clues about the target's pipeline.

---

## R-Services (Legacy)

Pre-SSH suite from BSD/UCB CSRG. **Plaintext, host-based trust.** Mostly extinct, but seen on Solaris, HP-UX, AIX. Worth checking on internal pentests — if R-services are running, password reuse and weak auth are likely systemic.

### The Suite
| Command | Daemon | Port | Transport | Purpose |
|---------|--------|------|-----------|---------|
| `rcp` | `rshd` | 514 | TCP | Remote file copy (no overwrite warning) |
| `rsh` | `rshd` | 514 | TCP | Open a shell (no login prompt) — relies on `.rhosts`/`hosts.equiv` |
| `rexec` | `rexecd` | 512 | TCP | Run command remotely (user/pass over plaintext) — overridden by trust files |
| `rlogin` | `rlogind` | 513 | TCP | Like telnet but Unix-only — overridden by trust files |
| `rwho` | — | UDP 513 | UDP | Lists authenticated users on the LAN (broadcast-based) |
| `rusers` | — | RPC | — | Detailed per-host user list |

### Trust Files
| File | Scope |
|------|-------|
| `/etc/hosts.equiv` | Global — all users |
| `~/.rhosts` | Per-user, in user's $HOME |

Format: `<host> <localusername>`. The `+` wildcard = "anything." `+ +` = "any host, any user."

```
pwnbox cry0l1t3       # specific
+      10.0.17.10     # any user from this IP
+      +              # any user from anywhere — trivial pwn
```

### Attack Path
1. **Find R-services running** — `nmap -sV -p 512,513,514`.
2. **Try `rlogin`** as common usernames — `rlogin <IP> -l root`, `-l admin`, `-l htb-student`. If a `.rhosts` exists in the user's home with `+ +`, you're in **without a password**.
3. **Once in, `rwho` + `rusers`** — map other authenticated users for further targeting.

---

## Important Notes Across All Three

- **Password reuse is the connecting tissue** — IPMI passwords, MySQL passwords, Rsync-pulled SSH keys, R-services auth — they all overlap on weakly-managed networks.
- **Treat any leaked SSH private key as a high-value finding** — pull from Rsync shares, NFS exports, GitHub repos, S3 buckets. Don't grep for them; *list every directory*.

## Lab — Questions & Answers
*(No standalone lab in Section 17 — credential reuse from earlier sections feeds the three skills-assessment labs at the end of the module. See [[19-lab-easy]], [[20-lab-medium]], [[21-lab-hard]].)*

## Key Takeaways
- `ssh-audit` is one command for full SSH posture audit — run it before anything else.
- `ssh -v` reveals the server's accepted auth methods → tells you whether brute force is even worth trying.
- Rsync over TCP/873 with **no password** (anonymous module) is alarmingly common — `nc -nv <IP> 873; #list` is the 30-second check.
- R-services on a modern network = paint a giant red flag; expect weak posture across the board.
- **Always grep your downloaded haul** for `id_rsa`, `id_dsa`, `id_ecdsa`, `BEGIN PRIVATE KEY`.

## Gotchas
- A successful `ssh -v` showing `Authentications that can continue: publickey,password,keyboard-interactive` does NOT mean all three actually work — `PasswordAuthentication no` may still apply post-handshake.
- Rsync `-av` preserves permissions/timestamps but downloads everything — large shares can be slow and noisy.
- `rlogin` is sometimes blocked at the TCP layer even when `rshd` is up — check both 513 and 514.
- `rwho` requires `rwhod` to be running on the target *and* multicast/broadcast on the network — silent on segmented LANs.
- `~/.rhosts` files require ownership by the target user (or root) and 600 perms — security checks may reject them otherwise.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[16-ipmi]] | [[19-lab-easy]] →
<!-- AUTO-LINKS-END -->
