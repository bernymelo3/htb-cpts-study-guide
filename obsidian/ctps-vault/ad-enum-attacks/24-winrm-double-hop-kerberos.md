
---

## Plain-English Explanation

**The problem in one sentence:**  
When you use `evil-winrm` or `Enter-PSSession` to get a remote PowerShell shell on a Windows box, that box can't "prove" to the Domain Controller who you are when you try to run tools like PowerView. So everything fails.

**Why it happens:**

- Normally when you log into Windows (RDP, physical login), your password is processed, and your computer gets two important pieces: a **TGT** (Ticket Granting Ticket — "I am user X, and the DC trusts me") and a **TGS** (Ticket Granting Service — "I am allowed to access specific server Y").
- When you connect via **WinRM** (which uses Kerberos), the remote host only gets the TGS for its *own* HTTP service. It *never* receives your TGT. It also never receives your NTLM hash (unlike PSExec or SMB-based tools).
- So when you're sitting on that remote shell and you run `get-domainuser` from PowerView, the remote host tries to query the DC. The DC says, "Who's asking?" The remote host can't answer, because it doesn't have your TGT. Error.

Think of it like this: you check into a hotel (the DC) and get a keycard (TGT). You go to a restaurant (DEV01 via WinRM). The restaurant gets a slip that says "this guest is allowed in the restaurant" (TGS), but they don't get your actual hotel keycard. When you then try to charge something to your room at the gift shop (DC01), the gift shop asks for proof you're a hotel guest. The restaurant can't provide it because they never had your keycard.

**Why RDP doesn't have this problem:**  
RDP sends your actual password through, which gets stored as an NTLM hash in memory on the remote host. That hash can then be used to authenticate you to other machines. `klist` will show a proper TGT.

**The two fixes:**

1. **PSCredential Object (Workaround #1):**  
   You manually re-send your username and password with every PowerView command. It's like showing your ID again every time you want to charge something to your room. Works from `evil-winrm` / Linux, but tedious.

2. **Register PSSession Configuration (Workaround #2):**  
   You set up a special WinRM session that actually *does* run with your full credentials. Now the remote host has your TGT cached, and everything works natively. This requires a real PowerShell console (not `evil-winrm`), so you'd use it from a Windows attack VM or after RDP'ing into a compromised host.

That's the section. Let me know if you want me to process the next one.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|ad-enum-attacks]]  
← [[23-privileged-access]] | [[25-bleeding-edge-vulnerabilities]] →
<!-- AUTO-LINKS-END -->
