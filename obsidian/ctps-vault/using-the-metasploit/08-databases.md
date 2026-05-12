# NOTES — Databases

## ID
607

## Module
Using the Metasploit Framework

## Kind
notes

## Title
Section 8 — Databases

## Description
Wire up PostgreSQL + msfdb so hosts, services, creds, and loot persist across sessions. Workspaces for project isolation, `db_import`/`db_nmap` for ingest, `db_export` for backup.

## Tags
metasploit, database, postgresql, msfdb, workspace, db_nmap, db_import, hosts, services, creds, loot

## Commands
- `sudo service postgresql status`
- `sudo systemctl start postgresql`
- `sudo msfdb init`
- `sudo msfdb status`
- `sudo msfdb run`
- `msfdb reinit`
- `cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/`
- `sudo service postgresql restart`
- `db_status`
- `help database`
- `workspace` / `workspace -a <name>` / `workspace -d <name>` / `workspace <name>` / `workspace -r <old> <new>` / `workspace -h`
- `db_import <file.xml>`
- `db_nmap -sV -sS <ip>` / `db_nmap -sV -p- -T5 -A <ip>`
- `hosts` / `hosts -h`
- `services` / `services -h`
- `creds` / `creds -h`
- `loot` / `loot -h`
- `db_export -f xml backup.xml`

## What This Section Covers
The msfconsole database (PostgreSQL-backed) keeps your scan results, hosts, services, credentials, and loot persistent and queryable. Workspaces let you separate engagements. `db_nmap` runs Nmap and ingests in one step; `db_import` ingests `.xml` from external Nmap/Nessus/etc.

## Initial Setup
```
sudo service postgresql status              # check
sudo systemctl start postgresql             # start if needed
sudo msfdb init                             # create user, msf + msf_test DBs, config
sudo msfdb run                              # launch msfconsole with DB connected
```

Confirm inside msfconsole:
```
db_status
[*] Connected to msf. Connection type: PostgreSQL.
```

## Recovering a Broken DB
If `msfdb init` aborts on an old install (e.g., Bundler error), update first then reinit:
```
apt update
sudo msfdb init
sudo msfdb status
```

Reset path if config password is broken:
```
msfdb reinit
cp /usr/share/metasploit-framework/config/database.yml ~/.msf4/
sudo service postgresql restart
msfconsole -q
```

## Database Help Menu
`help database` inside msfconsole. Key subcommands:
| Command | Use |
|---------|-----|
| `db_connect` | Connect to an existing DB |
| `db_disconnect` | Drop current connection |
| `db_export` | Export current workspace |
| `db_import` | Import a scan file (auto-detects type) |
| `db_nmap` | Run Nmap and auto-record results |
| `db_rebuild_cache` | Rebuild module cache |
| `db_status` | Show connection state |
| `hosts` / `services` / `vulns` / `loot` / `notes` | List recorded data |
| `workspace` | Switch / manage workspaces |

## Workspaces
Workspaces = project folders. The `*` marks the active one.
```
workspace                       # list
workspace -a Target_1           # add
workspace Target_1              # switch
workspace -d Target_1           # delete
workspace -D                    # delete all
workspace -r old new            # rename
workspace -v                    # verbose listing
```

Switch workspaces per target IP / subnet / domain to keep records clean.

## Importing Scan Results
External Nmap scan → XML → import:
```
db_import Target.xml
```
After import:
```
hosts
services
```
both populate automatically. `.xml` is the preferred format (richer than .nmap).

## Running Nmap Inside msfconsole
```
db_nmap -sV -sS 10.10.10.8
db_nmap -A --top-ports 60 -T5 <ip>
db_nmap -sV -p- -T5 -A 10.10.10.15
```
Output streams into msfconsole prefixed with `[*] Nmap:` and the hosts/services tables get populated.

## hosts / services
List + manage what's known about each target.
```
hosts -h                        # see all flags
hosts -R                        # set RHOSTS from results
hosts -c address,os_name,info   # column subset
hosts -d <ip>                   # delete entry
hosts -i / -n / -m              # edit info/name/comment
```
Same shape for `services -h` (filter by `-p <port>`, `-s <name>`, `-r <proto>`, `-u` up-only, `-R` set RHOSTS, etc.).

## creds — Credentials Store
Add captured creds in many formats (password, NTLM, Postgres MD5, SSH key, JtR hash, NonReplayableHash). Examples:
```
creds add user:admin password:notpassword realm:workgroup
creds add user:admin ntlm:E2FC...:A107...
creds add user:postgres postgres:md5be86a79bf...
creds add user:sshadmin ssh-key:/path/to/id_rsa
creds add hash:d19c32489b870735b5f587d76b934283 jtr:md5
```
List/filter:
```
creds                       # all
creds 1.2.3.4/24            # range
creds -p 22-25,445          # port spec
creds -s ssh,smb            # service names
creds -t NTLM               # type
creds -j md5                # JtR hash type
creds -d -s smb             # delete all SMB creds
creds -o out.jtr            # export to JtR format (extension matters: .jtr / .hcat / csv)
```

## loot
At-a-glance view of dumps (hashes, passwd, shadow, etc.).
```
loot                                # list
loot -a -f <file> -i <info> -t <type>   # add
loot -d <ip>                            # delete by host+type
```

## Backing Up
```
db_export -f xml backup.xml
```
Re-imports cleanly with `db_import` later.

## Key Takeaways
- New engagement → `workspace -a <target>` → `workspace <target>` first. Default workspace becomes a garbage dump otherwise.
- `db_nmap` instead of running Nmap externally — saves the import step and your scan results stay in the workspace.
- `.xml` is the right import format. `.nmap` text loses parsing fidelity.
- `creds` is your hash/password vault for the engagement — feed it pivoting credentials so `set RHOSTS` and `creds -R` can pull from it later.

## Gotchas
- `db_nmap -A --top-ports 60` is plenty for first-pass — full `-p-` scans inside msfconsole tie up the prompt for the entire scan duration (no easy way to background).
- If `msfdb init` says "already configured" but you can't connect, it's usually a password drift between `~/.msf4/database.yml` and the actual Postgres role. Recover via `msfdb reinit` + the copy step above.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[07-encoders]] | [[09-plugins-and-mixins]] →
<!-- AUTO-LINKS-END -->
