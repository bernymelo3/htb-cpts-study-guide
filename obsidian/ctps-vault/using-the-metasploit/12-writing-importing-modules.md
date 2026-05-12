# NOTES — Writing & Importing Modules

## ID
611

## Module
Using the Metasploit Framework

## Kind
notes

## Title
Section 12 — Writing & Importing Modules

## Description
Pull MSF-format exploits from ExploitDB or searchsploit, drop them into the right `modules/` subfolder with snake_case naming, and `reload_all` (or `loadpath`) to register them. Then how to port a non-MSF Ruby/Python/PHP exploit into a proper Metasploit module using existing modules as boilerplate.

## Tags
metasploit, modules, exploit-db, searchsploit, porting, ruby, mixins, reload-all, loadpath

## Commands
- `searchsploit nagios3`
- `searchsploit -t Nagios3 --exclude=".py"`
- `ls /usr/share/metasploit-framework/`
- `cp ~/Downloads/9861.rb /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb`
- `msfconsole -m /usr/share/metasploit-framework/modules/`
- `loadpath /usr/share/metasploit-framework/modules/`
- `reload_all`
- `use exploit/unix/webapp/nagios3_command_injection`

## What This Section Covers
Two distinct skills: (1) **importing** an existing MSF-format exploit `.rb` so msfconsole sees it locally without a full upstream update, and (2) **porting** a non-MSF exploit (Python/PHP/raw Ruby) into a proper Metasploit module by adapting an existing module as a template.

## Importing an Existing MSF Module
ExploitDB tags exploits as "Metasploit Framework (MSF)" — those `.rb` files drop in as-is.

### Find on ExploitDB
- Web: filter by `tag = Metasploit Framework`
- CLI: `searchsploit <keyword>`

```
searchsploit nagios3
# Nagios3 - 'history.cgi' Host Command Execution (Metasploit)        | linux/remote/24159.rb
# Nagios3 - 'history.cgi' Remote Command Execution                    | multiple/remote/24084.py
# Nagios3 - 'statuswml.cgi' 'Ping' Command Execution (Metasploit)    | cgi/webapps/16908.rb
# Nagios3 - 'statuswml.cgi' Command Injection (Metasploit)            | unix/webapps/9861.rb
```

Filter out non-Ruby noise:
```
searchsploit -t Nagios3 --exclude=".py"
```

### Install
Default install dir:
```
/usr/share/metasploit-framework/modules/exploits/<os>/<service>/
```
User-local override:
```
~/.msf4/modules/...                  # mirror the same folder structure
```

Copy in with a proper snake_case name:
```
cp ~/Downloads/9861.rb \
   /usr/share/metasploit-framework/modules/exploits/unix/webapp/nagios3_command_injection.rb
```

### Register
Either start msfconsole with an explicit module path:
```
msfconsole -m /usr/share/metasploit-framework/modules/
```
Or inside msfconsole:
```
loadpath /usr/share/metasploit-framework/modules/
# or
reload_all
```

Then `use exploit/unix/webapp/nagios3_command_injection` works and `show options` reveals the params.

## Naming Rules (will break if you ignore)
- Snake_case only
- Alphanumeric + underscores
- **No dashes**, no spaces
- Examples: `nagios3_command_injection.rb`, `our_module_here.rb`

## Not Every `.rb` is an MSF Module
Some ExploitDB Ruby exploits are stand-alone scripts (no `class MetasploitModule < Msf::Exploit::Remote`). Those need **porting**, not just copying.

## Porting a Script Into a Module
Don't write from scratch — take an existing similar module from the same `exploits/<os>/<service>/` directory and adapt it.

### Example: Porting Bludit 3.9.2 Auth Bruteforce Mitigation Bypass
1. Find boilerplate already in the framework:
   ```
   ls /usr/share/metasploit-framework/modules/exploits/linux/http/ | grep bludit
   # bludit_upload_images_exec.rb     ← use as template
   ```
2. Drop the new `.rb`:
   ```
   cp ~/Downloads/48746.rb \
      /usr/share/metasploit-framework/modules/exploits/linux/http/bludit_auth_bruteforce_mitigation_bypass.rb
   ```
3. Adjust the `include` mixins. Boilerplate had:
   ```ruby
   include Msf::Exploit::Remote::HttpClient
   include Msf::Exploit::PhpEXE
   include Msf::Exploit::FileDropper
   include Msf::Auxiliary::Report
   ```
   For the new exploit, drop `FileDropper` (no file cleanup needed); keep the others.

### Common Mixins (cheat sheet)
| Mixin | Purpose |
|-------|---------|
| `Msf::Exploit::Remote::HttpClient` | Act as an HTTP client against the target |
| `Msf::Exploit::PhpEXE` | Generate a first-stage PHP payload |
| `Msf::Exploit::FileDropper` | Transfer files + handle cleanup after session |
| `Msf::Auxiliary::Report` | Report data to the MSF database |

### Module Skeleton
```ruby
class MetasploitModule < Msf::Exploit::Remote
  Rank = ExcellentRanking

  include Msf::Exploit::Remote::HttpClient
  include Msf::Exploit::PhpEXE
  include Msf::Auxiliary::Report

  def initialize(info={})
    super(update_info(info,
      'Name'           => "Bludit 3.9.2 - Authentication Bruteforce Mitigation Bypass",
      'Description'    => %q{ ... },
      'License'        => MSF_LICENSE,
      'Author'         => [ 'rastating', '0ne-nine9' ],
      'References'     => [
        ['CVE', '2019-17240'],
        ['URL', '...'],
        ['PATCH', '...']
      ],
      'Platform'       => 'php',
      'Arch'           => ARCH_PHP,
      'Notes'          => {
        'SideEffects' => [ IOC_IN_LOGS ],
        'Reliability' => [ REPEATABLE_SESSION ],
        'Stability'   => [ CRASH_SAFE ]
      },
      'Targets'        => [[ 'Bludit v3.9.2', {} ]],
      'Privileged'     => false,
      'DisclosureDate' => "2019-10-05",
      'DefaultTarget'  => 0))

    register_options([
      OptString.new('TARGETURI', [true, 'The base path for Bludit', '/']),
      OptString.new('BLUDITUSER', [true, 'The username for Bludit']),
      OptPath.new('PASSWORDS', [ true, 'The list of passwords',
        File.join(Msf::Config.data_directory, "wordlists", "passwords.txt") ])
    ])
  end
end
```

Use `OptString` for free-text params, `OptPath` for filesystem path params (like a wordlist). Built-in wordlists live under `Msf::Config.data_directory + "wordlists/"`.

### Ruby + MSF Style Rules
- **Hard tabs** in MSF Ruby (not spaces). Mismatched indentation triggers warnings.
- Look up classes/methods on the official Metasploit Documentation site for current signatures.

## Key Takeaways
- ExploitDB's MSF-tagged `.rb`s drop straight in — no porting required if they declare `class MetasploitModule < Msf::Exploit::Remote`.
- Match the target's folder structure (`exploits/<os>/<service>/`) — that's how msfconsole namespaces the path you'll `use`.
- `reload_all` is the fast-iteration loop for module development — no msfconsole restart needed.
- Boilerplate-from-existing-module is the porting strategy — never write the metadata block from scratch.

## Gotchas
- Skip `reload_all` and msfconsole won't see your new file — old caches survive.
- Wrong filename (dash, space, capital) → silent skip during load.
- A `.rb` that looks like a Metasploit module but lacks `class MetasploitModule < ...` is just a Ruby script — won't load.

<!-- AUTO-LINKS-START -->
---
**Module:** [[00-overview|using-the-metasploit]]
← [[11-meterpreter]] | [[13-msfvenom]] →
<!-- AUTO-LINKS-END -->
