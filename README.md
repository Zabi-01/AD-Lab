# Active Directory Attack Lab

A structured, end-to-end writeup of common Active Directory attack
techniques, demonstrated in a self-hosted, isolated lab environment —
from unauthenticated network position through to full domain
compromise and persistence.

**Environment:** Isolated, self-owned virtual lab (no production systems involved)

> All IPs and passwords in this repository are placeholders/redacted.
> See [`LAB-TOPOLOGY.md`](LAB-TOPOLOGY.md) for the placeholder legend, and
> [`RUN-LOG.md`](RUN-LOG.md) for the real domain, hostnames, and usernames
> with attack order and outcomes (secrets masked).

---

## Objective

This repository documents a complete Active Directory attack chain,
executed and verified in a controlled lab, and written up in the style of
a professional penetration test report: methodology, exact commands,
interpretation of results, and corresponding defensive guidance for each
stage. The goal is to build a working, reproducible reference for AD
attack paths rather than a loose collection of commands.

## Lab Environment

| Role | OS | Notes |
|---|---|---|
| Domain Controller | Windows Server 2025 | AD DS, DNS, holds `krbtgt` |
| Domain-joined workstation #1 | Windows 11 Enterprise | Standard-user session, no local admin for the attacker account |
| Domain-joined workstation #2 | Windows 11 Enterprise | Secondary target, used to demonstrate lateral movement and pivot scope |
| Attacker host | Kali Linux | Runs Responder, the Impacket suite, mitm6, Hashcat, and Metasploit/Mimikatz payloads |

All four hosts sit on a single isolated virtual segment with no route to
any network outside the hypervisor. This is a deliberate design choice —
several stages (LLMNR poisoning, mitm6) rely on shared broadcast/multicast
scope with the victims and are contained entirely within the lab.

- Full topology, IP scheme, and account list (redacted): [`LAB-TOPOLOGY.md`](LAB-TOPOLOGY.md)
- Real domain, hostnames, and usernames, with attack order and outcomes (secrets masked): [`RUN-LOG.md`](RUN-LOG.md)

## Tooling

| Tool | Purpose |
|---|---|
| [Responder](https://github.com/lgandx/Responder) | LLMNR/NBT-NS poisoning, rogue SMB/HTTP authentication capture |
| [Impacket](https://github.com/fortra/impacket) suite (`GetUserSPNs`, `psexec`, `ntlmrelayx`, `wmiexec`) | Kerberoasting, relay, remote execution |
| [CrackMapExec](https://github.com/Porchetta-Industries/CrackMapExec) | SMB credential spraying, SAM dumping, pass-the-hash |
| [mitm6](https://github.com/dirkjanm/mitm6) | Rogue DHCPv6 / IPv6 DNS takeover |
| [Hashcat](https://hashcat.net/hashcat/) | Offline hash cracking (NetNTLMv2, Kerberos encryption types) |
| [Mimikatz](https://github.com/gentilkiwi/mimikatz) / [pypykatz](https://github.com/skelsec/pypykatz) | LSASS credential extraction, DCSync, ticket forging |
| Metasploit (Incognito extension) | Token enumeration and impersonation, post-exploitation |
| Nmap (`smb2-security-mode.nse`) | SMB signing enumeration |

## Attack Chain

```
Network Poisoning (LLMNR / IPv6-mitm6)
        ↓
NTLM Hash Capture / SMB Relay
        ↓
Initial Credentialed Access
        ↓
Kerberoasting (SPN Abuse)
        ↓
Pass-the-Hash Lateral Movement
        ↓
Token Impersonation (Post-Exploitation)
        ↓
LSASS Dump → DCSync → Golden/Silver Ticket
        ↓
Domain Persistence
```

## Writeups

| # | Technique | MITRE ATT&CK | Privilege Required | Document |
|---|---|---|---|---|
| 1 | LLMNR Poisoning | T1557.001 | None (same L2 segment) | [`attacks/01-llmnr-poisoning.md`](attacks/01-llmnr-poisoning.md) |
| 2 | SMB Relay | T1557.001 | None (relay target lacks SMB signing) | [`attacks/02-smb-relay.md`](attacks/02-smb-relay.md) |
| 3 | IPv6 / mitm6 MITM | T1557.001 | None (IPv6 enabled, unmanaged) | [`attacks/03-ipv6-mitm6.md`](attacks/03-ipv6-mitm6.md) |
| 4 | Kerberoasting | T1558.003 | Any valid domain account | [`attacks/04-kerberoasting.md`](attacks/04-kerberoasting.md) |
| 5 | Pass-the-Hash | T1550.002 | Valid password or NTLM hash | [`attacks/05-pass-the-hash.md`](attacks/05-pass-the-hash.md) |
| 6 | Windows Access Token Impersonation | T1134.001 | Local SYSTEM on a host with a cached privileged token | [`attacks/06-token-impersonation.md`](attacks/06-token-impersonation.md) |
| 7 | LSASS Dump → DCSync → Golden/Silver Ticket | T1003.001, T1003.006, T1550.002, T1558.001, T1558.002 | Local admin (LSASS) → Domain Admin / replication rights (DCSync) | [`attacks/07-lsass-dcsync-tickets.md`](attacks/07-lsass-dcsync-tickets.md) |

Each writeup follows a consistent structure — **Overview → Prerequisites →
Step-by-Step → Key Notes/Gotchas → Detection & Defenses → Quick
Reference** — so any stage can be read independently without walking the
full chain first.

## How the Stages Connect

- **Stages 1–3** establish credential access without prior credentials:
  they move an attacker from zero to a first valid domain credential (or
  a live relayed session) using network position alone.
- **Stage 4** converts that initial low-privilege credential into a shot
  at a service account with potentially greater privilege, using ordinary,
  authorized-looking Kerberos traffic.
- **Stage 5** tests whether a recovered credential is reused elsewhere in
  the environment — the pivot from a single host to many.
- **Stage 6** is post-exploitation on an already-compromised host: once
  local SYSTEM is reached, it looks for higher-value identities already
  resident in memory rather than harvesting further credentials from the
  wire.
- **Stage 7** is the endgame. Once Domain Admin (or equivalent
  replication rights) is reached by any of the preceding paths, DCSync
  and forged tickets convert that momentary access into durable,
  detection-resistant control of the domain.

## Repository Structure

```
active-directory-attack-lab/
├── README.md              — this file
├── LAB-TOPOLOGY.md         — network diagram and placeholder legend
├── RUN-LOG.md              — real domain/hostnames/usernames, attack order, secrets masked
├── .gitignore              — excludes raw hash/dump/loot artifacts
└── attacks/
    ├── 01-llmnr-poisoning.md
    ├── 02-smb-relay.md
    ├── 03-ipv6-mitm6.md
    ├── 04-kerberoasting.md
    ├── 05-pass-the-hash.md
    ├── 06-token-impersonation.md
    └── 07-lsass-dcsync-tickets.md
```

## Disclaimer

Every technique documented here was performed exclusively against an
isolated lab domain owned and controlled by the author, for training
purposes. None of this material should be applied to any system without
explicit, written authorization to test.
