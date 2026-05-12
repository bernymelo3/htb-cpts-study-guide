# THEORY — Plugins & Mixins

## ID
608

## Module
Using the Metasploit Framework

## Kind
theory

## Title
Section 9 — Plugins & Mixins

## Description
Plugins extend msfconsole at runtime (Nessus, OpenVAS, sqlmap, etc.) — install by dropping `.rb` into `plugins/` and `load <name>`. Mixins are Ruby modules included into MSF classes for shared behavior.

## Tags
metasploit, plugins, mixins, nessus, darkoperator, ruby

## Commands
- `ls /usr/share/metasploit-framework/plugins`
- `load <plugin>` (e.g. `load nessus`, `load pentest`)
- `git clone https://github.com/darkoperator/Metasploit-Plugins`
- `sudo cp ./Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb`
- `msfconsole -q`
- `<plugin>_help` (e.g. `nessus_help`)

## What This Section Covers
Plugins are 3rd-party Ruby scripts that hook into msfconsole's API at runtime to add commands and integrate external tools (Nessus, NexPose, OpenVAS, sqlmap, Mimikatz, …). Mixins are a Ruby-language concept the Framework uses internally — modules-as-mix-ins for shared methods.

## Plugin Filesystem
Default path:
```
/usr/share/metasploit-framework/plugins/
```
Common pre-installed: `nessus.rb`, `nexpose.rb`, `openvas.rb`, `sqlmap.rb`, `wmap.rb`, `aggregator.rb`, `pcap_log.rb`, `session_notifier.rb`, `socket_logger.rb`, `token_adduser.rb`, `token_hunter.rb`, `sounds.rb`, `wiki.rb`, …

Per-user overrides also live under `~/.msf4/plugins/`.

## Loading a Plugin
```
load nessus
[*] Nessus Bridge for Metasploit
[*] Type nessus_help for a command listing
[*] Successfully loaded Plugin: Nessus

nessus_help
```
Most plugins follow the `<name>_help` / `<name>_<verb>` convention.

If the plugin isn't installed:
```
load Plugin_That_Does_Not_Exist
[-] Failed to load plugin from /usr/share/metasploit-framework/plugins/Plugin_That_Does_Not_Exist.rb: cannot load such file
```

## Installing a New Plugin
Drop the `.rb` into the plugins folder with correct perms:
```
git clone https://github.com/darkoperator/Metasploit-Plugins
sudo cp ./Metasploit-Plugins/pentest.rb /usr/share/metasploit-framework/plugins/pentest.rb
msfconsole -q
load pentest
```
DarkOperator's `pentest` plugin adds command categories like `Tradecraft`, `auto_exploit`, `Discovery`, `Project`, `Postauto` — including `network_discover`, `pivot_network_discover`, `multi_meter_cmd`, `multi_post`, etc.

## Notable Plugin List
| Plugin | Purpose |
|--------|---------|
| nmap (pre-installed) | External nmap integration |
| NexPose (pre-installed) | Rapid7 NexPose scanner bridge |
| Nessus (pre-installed) | Tenable Nessus bridge |
| Mimikatz V.1 (pre-installed) | Credential extraction (legacy — Kiwi is the successor) |
| Stdapi (pre-installed) | Meterpreter standard API |
| Incognito (pre-installed) | Token manipulation |
| Railgun | Direct Windows API calls from Meterpreter |
| Priv | Privilege manipulation |
| DarkOperator's | Pentest workflow automation |

## Mixins (Ruby Concept)
Mixins are Ruby modules included into classes via `include <Module>` so the class gains the module's methods without inheritance. Used when:
- A class needs many optional features (mix only the ones you want).
- One feature needs to be shared across many unrelated classes.

You don't need to write mixins to use Metasploit, but they're behind most of the framework's flexibility — every `Msf::Exploit::Remote::HttpClient` etc. is a mixin.

## Key Takeaways
- A plugin is just a `.rb` in `plugins/` — installing one is `cp` + `load`. No package manager involved.
- After `load <name>`, look for `<name>_help` first — every plugin has its own subcommand prefix.
- Mimikatz the plugin is **gone** post-Aug-2020; loading "Mimikatz" inside Meterpreter actually loads **Kiwi**.

## Gotchas
- Plugin permissions: if you `cp` as root but msfconsole runs as your user, fix ownership/perms or `load` fails silently.
- Plugin names use snake_case alphanumerics + underscores — dashes or other chars in the filename break the `load` call.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[08-databases]] | [[10-sessions-and-jobs]] →
<!-- AUTO-LINKS-END -->
