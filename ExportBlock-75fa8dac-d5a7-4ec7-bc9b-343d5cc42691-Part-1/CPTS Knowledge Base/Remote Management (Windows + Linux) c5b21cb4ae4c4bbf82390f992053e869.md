# Remote Management (Windows + Linux)

Category: Enumeration

## Windows remote management

### RDP (3389/tcp, 3389/udp)

- Purpose: Encrypted GUI access to Windows hosts.
- What to check
    - NLA/CredSSP enabled (harder to brute-force + better security)
    - TLS vs legacy “RDP Security” / self-signed cert warnings
    - Account lockout / MFA / allowed users (Remote Desktop Users)
- Quick enumeration
    - `nmap -p3389 --script "rdp-enum-encryption,rdp-ntlm-info,rdp-vuln-ms12-020" <IP>`
- Connect from Linux
    - `xfreerdp /v:<IP> /u:<USER> /p:<PASS> /cert:ignore /dynamic-resolution`

### WinRM (5985/http, 5986/https)

- Purpose: PowerShell remoting / WinRS.
- What to check
    - 5986 (HTTPS) preferred; 5985 often exposed internally
    - Auth methods (Negotiate/Kerberos/NTLM; Basic should be avoided unless over TLS)
    - Whether it’s enabled (servers often are; clients usually require enabling)
- Quick enumeration
    - `nmap -p5985,5986 --script http-title,http-auth-finder <IP>`
    - (On Windows) `Test-WsMan <IP>`
- Common pentest usage (valid creds)
    - `evil-winrm -i <IP> -u <USER> -p <PASS>`

### WMI / DCOM (135/tcp + dynamic high ports)

- Purpose: Deep system management (processes, services, event logs, etc.).
- What to check
    - RPC endpoint mapper reachable (135) and dynamic ports allowed
    - Local admin / WMI permissions (often requires admin-equivalent)
- Quick enumeration
    - `nmap -p135,139,445 <IP>` (WMI lateral movement usually accompanies SMB/RPC exposure)
- Common pentest usage (valid creds)
    - `impacket-wmiexec <DOMAIN>/<USER>:<PASS>@<IP> "hostname"`

---

## Linux remote management

### SSH (22/tcp)

- Purpose: Encrypted remote shell + port forwarding + file transfer (scp/sftp).
- High-risk `sshd_config` settings
    - `PasswordAuthentication yes` (brute-force/password spray exposure)
    - `PermitEmptyPasswords yes`
    - `PermitRootLogin yes`
    - `Protocol 1` (legacy/insecure)
    - Unnecessary features: `X11Forwarding yes`, broad `AllowTcpForwarding yes` (tunneling)
- Quick enumeration
    - `nmap -p22 -sV --script "ssh2-enum-algos,ssh-hostkey" <IP>`
    - `ssh-audit <IP>`

### Rsync (873/tcp, or over SSH)

- Purpose: Fast file sync/copy; common for backups/mirroring.
- Misconfig to look for
    - Anonymous/weakly protected modules allowing listing and download (secrets, scripts, keys)
- Quick enumeration
    - `nmap -p873 --script rsync-list-modules <IP>`
    - `rsync rsync://<IP>/` (list modules)
    - `rsync -av rsync://<IP>/<module>/ ./loot/` (pull files)

### Legacy R-services (512/513/514 tcp)

- Services: `rsh`, `rlogin`, `rexec`, `rcp`, etc. (unencrypted; trust-based auth).
- Abuse pattern
    - Trust files: `/etc/hosts.equiv` and per-user `~/.rhosts`
    - Dangerous wildcards like `+` can trust “any host” / “any user”
- Quick enumeration
    - `nmap -p512-514 -sV <IP>`
    - User discovery: `rwho`, `rusers` (if exposed)

---

## Pentest checklist (fast)

- Identify exposure: 22, 873, 512-514, 3389, 5985/5986, 135 (plus SMB 445).
- For each service, capture:
    - Version/banner, security mode (TLS/NLA/HTTPS), auth methods, and any anonymous access.
- With credentials, test the lowest-friction remote execution first:
    - Windows: WinRM (`evil-winrm`) → WMI (`wmiexec`) → RDP.
    - Linux: SSH → rsync modules → r-services.

## Sources

- OpenSSH release notes: [https://www.openssh.com/txt/release-8.2](https://www.openssh.com/txt/release-8.2)
- OpenSSH hardening + ssh-audit discussion: [https://blog.jeanbruenn.info/2023/12/23/hardening-your-openssh-configuration-do-you-know-about-the-tool-ssh-audit/](https://blog.jeanbruenn.info/2023/12/23/hardening-your-openssh-configuration-do-you-know-about-the-tool-ssh-audit/)
- Rapid7 rsync exposure deep dive: [https://www.rapid7.com/blog/post/2020/09/25/nicer-protocol-deep-dive-internet-exposure-of-rsync/](https://www.rapid7.com/blog/post/2020/09/25/nicer-protocol-deep-dive-internet-exposure-of-rsync/)
- Berkeley r-commands overview: [https://handwiki.org/wiki/Software:Berkeley_r-commands](https://handwiki.org/wiki/Software:Berkeley_r-commands)