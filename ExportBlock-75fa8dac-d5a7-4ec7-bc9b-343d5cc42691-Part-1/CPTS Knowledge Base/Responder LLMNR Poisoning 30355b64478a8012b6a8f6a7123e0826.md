# Responder LLMNR Poisoning

Category: Active Directory
Command: responder -I <interface> -wf
Description: Captures NTLM hashes by poisoning LLMNR, NBT-NS, and MDNS requests on the network. Use in Active Directory environments to capture credentials when users mistype share names or when automatic authentication occurs. Flags: -w starts WPAD rogue server, -f forces NTLM/Basic authentication.
Tags: active-directory, enumeration, responder