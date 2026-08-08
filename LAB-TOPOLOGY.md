# Lab Topology & Placeholder Legend

## Topology

```
                 ┌─────────────────────────┐
                 │   Domain Controller      │
                 │   Windows Server 2025    │
                 │   <DC_HOSTNAME>          │
                 │   IP: <DC_IP>            │
                 └────────────┬─────────────┘
                              │
              ┌───────────────┴────────────────┐
              │                                 │
   ┌──────────▼─────────┐           ┌───────────▼──────────┐
   │  Workstation 1      │           │  Workstation 2         │
   │  Windows 11 Ent.     │           │  Windows 11 Ent.       │
   │  <WS1_HOSTNAME>      │           │  <WS2_HOSTNAME>        │
   │  IP: <WS1_IP>        │           │  IP: <WS2_IP>          │
   └──────────────────────┘           └────────────────────────┘

   ┌──────────────────────┐
   │  Attacker Box          │
   │  Kali Linux            │
   │  IP: <ATTACKER_IP>     │
   └────────────────────────┘

Domain: <DOMAIN>  (e.g. corp.local)
```

## Placeholder Legend

Use this table to substitute your real lab values back in when reproducing
steps locally. **Never commit the filled-in version of this table.**

| Placeholder | Meaning |
|---|---|
| `<DOMAIN>` | AD domain name |
| `<DC_HOSTNAME>` / `<DC_IP>` | Domain Controller name / IP |
| `<WS1_HOSTNAME>` / `<WS1_IP>` | Workstation 1 name / IP |
| `<WS2_HOSTNAME>` / `<WS2_IP>` | Workstation 2 name / IP |
| `<ATTACKER_IP>` | Kali attacker IP |
| `<LOWPRIV_USER>` / `<LOWPRIV_PASS>` | Initial low-privilege domain account |
| `<ADMIN_USER>` / `<ADMIN_PASS>` | Domain admin account |
| `<SVC_ACCOUNT>` / `<SVC_PASS>` | SPN-registered service account |
| `<LOCAL_USER>` | Local account found via SAM dump |
| `<NTLM_HASH>` | Any captured/cracked NTLM hash |
| `<SID>` | Domain SID |
| `<KRBTGT_HASH>` | krbtgt account hash (Golden Ticket material) |

## Accounts Used in This Lab (redacted)

| Account | Role | Notes |
|---|---|---|
| `<LOWPRIV_USER>` | Standard domain user | Initial foothold credential |
| `<SVC_ACCOUNT>` | SPN-registered service account | Target of Kerberoasting |
| `<ADMIN_USER>` | Domain Admin | End goal / high-value target |
| `<LOCAL_USER>` | Local account on WS2 | Recovered from SAM dump |

## Notes on Redaction

- Real IPs used during testing were on a private `192.168.x.0/24` range not
  reachable outside the lab hypervisor network.
- All passwords shown in original captures were lab-only test values, never
  reused elsewhere, and are excluded from this repo.
- Ticket/hash blobs (`$krb5tgs$...`, NTLM `HHHHH...` strings) are omitted or
  truncated in the writeups — they're artifacts of a specific run and aren't
  needed to understand or reproduce the methodology.
