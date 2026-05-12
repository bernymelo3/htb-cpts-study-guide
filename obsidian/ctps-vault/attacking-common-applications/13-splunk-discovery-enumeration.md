# Section 13 — Splunk - Discovery & Enumeration

## ID
904

## Module
Attacking Common Applications

## Kind
notes

## Title
Section 13 — Splunk - Discovery & Enumeration

## Description
Covers discovering and fingerprinting Splunk instances, understanding the free vs Enterprise licensing model that can lead to unauthenticated access, and identifying RCE vectors through built-in functionality like scripted inputs and custom apps.

## Tags
splunk, enumeration, fingerprinting, siem, default-creds, scripted-inputs

## Commands
- `sudo nmap -sV <TARGET>`
- `nmap -A -Pn <TARGET>`

## What This Section Covers
Splunk is a log analytics platform commonly used as a SIEM in enterprise environments. It's extremely prevalent on internal pentests — 92 of the Fortune 100 use it. While Splunk has relatively few exploitable CVEs historically, the real attack surface comes from weak/null authentication and built-in functionality that allows custom app deployment and scripted inputs. Splunk typically runs as `root` on Linux or `SYSTEM` on Windows, making it a high-value target for privilege escalation and lateral movement.

## Methodology
1. Discover Splunk via Nmap — the `Splunkd httpd` service runs on port 8000 (web UI) and port 8089 (REST API / management port). Both ports fingerprint clearly in service scans.
2. Browse to `https://<TARGET>:8000` (note: HTTPS, not HTTP). The Splunk version is displayed on the login page title.
3. Check default credentials: older versions default to `admin:changeme` (displayed on the login page itself). Current versions set credentials at install, but try common weak passwords: `admin`, `Welcome`, `Welcome1`, `Password123`, `changeme`.
4. Check if the instance is running Splunk Free (no authentication required). The Enterprise trial auto-converts to Free after 60 days — sysadmins frequently install a trial, forget about it, and it silently becomes an unauthenticated instance.
5. Once authenticated (or if Free), enumerate the environment: browse data, check installed apps via Splunkbase, review dashboards and reports for sensitive data, and identify the OS for payload selection.
6. RCE vectors through built-in functionality include: scripted inputs (Bash/PowerShell/Batch/Python scripts that run with Splunk's service account privileges), server-side Django apps, REST endpoints, and alerting scripts.

## Lab — Questions & Answers
| Q | Answer | Found In / Method |
|---|--------|-------------------|
| Enumerate the Splunk instance as an unauthenticated user. Submit the version number (format 1.2.3). | 8.2.2 | `nmap -A -Pn <TARGET>` shows Splunkd on port 8000, browse to `https://<TARGET>:8000` — version displayed on the home page title |

## Key Takeaways
- Splunk's real vulnerability is weak/null auth + built-in code execution features, not CVEs. Vulnerability scanners will flag many non-exploitable issues — understanding the built-in abuse path is more valuable.
- The Enterprise → Free auto-conversion after 60 days is a common misconfiguration on internal networks. Free Splunk has zero authentication — anyone on the network gets full admin access.
- Splunk ships with Python on every installation (Windows and Linux), so Python reverse shells work universally regardless of the target OS.
- Port 8089 (Splunk REST API) is a secondary indicator and can sometimes be accessed independently for enumeration or SSRF attacks (CVE-2018-11409).
- Every Splunk install is a potential pivot point: if the compromised host is a deployment server, you can push malicious apps to all hosts running Universal Forwarders.
