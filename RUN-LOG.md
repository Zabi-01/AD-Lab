# Run Log — Attack Order & Findings

Real domain/hostnames/usernames from the actual lab run, for traceability
against the individual writeups in `attacks/`. **Passwords, hashes, and
tickets are masked** — only their format/length is shown. IPs use the
placeholders from [`LAB-TOPOLOGY.md`](LAB-TOPOLOGY.md).

## Environment

| Role | Hostname | IP |
|---|---|---|
| Domain Controller | `OBS-DC01` | `<DC_IP>` |
| Workstation (gateway-facing box) | `ROOTWIN` | `<WS1_IP>` |
| Workstation (secondary target) | `ROOTWIN-Y` | `<WS2_IP>` |
| Attacker (Kali) | — | `<ATTACKER_IP>` |

Domain: **`obs.corp`**

## Accounts Encountered

| Account | Type | Password / Secret | Notes |
|---|---|---|---|
| `Zabi` | Domain user | `[REDACTED]` | Initial foothold — captured via LLMNR poisoning |
| `Tony Stark` | Domain Admin | `[REDACTED]` | High-value target |
| `SQLService` | Domain service account (SPN-registered) | `[REDACTED]` | Kerberoasted; SPN `OBS-DC01/SQLService.obs.corp:<PORT>` |
| `mharoon` | Domain user + local acct on `ROOTWIN-Y` | `[REDACTED — NetNTLMv2 + local NT hash]` | Captured via Responder; local hash recovered from SAM dump |
| `Masab` | Local account on `ROOTWIN-Y` | `[REDACTED — local NT hash]` | From SAM dump |
| `Administrator` (local, `ROOTWIN-Y`) | Local built-in | `[REDACTED]` | Observed in SAM dump |

## Attack Order & Outcome

| # | Stage | Target(s) | Result |
|---|---|---|---|
| 1 | LLMNR poisoning ([`01`](attacks/01-llmnr-poisoning.md)) | `ROOTWIN-Y` users | Captured + cracked `Zabi`'s NetNTLMv2 hash |
| 2 | Credential validation / spray | Whole `/24` | `Zabi` valid on `OBS-DC01` and `ROOTWIN-Y` (local admin on the latter) |
| 3 | SAM dump | `ROOTWIN-Y` | Local hashes for `Administrator`, `Guest`, `DefaultAccount`, `WDAGUtilityAccount`, `Masab`, `mharoon` |
| 4 | Pass-the-hash ([`05`](attacks/05-pass-the-hash.md)) | `ROOTWIN-Y` | Authenticated as `mharoon` using the local NT hash directly, no cracking |
| 5 | Kerberoasting ([`04`](attacks/04-kerberoasting.md)) | `SQLService` SPN | AES256 (`$18$`) ticket cracked → `SQLService` plaintext recovered |
| 6 | mitm6 / IPv6 relay ([`03`](attacks/03-ipv6-mitm6.md)) | `OBS-DC01` LDAPS | Relay succeeded; full domain enumeration dumped (users/groups/computers/trusts/policy) |
| 7 | SMB relay ([`02`](attacks/02-smb-relay.md)) | `ROOTWIN-Y` | Confirmed signing disabled, relayed a captured hash, authenticated with resulting NT hash |
| 8 | Token impersonation ([`06`](attacks/06-token-impersonation.md)) | Compromised host | SYSTEM → Incognito → impersonated a privileged domain token |
| 9 | LSASS → DCSync → Golden Ticket ([`07`](attacks/07-lsass-dcsync-tickets.md)) | `OBS-DC01` | Dumped LSASS, DCSync'd `Administrator`, forged a Golden Ticket with `krbtgt` |

## Notes to Self

- `ROOTWIN-Y` was the weak link throughout — no SMB signing, local-admin
  password reuse of the domain low-priv account, and where the SAM
  dump/PtH chain actually paid off. Worth hardening that box specifically
  for round 2.
- `ROOTWIN` never got fully compromised this run — SMB connection kept
  timing out. Follow up separately.
- `SQLService`'s SPN was registered manually for this lab via `setspn -a`
  — in a real environment this would already exist from a prior
  deployment, worth keeping in mind for the framing in `07`.

---

*Domain `obs.corp` is an isolated, self-owned lab environment. Secrets
above are intentionally masked; see your own local notes for the real
values if you need to reproduce a step.*
