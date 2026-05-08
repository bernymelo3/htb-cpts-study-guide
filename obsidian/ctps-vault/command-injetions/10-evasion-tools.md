## ID
750

## Module
Command Injections

## Kind
notes

## Title
Section 10 — Evasion Tools

## Description
Introduces automated obfuscation tools (**Bashfuscator** for Linux bash commands and **DOSfuscation** for Windows CMD) that bypass advanced command‑injection filters when manual obfuscation techniques fall short.

## Tags
evasion, obfuscation, bashfuscator, dosfuscation, command-injection, tools

## Commands
- `git clone https://github.com/Bashfuscator/Bashfuscator`
- `pip3 install setuptools==65 && python3 setup.py install --user`
- `./bashfuscator -c '<COMMAND>' -s 1 -t 1 --no-mangling --layers 1`
- `bash -c '<OBFUSCATED_COMMAND>'`
- `git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git`
- `Import-Module .\Invoke-DOSfuscation.psd1`
- `Invoke-DOSfuscation`
- (inside DOSfuscation) `SET COMMAND type C:\Users\htb-student\Desktop\flag.txt`
- (inside DOSfuscation) `encoding` → `1`

## What This Section Covers
When a WAF or custom filter blocks obvious injection characters and commands, manual obfuscation (e.g., `${IFS}`, backticks, variable expansion) may not be enough. This section demonstrates two automated tools that generate highly obfuscated payloads: **Bashfuscator** produces randomised, multi‑layer bash payloads; **Invoke‑DOSfuscation** does the same for Windows batch/PowerShell commands. Both tools help turn a simple command into a string that looks nothing like the original but still executes identically.

## Methodology
1. **Set up Bashfuscator (Linux):**
   - Clone the repository: `git clone https://github.com/Bashfuscator/Bashfuscator`
   - Install dependencies: `pip3 install setuptools==65` then `python3 setup.py install --user`
   - Navigate to `bashfuscator/bin/` and run `./bashfuscator -h` to see options.

2. **Generate an obfuscated bash command:**
   - Basic random obfuscation: `./bashfuscator -c 'cat /etc/passwd'` (but output may be huge).
   - Controlled, compact output: add flags `-s 1 -t 1 --no-mangling --layers 1` for minimal size and complexity.
   - Test that the obfuscated command works: `bash -c '<OUTPUT_COMMAND>'`.

3. **Set up DOSfuscation (Windows / PowerShell):**
   - Clone the repo: `git clone https://github.com/danielbohannon/Invoke-DOSfuscation.git`
   - Import the module: `Import-Module .\Invoke-DOSfuscation.psd1`
   - Launch the interactive shell: `Invoke-DOSfuscation`
   - (Optional) Type `help` or `tutorial` inside the tool.

4. **Obfuscate a Windows command:**
   - Inside the tool, set the command: `SET COMMAND type C:\Users\htb-student\Desktop\flag.txt`
   - Then type `encoding` and pick option `1` (or other techniques) to generate a payload.
   - Copy the output and run it directly in `cmd.exe` (or in `pwsh` on Linux) to confirm execution.

5. **Use in real injection scenarios:**
   - Replace your simple payload with the tool’s output, being mindful of character limits and special characters that the web application may still block (the tool may still include forbidden chars; tweak options or layer with manual encoding).

## Key Takeaways
- Automated obfuscation is not a silver bullet: always test the generated payload in a local shell first; some WAFs may still block long strings or patterns.
- Bashfuscator’s flags (`-s`, `-t`, `--layers`) let you balance between stealth and size; shorter payloads are less suspicious but may still contain preserved words.
- DOSfuscation uses environment variable substrings (`%TEMP:~-3,-2%`) to reconstruct characters, making the command look like gibberish.
- Combining tool‑based obfuscation with manual encoding (URL encoding, double URL encoding, hex) can further increase evasion when dealing with layered filters.
- Always have a fallback: if the obfuscated command still fails, consider out‑of‑band techniques (DNS, ICMP) or writing a web shell in a different directory.

## Gotchas
- Bashfuscator can produce commands over a million characters long without the `-s` and `--layers` limitations; huge payloads can cause the target application to crash or truncate.
- DOSfuscation is interactive; automation inside scripts is harder – but you can record a series of commands and replay them via `SendKeys` or similar.
- Some obfuscated payloads still contain whitespace or special chars (`%`, `;`, `$`). Always check the output for characters that the specific injection point filters out.
- On Windows, `pwsh` (PowerShell Core) can be used on Linux to test DOS‑obfuscated commands, but environment variables like `%CommonProgramFiles%` may not exist, causing failure.