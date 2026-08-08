# Windows Access Token Impersonation

**MITRE ATT&CK:** T1134.001 — Access Token Manipulation: Token Impersonation/Theft

## Overview

Token impersonation abuses Windows' security token model to act as a
different, more privileged user without ever knowing their password or
hash. When a user authenticates to a machine — interactively, via RDP, via a
scheduled task, or over the network — Windows caches an access token
representing that identity in memory. An attacker with SYSTEM (or another
sufficiently privileged context) on that machine can locate tokens
belonging to *other* users still resident in memory and impersonate them,
inheriting their privileges with no credential theft in the traditional
sense.

Commonly executed via Metasploit's **Incognito** extension.

## Prerequisites

- Code execution on the target host.
- **SYSTEM-level privileges**, or at minimum `SeImpersonatePrivilege` /
  `SeAssignPrimaryTokenPrivilege`. Without SYSTEM, token enumeration is
  typically restricted to the current user's own tokens.
- A target machine where a higher-privileged account (local admin, domain
  admin, service account) has an active or cached logon session.

## Steps

### 1. Establish access and confirm privilege level

```
getuid
```

If not `NT AUTHORITY\SYSTEM`:

```
getsystem
```

SYSTEM is required for Incognito to reliably enumerate other users' tokens.

### 2. Load the Incognito extension

```
load incognito
```

### 3. Enumerate available tokens

```
list_tokens -u
```

Two categories:
- **Delegation tokens** — full-fidelity, usable for both local and network
  authentication (from interactive logons — console, RDP). Most valuable.
- **Impersonation tokens** — usable only for local authentication on that
  machine (e.g. from a network logon like SMB). Cannot authenticate onward
  to other hosts.

Look for domain accounts indicating admin-level access.

### 4. Impersonate a target token

```
impersonate_token <DOMAIN>\\<TARGET_USER>
```

Note the **double backslash** — Metasploit's console treats a single
backslash as an escape character.

### 5. Verify

```
getuid
```

Should now report the impersonated identity instead of SYSTEM.

### 6. Use the impersonated context

- Drop a shell running under the impersonated token: `shell`
- Move laterally to other hosts the impersonated user has access to
- Read files/shares requiring the impersonated user's permissions

### 7. Revert when finished

```
rev2self
```

## Key Limitations Observed in Practice

- **Impersonation ≠ full privilege inheritance for every action.** Certain
  SYSTEM-only operations (e.g. direct SAM parsing via `hashdump`) can still
  fail with access-denied or error 1168 — those specific API calls depend on
  the process's actual owning context and specific privileges (e.g.
  `SeBackupPrivilege`), which impersonation alone doesn't grant.
- **Delegation vs. impersonation tokens matter** for lateral movement
  planning — impersonation-level tokens can't authenticate onward.
- **No useful token ≠ dead end** — other paths (LSASS dumping, SAM/SYSTEM
  hive extraction, Kerberoasting) may still be viable.
- **Token availability is session-dependent** — a privileged user's token
  only appears if they have an active/recent session on that host.
  Re-checking `list_tokens` after triggering an admin login (scheduled
  task, coercion technique, routine maintenance) can surface new tokens.

## Detection & Defensive Notes

- **Event ID 4624** with Logon Type 9 (`NewCredentials`) or unusual Logon
  Type 3/10 combinations from one host in a short window can indicate
  token duplication activity.
- **Sysmon Event ID 10** (ProcessAccess) targeting `lsass.exe` is a strong
  indicator.
- Restrict `SeImpersonatePrivilege` / `SeAssignPrimaryTokenPrivilege` to
  only accounts/services that truly need them.
- Avoid leaving high-privilege interactive sessions open on lower-tier
  hosts.
- Enforce **Credential Guard** — isolates LSASS secrets in a
  virtualization-based security container, significantly limiting classic
  token/credential theft.

## Quick Reference

| Step | Command |
|---|---|
| Check current identity | `getuid` |
| Escalate to SYSTEM | `getsystem` |
| Load extension | `load incognito` |
| List tokens | `list_tokens -u` / `list_tokens -g` |
| Impersonate | `impersonate_token DOMAIN\\User` |
| Verify | `getuid` |
| Revert | `rev2self` |

---

*Performed against an isolated, self-owned lab domain for training purposes only.*
