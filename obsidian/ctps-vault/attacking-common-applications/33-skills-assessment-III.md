# Attacking Common Applications — Skills Assessment III

## ID
532

## Module
Attacking Common Applications

## Kind
lab

## Title
Skills Assessment III — .NET DLL Reverse Engineering to Extract Hardcoded MSSQL Credentials

## Description
RDP into a Windows target with provided administrator credentials, locate the `MultimasterAPI.dll` web application binary, decompile it with dnSpy, and extract a hardcoded MSSQL connection string password from the source code.

## Tags
thick-client, dll, dnspy, reverse-engineering, mssql, hardcoded-credentials

## Commands
- xfreerdp /v:<TARGET_IP> /u:administrator /p:xcyj8izxNVzhf4z /dynamic-resolution
- Navigate to C:\inetpub\wwwroot\bin\ in File Explorer
- Open C:\Tools\dnSpy\dnSpy.exe
- Drag MultimasterAPI.dll onto dnSpy

## What This Section Covers
This skills assessment focuses on thick-client and server-side application analysis — specifically, reverse engineering a compiled .NET DLL to extract hardcoded secrets. The scenario simulates a penetration test where the team has already obtained Windows administrator credentials and needs to extract a hardcoded database password from a deployed web application. This is a common real-world finding: developers embedding database connection strings (including passwords) directly in application binaries, assuming that compiled code is "safe" from inspection. .NET assemblies are trivially decompilable back to near-source-code quality using tools like dnSpy, ILSpy, or dotPeek, making this a critical vulnerability. The exercise teaches the importance of examining deployed application binaries during post-exploitation, not just configuration files.

## Methodology
1. **RDP into the target** — Connect to the Windows target using the provided administrator credentials:
   ```
   xfreerdp /v:<TARGET_IP> /u:administrator /p:xcyj8izxNVzhf4z /dynamic-resolution
   ```
   Breaking down the flags:
   - `/v:` — target IP address
   - `/u:` — username (administrator)
   - `/p:` — password
   - `/dynamic-resolution` — allows the RDP window to resize dynamically, making it easier to work with the GUI

2. **Locate the web application binary** — Open File Explorer and navigate to `C:\inetpub\wwwroot\bin\`. This is the default deployment directory for ASP.NET/IIS web applications. The `bin` folder contains compiled .NET assemblies (DLLs) that make up the application. Here you'll find `MultimasterAPI.dll` — the main application binary for what appears to be an internal API service.

   Why `C:\inetpub\wwwroot\bin\`? IIS (Internet Information Services) serves web applications from `C:\inetpub\wwwroot\` by default. The `bin\` subdirectory is where ASP.NET compiles and stores application assemblies. During a pentest, this is one of the first directories to check on any IIS server for hardcoded secrets in application code.

3. **Open dnSpy for decompilation** — Open a second File Explorer window and navigate to `C:\Tools\dnSpy\`. Launch `dnSpy.exe`. dnSpy is a .NET debugger and assembly editor that can decompile .NET assemblies (DLLs and EXEs) back into readable C# or VB.NET source code. It's the go-to tool for .NET reverse engineering because it provides a full IDE-like interface with navigation, search, and debugging capabilities.

4. **Load and analyze the DLL** — Drag `MultimasterAPI.dll` from the File Explorer window onto the dnSpy window. dnSpy will decompile the assembly and display the namespace/class hierarchy in the left panel. Browse through the decompiled classes, looking for:
   - Database connection strings
   - Hardcoded credentials
   - API keys or tokens
   - Configuration constants

   In the decompiled source, locate the SQL connection string which contains the hardcoded password: `D3veL0pM3nT!`. This is the MSSQL service password embedded directly in the application code.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| What is the hardcoded password for the database connection in MultimasterAPI.dll? | D3veL0pM3nT! | RDP in → navigate to C:\inetpub\wwwroot\bin\ → drag MultimasterAPI.dll into dnSpy → find SQL connection string in decompiled source |

## Key Takeaways
- **Compiled ≠ secure** — .NET assemblies (DLLs/EXEs) are trivially decompilable back to near-source-code quality; developers who assume compiled code hides secrets are wrong; any attacker with file system access can extract them in minutes
- **Always check `C:\inetpub\wwwroot\bin\` on IIS servers** — this is the standard deployment directory for ASP.NET applications and frequently contains DLLs with hardcoded secrets; it should be part of every Windows post-exploitation checklist
- **dnSpy is the primary tool for .NET reverse engineering** — it provides decompilation, debugging, and editing in a single tool; alternatives include ILSpy (lighter, open-source) and JetBrains dotPeek (free, read-only); all three can extract secrets from .NET assemblies
- **Connection strings are the #1 finding in .NET DLLs** — developers routinely hardcode database connection strings with embedded credentials rather than using Windows Authentication, Azure Key Vault, or externalized configuration; look for `SqlConnection`, `ConnectionString`, `Server=`, `Password=`, `Data Source=` in decompiled output
- **This finding enables lateral movement** — a database password like `D3veL0pM3nT!` doesn't just give you database access; it might be reused across other services, and the MSSQL server itself can be used for command execution via `xp_cmdshell` or credential extraction
- **Post-exploitation isn't just about dumping hashes** — examining deployed application code, configuration files, scripts, and scheduled tasks often yields credentials and secrets that open new attack paths; always check web roots, service binaries, and script directories

## Gotchas
- **dnSpy must be on the target already** — in this lab it's pre-installed at `C:\Tools\dnSpy\`; in a real engagement you'd either need to upload it (risky from an opsec perspective) or copy the DLL back to your machine and decompile locally, which is the preferred approach
- **Don't overlook the `bin` folder** — pentesters sometimes check `web.config` for connection strings but skip the compiled assemblies; even if `web.config` uses encrypted connection strings or references external key stores, the DLL itself may still contain a hardcoded fallback or development password
- **The password uses leetspeak substitution (`D3veL0pM3nT!`)** — this is a common pattern for "development" or "staging" passwords that were never rotated before production deployment; flag these as findings because they suggest weak credential management practices
- **RDP with `/dynamic-resolution`** — without this flag, the RDP session may open in a tiny fixed resolution that makes navigating File Explorer and dnSpy painful; always include it for GUI-heavy work
