## ID
705

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 28 — Attacking Applications Connecting to Services

## Description
Extracts hardcoded database credentials from compiled binaries — using GDB/PEDA to debug an ELF executable's ODBC connection string, and dnSpy to decompile a .NET DLL assembly.

## Tags
gdb, peda, reverse-engineering, credentials, odbc, dnspy, dll, dotnet

## Commands
- ssh htb-student@<TARGET_IP>
- gdb ./octopus_checker
- set disassembly-flavor intel
- disas main
- b *0x5555555551b0
- run
- Get-FileMetaData .\MultimasterAPI.dll

## What This Section Covers
Applications connecting to backend services (databases, APIs) often embed connection strings with credentials. This section covers two techniques for extracting them: debugging a Linux ELF binary with GDB/PEDA to intercept the ODBC connection string at runtime, and decompiling a Windows .NET DLL with dnSpy to read the source code directly. Both yield plaintext credentials that can be reused for lateral movement or privilege escalation.

## Methodology
1. SSH into the target: `ssh htb-student@<TARGET_IP>` (password: `HTB_@cademy_stdnt!`)
2. Run the binary to observe behavior: `./octopus_checker` — note it attempts a SQL connection (ODBC Driver 17 for SQL Server)
3. Load in GDB: `gdb ./octopus_checker`
4. Set Intel syntax and disassemble: `set disassembly-flavor intel` then `disas main`
5. Scroll through the disassembly looking for the `call SQLDriverConnect@plt` instruction — note that string fragments in the assembly may appear reversed due to endianness
6. Set a breakpoint on the `SQLDriverConnect` address: `b *0x5555555551b0`
7. Run the program: `run` — when the breakpoint hits, the RDX register contains the full ODBC connection string with plaintext credentials
8. **For .NET DLLs:** Use `Get-FileMetaData` to confirm it's a .NET assembly, then open in dnSpy → inspect controllers for connection strings containing `UID`/`PWD` or `User ID`/`Password`

## Key Takeaways
- Two distinct approaches depending on binary type: **GDB/PEDA for ELF executables** (Linux), **dnSpy for .NET DLLs** (Windows)
- In the GDB approach, the connection string lives in the RDX register at the `SQLDriverConnect` call — format: `DRIVER={ODBC Driver 17 for SQL Server};SERVER=localhost, 1401;UID=SA;PWD=N0tS3cr3t!;`
- String fragments in disassembly may appear reversed due to endianness — don't try to manually reconstruct them, just set a breakpoint and read the register at runtime
- Setting the breakpoint on the PLT address (`b *0x5555555551b0`) works; alternatively use the function name (`b SQLDriverConnect`)
- dnSpy allows full source code viewing of .NET assemblies — look in controller classes for `SqlConnection`, `ConnectionString`, or `SqlCommand` patterns
- Always test discovered credentials for **password reuse** across other services (SSH, RDP, SMB, other databases) and consider **password spraying** against other accounts

## Gotchas
- The binary requires ODBC Driver 17 to actually connect — the credentials are still extractable even if the driver isn't installed (the breakpoint fires before the connection attempt)
- GDB-peda enhances output with colored registers/stack/code views — if peda isn't available, standard GDB still works but you'll need `info registers rdx` and `x/s $rdx` to view the string

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What credentials were found for the local database instance? | SA:N0tS3cr3t! | ODBC connection string in RDX register at SQLDriverConnect breakpoint in GDB |
