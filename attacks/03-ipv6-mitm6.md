# IPv6 Spoofing / mitm6 MITM Attack

**MITRE ATT&CK:** T1557.001 — Adversary-in-the-Middle; T1200 — Hardware Additions (n/a here, listed for the coercion/relay chain context)

## Overview

Most Windows networks run dual-stack with IPv6 enabled but no IPv6 DHCP/DNS
infrastructure configured — Windows still prefers IPv6 when available. `mitm6`
exploits this: it answers DHCPv6 requests, hands out an attacker-controlled
IPv6 DNS server, and clients start sending DNS/WPAD lookups through the
attacker. Combined with `ntlmrelayx`, this becomes a path to relay
authentication straight to LDAP/LDAPS on the DC — often enough for a full
domain compromise via new-computer or attribute-write abuse.

A general MITM in an AD context aims not to steal a password directly, but to
obtain or relay NTLM authentication.

## Prerequisites

- Attacker on the same L2 segment as victims and the DC.
- IPv6 enabled on clients (default on modern Windows) with no legitimate
  IPv6 DHCP/DNS in place.
- LDAP signing / channel binding not enforced on the DC (or the specific
  abuse path being used doesn't require write access blocked by it).

## Steps

### 1. Start mitm6 scoped to the target domain

```bash
sudo mitm6 -d <DOMAIN>
```

This answers DHCPv6 solicitations domain-wide, pushing the attacker as the
IPv6 DNS server to any client that asks.

### 2. Start ntlmrelayx targeting LDAPS on the DC

```bash
impacket-ntlmrelayx -6 -t ldaps://<DC_IP> -wh fakewpad.<DOMAIN> -l /tmp/loot
```

- `-6` — operate over IPv6
- `-wh` — serve a rogue WPAD file so HTTP-based lookups get relayed too
- `-l` — loot directory for anything dumped from a successful relay

### 3. Wait for a machine/user to authenticate

Any DNS query, WPAD probe, or background service that triggers outbound auth
from a domain-joined host gets relayed to LDAPS.

### 4. Review loot

A successful LDAP relay against the DC can dump the domain via
`ntlmrelayx`'s built-in ldapdomaindump, producing:

```
/tmp/loot/
├── domain_computers.html / .json / .grep
├── domain_groups.html / .json / .grep
├── domain_policy.html / .json / .grep
├── domain_trusts.html / .json / .grep
├── domain_users.html / .json / .grep
└── domain_users_by_group.html
```

This gives a full enumeration of users, groups, computers, trusts, and
policy — extremely useful recon even before any credential is cracked, and a
strong starting point for BloodHound-style path analysis.

## Key Notes

- `mitm6` + `ntlmrelayx` against LDAPS is one of the most reliable initial
  domain-recon/compromise chains in a default (non-hardened) AD environment,
  because IPv6 is on by default and rarely monitored.
- Depending on the relayed account's privileges, this can escalate further —
  e.g. relaying a privileged account's auth to LDAP with write access enables
  attribute abuse (RBCD, adding a computer account, etc.).

## Detection & Defenses

- **Disable IPv6 if unused**, or deploy a legitimate, monitored IPv6
  DHCP/DNS infrastructure so rogue DHCPv6 servers stand out.
- **Enforce LDAP signing and LDAP channel binding** on all DCs — this is the
  actual fix for the LDAP-relay half of this chain.
- Monitor for unexpected DHCPv6 Advertise/Reply traffic from non-infrastructure
  hosts.
- Disable WPAD where not in use (`Group Policy → disable WinHTTP AutoProxy`).

---

*Performed against an isolated, self-owned lab domain for training purposes only.*
