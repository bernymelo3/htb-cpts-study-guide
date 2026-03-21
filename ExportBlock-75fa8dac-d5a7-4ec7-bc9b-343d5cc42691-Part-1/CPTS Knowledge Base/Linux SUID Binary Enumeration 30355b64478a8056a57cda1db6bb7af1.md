# Linux SUID Binary Enumeration

Category: Privilege Escalation
Command: find / -perm -4000 -type f 2>/dev/null
Description: Searches the entire filesystem for files with the SUID bit set. SUID binaries run with the permissions of their owner (often root), making them prime targets for privilege escalation. Review GTFOBins for exploitation techniques for any discovered binaries.
Tags: enumeration, privilege-escalation, suid