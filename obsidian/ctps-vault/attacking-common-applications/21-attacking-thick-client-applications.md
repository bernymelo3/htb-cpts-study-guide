# Section 21 — Attacking Thick Client Applications

## ID
1902

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 21 — Attacking Thick Client Applications

## Description
Retrieving hardcoded credentials from a thick client application by capturing temp files with ProcMon, decoding an embedded Base64 executable, and reverse-engineering the .NET binary with x64dbg, de4dot, and dnSpy.

## Tags
thick-client, reverse-engineering, hardcoded-credentials, procmon, dnspy, x64dbg, de4dot

## Commands
- xfreerdp /v:<TARGET_IP> /u:<USER> /p:'<PASS>' /dynamic-resolution /drive:share,/home/<USER>
- .\Restart-OracleService.exe
- cat C:\programdata\monta.ps1
- powershell.exe -exec bypass -file c:\programdata\monta.ps1
- C:\TOOLS\Strings\strings64.exe .\<DUMP_FILE>.bin

## What This Section Covers
Thick client applications run locally and often store sensitive data on the client side, making them susceptible to hardcoded credential extraction, DLL hijacking, buffer overflows, and insecure storage. This section walks through a real-world scenario: an executable found on an SMB NETLOGON share drops a temporary batch file that decodes a Base64-embedded .NET binary. By intercepting the temp files (via ProcMon + folder permission manipulation), decoding the payload, dumping the .NET assembly from memory with x64dbg, deobfuscating with de4dot, and decompiling with dnSpy, hardcoded service credentials are recovered for lateral movement.

## Methodology
1. RDP into the target with a shared drive: `xfreerdp /v:<IP> /u:cybervaca /p:'&aue%C)}6g-d{w' /dynamic-resolution /drive:share,/home/<USER>`.
2. Launch **ProcMon64** from `C:\TOOLS\ProcessMonitor` and run `C:\Apps\Restart-OracleService.exe` from the command line.
3. Filter ProcMon by the process name — observe that it creates a temp `.bat` file in `C:\Users\<USER>\AppData\Local\Temp` and immediately deletes it.
4. Prevent deletion by modifying the Temp folder permissions: Properties → Security → Advanced → select user → Disable inheritance → Convert → Edit → uncheck **Delete subfolders and files** and **Delete**. Apply the same restrictions to SYSTEM and Administrators.
5. Run the executable again — the `.bat` file now persists in the Temp folder.
6. Edit the batch script in Notepad: remove the `del` lines for `monta.ps1`, `oracle.txt`, and `restart-service.exe` so the decoded artifacts are preserved.
7. Double-click the modified batch script to execute it, then confirm `oracle.txt` and `monta.ps1` exist in `C:\ProgramData`.
8. Run the PowerShell decoder script: `powershell.exe -exec bypass -file c:\programdata\monta.ps1` — this decodes `oracle.txt` (Base64) into `restart-service.exe`.
9. Open `restart-service.exe` in **x64dbg** (as Administrator). Set preferences to break only on **Exit Breakpoint** (uncheck all others).
10. Run the program, then open the **Memory Map** pane. Look for a region with type **MAP** and protection **-RW--** (size ~0x3000). Follow it in the dump and confirm `MZ` magic bytes in the ASCII column — this is a .NET assembly loaded in memory.
11. Right-click the region → **Dump Memory to File** to export the `.bin`.
12. Drag the `.bin` onto **de4dot** (`C:\TOOLS\de4dot\`) to deobfuscate → produces a `-cleaned.bin` file.
13. Open the cleaned binary in **dnSpy** — read the decompiled C# source to find the hardcoded credentials.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Perform an analysis of C:\Apps\Restart-OracleService.exe and identify the credentials hidden within its source code (username:password) | svc_oracle:#oracle_s3rV1c3!2010 | dnSpy decompilation of memory-dumped .NET binary |

## Key Takeaways
- Thick clients come in two-tier (app ↔ DB directly) and three-tier (app ↔ app server ↔ DB) architectures — two-tier is less secure since attackers can interact with the database directly.
- Executables that create and immediately delete temp files are a red flag — use ProcMon to identify the file paths, then lock the folder permissions to prevent deletion and capture the artifacts.
- Base64-encoded payloads embedded in batch scripts are a common obfuscation technique for dropping executables — the decode chain here was: batch → `oracle.txt` (Base64 lines) → `monta.ps1` (decoder) → `restart-service.exe`.
- Memory-mapped files in x64dbg with type MAP and -RW-- protection that start with `MZ` bytes are embedded executables — dump them for further analysis.
- de4dot strips .NET obfuscation, and dnSpy decompiles the cleaned binary back to readable C# source — together they defeat most basic .NET protections.
- Web-specific vulns (XSS, CSRF, Clickjacking) don't apply to thick clients, but they introduce their own attack surface: hardcoded creds, DLL hijacking, buffer overflows, insecure local storage, and SQL injection via direct DB connections.

## Gotchas
- You must remove delete permissions for ALL principals (user, SYSTEM, Administrators) on the Temp folder — missing one means the executable can still clean up after itself.
- The temp batch file gets a random name each execution — don't look for a specific filename, just check what's new in the Temp folder after running.
- In x64dbg, uncheck all breakpoint options except **Exit Breakpoint** — otherwise you'll land inside DLL initialization code and waste time stepping through irrelevant instructions.
- The memory-mapped region of interest has type MAP (not IMG or PRV) and -RW-- protection — there may be multiple regions, so confirm with the `MZ` header in the dump view before exporting.
- `strings` output on the dump reveals `.NETFramework,Version=v4.0` — this confirms it's a .NET binary and tells you to use de4dot/dnSpy rather than Ghidra or IDA for analysis.
