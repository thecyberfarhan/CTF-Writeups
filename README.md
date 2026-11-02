# CTF Writeups

Hands-on penetration testing writeups from TryHackMe and Hack The Box. Each one follows the same structure I would use in a report: how I found the target, how I got in, how I escalated, and how the issue should be fixed.

**Focus areas:** web application security, privilege escalation (Linux and Windows), network services, password attacks and vulnerability research.

## At a glance

| Platform | Writeups | Difficulty mix |
|---|---|---|
| TryHackMe | 32 | 22 Easy, 10 Medium |
| Hack The Box | 5 | 4 Very Easy (Starting Point), 1 Easy |
| **Total** | **37** | |

## How each writeup is structured

1. **TL;DR:** the attack chain in a few sentences.
2. **Recon and foothold:** scans, enumeration and the first weakness.
3. **Shell and escalation:** how access turned into root or SYSTEM.
4. **Rabbit holes:** dead ends and why they were dead ends.
5. **Mitigations:** how to fix each issue found.
6. **Key takeaway and "If I did it again":** what I learned and what I would do faster.

## Hack The Box

| Machine | Difficulty | What it covered |
|---|---|---|
| [Archetype](HackTheBox/archetype_writeup.md) | Very Easy | An SMB `backups` share was accessible without credentials and contained a SSIS `.dtsConfig` file with plaintext SQL Server credentials. |
| [Cap](HackTheBox/cap_writeup.md) | Easy | The web app ran a "Security Dashboard" that let users trigger network captures. The capture results were stored at `/data/<id>` with no access controls — navigating to `/data/0`… |
| [Oopsie](HackTheBox/oopsie_writeup.md) | Very Easy | Apache box running a repair management web app. Guest login is available but the upload feature is locked behind an authorization check that trusts client-supplied cookies. |
| [Unified](HackTheBox/unified_writeup.md) | Very Easy | UniFi Network Application 6.4.54 running on port 8443 is vulnerable to Log4Shell (CVE-2021-44228). |
| [Vaccine](HackTheBox/vaccine_writeup.md) | Very Easy | Three services, three layers to peel. Anonymous FTP hands you a password-protected ZIP. |

## TryHackMe

| Machine | Difficulty | What it covered |
|---|---|---|
| [0day](TryHackme/0day_writeup.md) | Medium | A web server leaking an encrypted SSH private key via an exposed `/backup/` directory, combined with a Shellshock-vulnerable CGI endpoint, gave initial access as `www-data`. |
| [Anonforce](TryHackme/anonforce_writeup.md) | Easy | Anonymous FTP served the entire server filesystem. The user flag was readable directly from `/home/melodias/`. |
| [Anonymous](TryHackme/anonymous_writeup.md) | Easy | A world-writable FTP `scripts/` directory contained a cleanup script that runs on a schedule, plus a to-do note literally saying "I really need to disable the anonymous login."… |
| [Basic Pentesting](TryHackme/basicpentesting_writeup.md) | Easy | A web developer's changelog mentioned auditing `/etc/shadow` and easily cracking a colleague's password — itself a hint about the box's weak password culture. |
| [Billing](TryHackme/billing_writeup.md) | Medium | MagnusBilling's `icepay.php` accepts a `democ` parameter that gets passed unsanitised into a shell context — a single GET request with a chained `curl \| bash` payload landed a… |
| [Blog](TryHackme/blog_writeup.md) | Medium | WPScan fingered WordPress 5.0 and enumerated two users. Hydra cracked `kwheel:cutiepie1`. |
| [Boiler CTF](TryHackme/boilerctf_writeup.md) | Medium | A multi-layer decoding puzzle in `robots.txt` (ASCII → Base64 → MD5, cracked to `kidding`) turned out to be a taunt rather than a credential. |
| [Bounty Hacker](TryHackme/bountyhacker_writeup.md) | Easy | Anonymous FTP served up a username (`lin` from `task.txt`) and a custom password wordlist (`locks.txt`). |
| [Brooklyn Nine-Nine](TryHackme/brooklynninenine_writeup.md) | Easy | An anonymous FTP login hands over a note hinting that Jake's password is weak, and the site's HTML source drops a not-so-subtle "have you heard of steganography?" comment pointing… |
| [Brute It](TryHackme/bruteit_writeup.md) | Easy | A username left in an HTML comment led to a Hydra brute force of the admin panel. The panel served an encrypted SSH private key for user `john`; John the Ripper cracked the… |
| [Chill Hack](TryHackme/chillhack_writeup.md) | Medium | A command execution panel filtered most shell commands but missed `find`, which was abused via `-exec` to spawn a Python reverse shell. |
| [Cyber Heroes](TryHackme/cyberheros_writeup.md) | Easy | The entire challenge lives in a single JavaScript `authenticate()` function left fully exposed in the browser's page source. |
| [Cyborg](TryHackme/cyborg_writeup.md) | Easy | A publicly exposed Squid proxy config and password file handed over an md5crypt hash cracked to `squidward`. |
| [Fowsniff CTF](TryHackme/fowsniff_writeup.md) | Easy | OSINT on the company's GitHub revealed a posted credential dump with 9 MD5 hashes — all cracked in seconds. |
| [Ignite](TryHackme/ignite_writeup.md) | Easy | The homepage announces "Welcome to FUEL CMS" directly in the title — no version fingerprinting needed before pulling Fuel CMS exploits from Exploit-DB. |
| [Lian Yu](TryHackme/lianyu_writeup.md) | Medium | A layered reconnaissance trail: a white-text hidden word on a web page pointed to a directory, an HTML comment hinted at a `.ticket` file extension, and a Feroxbuster scan with… |
| [Mr Robot CTF](TryHackme/mrrobot_writeup.md) | Medium | `robots.txt` leaked both key 1 and a custom wordlist. Deduplication shrunk the wordlist from 858K to 11K words before running Hydra in two passes — first for the username… |
| [Mustacchio](TryHackme/mustacchio_writeup.md) | Easy | A SQLite backup file exposed in a public JS directory yielded an admin SHA1 hash cracked to `bulldog19`. |
| [Neighbour](TryHackme/neighbour_writeup.md) | Easy | A PHP login page left guest credentials in an HTML comment. Logging in as `guest` revealed a profile page controlled by a `?user=` URL parameter with zero server-side… |
| [Overpass](TryHackme/overpass_writeup.md) | Easy | Overpass serves a Go-based "password manager" on port 80. `gobuster` finds `/admin`, whose login is gated entirely in the browser — `login.js` stores whatever the API returns as a… |
| [Pickle Rick](TryHackme/picklerick_writeup.md) | Easy | Rick left his username in an HTML comment and his password in `robots.txt`. Logging into the web portal revealed a command execution panel — with `cat` blocked but `sudo less`… |
| [Publisher](TryHackme/publisher_writeup.md) | Easy | Pre-authenticated as `think` via SSH key. A SUID binary called `run_container` shells out to `/opt/run_container.sh` — writable with root privileges during execution. |
| [Red](TryHackme/red_writeup.md) | Medium | An LFI in a "free business template" PHP app leaks its own source code via `php://filter`, exposing a single-pass sanitization flaw that's trivially bypassed. |
| [RootMe](TryHackme/rootme_writeup.md) | Easy | A file upload panel at `/panel/` blocked `.php` but accepted `.phtml`, landing a `www-data` shell. |
| [Silver Platter](TryHackme/silverplatter_writeup.md) | Medium | A contact form named a Silverpeas username. CeWL wordlist plus Hydra cracked the password. |
| [Simple CTF](TryHackme/simplectf_writeup.md) | Easy | Anonymous FTP exposed a note revealing username `mitch`. A CMS Made Simple 2.2.8 installation on the web server was vulnerable to CVE-2019-9053 (unauthenticated SQL injection)… |
| [Source](TryHackme/source_writeup.md) | Easy | Nmap fingerprinned Webmin 1.890 on port 10000. That version contains a supply-chain backdoor (CVE-2019-15107) introduced into the official download. |
| [Team](TryHackme/team_writeup.md) | Easy | `robots.txt` gave a username, a stale script backup exposed FTP credentials, FTP yielded a subdomain hint, and that subdomain ran a PHP page vulnerable to LFI. |
| [tomghost](TryHackme/tomghost_writeup.md) | Easy | Tomcat 9.0.30 with AJP exposed on port 8009 is vulnerable to Ghostcat (CVE-2020-1938) — an unauthenticated file read via the AJP protocol. |
| [w1seGuy](TryHackme/w1seguy_writeup.md) | Easy | The server XORs a hardcoded fake flag (`THM{thisisafakeflag}`) with a random 5-character key and hands you the hex-encoded result. |
| [Watcher](TryHackme/watcher_writeup.md) | Medium | Six escalation hops from `www-data` to root, each building on the previous: LFI read FTP credentials → FTP shell upload → `www-data` → `sudo php` as `toby` → writable cron script… |
| [Wgel CTF](TryHackme/wgel_writeup.md) | Easy | A username leaked in an HTML comment, an SSH private key sitting in a publicly browsable `.ssh` directory under `/sitemap/`, and a `sudo wget` rule that lets Jessie exfiltrate any… |

## Notes

All writeups cover retired machines or training rooms on legal lab platforms. No real systems or client data are involved.
