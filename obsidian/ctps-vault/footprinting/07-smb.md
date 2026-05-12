# NOTE — SMB

## ID
20

## Module
Footprinting

## Kind
notes

## Title
Section 7 — SMB

## Description
SMB share enumeration, null sessions, version negotiation, RID brute via rpcclient, and parallel tooling (smbclient, smbmap, crackmapexec, enum4linux-ng, samrdump). Covers Samba defaults and dangerous settings.

## Tags
smb, samba, cifs, rpcclient, smbclient, smbmap, crackmapexec, enum4linux, null-session, rid-brute

## Commands
- `smbclient -N -L //<IP>` — null-session list of shares
- `smbclient //<IP>/<share>` — connect; `<Enter>` for empty password if anon
- `smbclient //<IP>/<share>` then `help`, `ls`, `get <file>`, `!<localcmd>`
- `rpcclient -U "" <IP>` — null-session RPC client
- `rpcclient $> srvinfo` / `enumdomains` / `querydominfo` / `netshareenumall`
- `rpcclient $> netsharegetinfo <share>`
- `rpcclient $> enumdomusers` / `queryuser <RID>` / `querygroup <RID>`
- RID brute (Bash): `for i in $(seq 500 1100); do rpcclient -N -U "" <IP> -c "queryuser 0x$(printf '%x\n' $i)" | grep "User Name\|user_rid\|group_rid" && echo ""; done`
- `samrdump.py <IP>` — Impacket alternative for user enum
- `smbmap -H <IP>` — share permissions overview
- `crackmapexec smb <IP> --shares -u '' -p ''` — auth-less share enum
- `enum4linux-ng -A <IP>` — automated full SMB recon
- `sudo nmap <IP> -sV -sC -p139,445`
- Server-side: `smbstatus` — see who's connected

## Concept Overview
SMB (Server Message Block) = Windows-native file/print/IPC sharing. **CIFS** is essentially SMBv1. **Samba** is the open-source SMB server for Linux/Unix; from v4 it can be a full AD DC. Two daemons: `smbd` (SMB) and `nmbd` (NetBIOS naming).

### Version → OS Mapping
| SMB | Shipped with | Notable feature |
|-----|--------------|-----------------|
| CIFS | Windows NT 4.0 | NetBIOS only |
| 1.0 | Windows 2000 | Direct over TCP |
| 2.0 | Vista / 2008 | Perf, signing, caching |
| 2.1 | Win7 / 2008 R2 | Locking improvements |
| 3.0 | Win8 / 2012 | Multichannel, end-to-end encryption |
| 3.0.2 | Win8.1 / 2012 R2 | — |
| 3.1.1 | Win10 / 2016 | Integrity check, AES-128 |

## Key Configuration File — Samba
`/etc/samba/smb.conf` — `[global]` section + per-share sections.

### Defaults (notable)
- `[printers]` and `[print$]` shares created by default for printer drivers.
- `usershare allow guests = yes` — guest sharing allowed.
- `map to guest = bad user` — invalid users silently fall through to guest.

### Dangerous Per-Share Settings
| Setting | Why it's dangerous |
|---------|-------------------|
| `browseable = yes` | Share appears in listings — easier to find |
| `read only = no` / `writable = yes` | Allows file write |
| `guest ok = yes` | No password required |
| `enable privileges = yes` | Honors SID-bound privileges |
| `create mask = 0777` / `directory mask = 0777` | World-writable files/dirs |
| `logon script = <sh>` | Auto-runs on user login |
| `magic script = <sh>` + `magic output = <out>` | Runs script when file is closed — can be triggered remotely |

## Methodology
1. **List shares with null session** — `smbclient -N -L //<IP>`. Add `-U ""` flavors if needed.
2. **Identify version + signing** — Nmap `smb2-security-mode`, `smb-os-discovery`.
3. **Connect to interesting shares** with `smbclient //<IP>/<share>` (try empty password first).
4. **Inside the session**: `recurse ON`, `prompt OFF`, `mget *` to bulk-pull files. `!<cmd>` runs local commands.
5. **Pivot to RPC** — `rpcclient -U "" <IP>` and run the enumeration commands below.
6. **Enumerate users via RID brute** when `enumdomusers` is restricted but `queryuser <RID>` isn't.
7. **Run multiple tools** — smbmap, crackmapexec, enum4linux-ng often disagree. Cross-reference.

## Important RPC Queries
| Query | What it gives you |
|-------|-------------------|
| `srvinfo` | Server name, OS version, role |
| `enumdomains` | Domains hosted |
| `querydominfo` | Domain, server, user counts, server role |
| `netshareenumall` | All shares (incl. hidden) + paths |
| `netsharegetinfo <share>` | ACL + permission bits for one share |
| `enumdomusers` | Domain user list (often restricted) |
| `queryuser <RID>` | Per-user details (rarely restricted — RID brute target) |
| `querygroup <RID>` | Group name + members |

## Tool Comparison
| Tool | Best for |
|------|----------|
| `smbclient` | Interactive, file get/put, manual exploration |
| `smbmap` | Quick share-permissions table |
| `crackmapexec` | Bulk SMB recon + exec across many hosts |
| `rpcclient` | Manual RPC queries — full control |
| `enum4linux-ng` | One-shot automated enum (shares, users, OS, policies, dialect) |
| `samrdump.py` (Impacket) | User enum via SAMR |

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Q1 — SMB server version (banner) | **`Samba smbd 4.6.2`** | `sudo nmap -p445 -sV -sC <IP>` |
| Q2 — Name of the accessible share | **`sambashare`** | `smbclient -N -L <IP>` lists `print$`, `sambashare`, `IPC$` |
| Q3 — Contents of `flag.txt` in that share | **`HTB{o873nz4xdo873n4zo873zn4fksuhldsf}`** | `smbclient //<IP>/sambashare -N` → `get contents\flag.txt` → `!cat contents\\flag.txt` |
| Q4 — Domain the server belongs to | **`DEVOPS`** | `rpcclient -U "" <IP>` → `querydominfo` |
| Q5 — Customised version of the share (`remark`) | **`InFreight SMB v3.1`** | Inside rpcclient: `netsharegetinfo sambashare` → `remark` field |
| Q6 — Full system path of the share | **`/home/sambauser`** | Same `netsharegetinfo sambashare` output → `path: C:\home\sambauser\` (Samba prepends `C:`) |

## Key Takeaways
- **Always try null session first** (`-N` / `-U ""`). Misconfigured Samba accepts it surprisingly often.
- **RID brute (`queryuser 0x<hex>`)** sidesteps a locked-down `enumdomusers`.
- **`smbstatus`** on the server side shows your connection too — be aware in opsec-sensitive engagements.
- Different tools yield different results — never trust one. enum4linux-ng misses stuff rpcclient catches and vice-versa.
- A `[notes]` or `[dev]` share with `guest ok = yes` + `writable = yes` is a free win — list, dump, then test write.

## Gotchas
- `enumdomusers` may be denied but `queryuser <RID>` allowed — the bypass is to RID-brute.
- "Anonymous login successful" but `ls` errors → you're authenticated to IPC$ but not authorised on the share.
- `enable privileges = yes` only matters when SID-based ACLs grant something interesting — note it but don't waste time if no SIDs are mapped.
- SMB1 disabled on modern boxes — older tools may misreport "no shares" when they really mean "no SMB1". Use `--option=client min protocol=NT1` on smbclient if you must.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|footprinting]]
← [[06-ftp]] | [[08-nfs]] →
<!-- AUTO-LINKS-END -->
