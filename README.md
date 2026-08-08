# Active Directory Attack Lab — Writeup

Personal home-lab exercise covering common Active Directory attack paths, from
network-level credential capture through domain compromise. Built and tested
in an isolated, self-owned lab environment for learning/CTF-prep purposes.

> All IPs, hostnames, usernames, and passwords in this repo are placeholders.
> See [`LAB-TOPOLOGY.md`](LAB-TOPOLOGY.md) for the placeholder legend.

## Lab Environment

| Role | OS |
|---|---|
| Domain Controller | Windows Server 2025 |
| Domain-joined workstation #1 | Windows 11 Enterprise |
| Domain-joined workstation #2 | Windows 11 Enterprise |
| Attacker box | Kali Linux |

Full topology, IP scheme, and account list (redacted): [`LAB-TOPOLOGY.md`](LAB-TOPOLOGY.md)

## Attack Chain Covered

```
Network Poisoning (LLMNR / IPv6-mitm6)
        ↓
NTLM Hash Capture / SMB Relay
        ↓
Initial Credentialed Access
        ↓
Kerberoasting (SPN abuse)
        ↓
Pass-the-Hash Lateral Movement
        ↓
Token Impersonation (post-exploitation)
        ↓
LSASS Dump → DCSync → Golden/Silver Ticket
        ↓
Domain Persistence
```

## Writeups

| # | Technique | MITRE ATT&CK | File |
|---|---|---|---|
| 1 | LLMNR Poisoning | T1557.001 | [`attacks/01-llmnr-poisoning.md`](attacks/01-llmnr-poisoning.md) |
| 2 | SMB Relay | T1557.001 | [`attacks/02-smb-relay.md`](attacks/02-smb-relay.md) |
| 3 | IPv6 / mitm6 MITM | T1557.001, T1200 | [`attacks/03-ipv6-mitm6.md`](attacks/03-ipv6-mitm6.md) |
| 4 | Kerberoasting | T1558.003 | [`attacks/04-kerberoasting.md`](attacks/04-kerberoasting.md) |
| 5 | Pass-the-Hash | T1550.002 | [`attacks/05-pass-the-hash.md`](attacks/05-pass-the-hash.md) |
| 6 | Windows Access Token Impersonation | T1134.001 | [`attacks/06-token-impersonation.md`](attacks/06-token-impersonation.md) |
| 7 | LSASS Dump → DCSync → Golden/Silver Ticket | T1003.001, T1003.006, T1558.001/.002 | [`attacks/07-lsass-dcsync-tickets.md`](attacks/07-lsass-dcsync-tickets.md) |

## Disclaimer

Every technique here was performed **only** against an isolated lab domain
that I own and control, for training purposes. None of this should be run
against systems you don't have explicit written authorization to test.
