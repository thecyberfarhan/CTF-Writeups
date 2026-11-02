<p align="center">
  <img src="assets/hero.svg" alt="CTF Writeups: 37 writeups from TryHackMe and Hack The Box" width="880" height="250" />
</p>

Every writeup here is written the way I would hand it over in an engagement: how I found the target, how I got in, how I escalated, how the issue should be fixed, and what I would do faster next time. Platforms are TryHackMe and Hack The Box. All targets are retired machines or training rooms on legal lab platforms.

<p align="center">
  <img src="assets/stats.svg" alt="At a glance: 37 writeups, difficulty mix and writing activity" width="880" height="300" />
</p>

<p align="center">
  <img src="assets/topics.svg" alt="Topics covered across the writeups" width="880" />
</p>

## Start here

If you only read three, read these. Each one shows a different part of how I work.

- **[Mr Robot CTF](TryHackme/mrrobot_writeup.md)**: a wordlist cut from 858,160 entries to 11,451 made Hydra 75 times faster, with no loss of coverage.
- **[Mustacchio](TryHackme/mustacchio_writeup.md)**: XXE on an XML endpoint that turns the parser into a file reader.
- **[Archetype](HackTheBox/archetype_writeup.md)**: a Windows foothold through plaintext credentials in an SSIS config file and PowerShell history.

## What every writeup contains

| Section | What you get |
|---|---|
| **TL;DR** | The full attack chain in a few sentences. |
| **Recon and foothold** | Scans, enumeration and the first weakness found. |
| **Shell and escalation** | How access became root or SYSTEM. |
| **Rabbit holes** | Dead ends, and why they were dead ends. |
| **Mitigations** | How each issue should be fixed. |
| **Key takeaway** | The one lesson worth keeping, plus "if I did it again". |

## Hack The Box

| Machine | Level | Key takeaway |
|---|---|---|
| [Archetype](HackTheBox/archetype_writeup.md) | <img src="assets/lvl-very-easy.svg" alt="Very Easy" /> | `ConsoleHost_history.txt` is the Windows equivalent of `.bash_history` — and people are far less likely to think about clearing it. |
| [Cap](HackTheBox/cap_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Linux capabilities are quieter than SUID bits and just as dangerous. |
| [Oopsie](HackTheBox/oopsie_writeup.md) | <img src="assets/lvl-very-easy.svg" alt="Very Easy" /> | Authorization in web applications has to live on the server — never in a cookie the user controls. |
| [Unified](HackTheBox/unified_writeup.md) | <img src="assets/lvl-very-easy.svg" alt="Very Easy" /> | Log4Shell is the cleanest example of why logging user-controlled input without sanitization is dangerous: the exploit isn't in the application logic, it's in the logging… |
| [Vaccine](HackTheBox/vaccine_writeup.md) | <img src="assets/lvl-very-easy.svg" alt="Very Easy" /> | This box is a clean demonstration of credential chaining: each service exposes just enough information to compromise the next one. |

## TryHackMe

| Machine | Level | Key takeaway |
|---|---|---|
| [0day](TryHackme/0day_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | A `cgi-bin` directory on any server running a pre-2014 Bash is a Shellshock waiting to happen — always pair directory enumeration results with a vulnerability scan… |
| [Anonforce](TryHackme/anonforce_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Anonymous FTP is almost never appropriate for production systems — but it's especially catastrophic when the FTP root is the server's filesystem root. |
| [Anonymous](TryHackme/anonymous_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | A growing log file with repeated identical entries is direct evidence of a scheduled task — and a writable script anywhere in that execution chain is a privilege… |
| [Basic Pentesting](TryHackme/basicpentesting_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | A security audit that identifies a weak password but doesn't enforce remediation, combined with home directory permissions that let other local users traverse into… |
| [Billing](TryHackme/billing_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | Granting `sudo` access to a tool's *client* binary is not the same as granting access to only its "safe" read operations — `fail2ban-client`'s `set` subcommand can… |
| [Blog](TryHackme/blog_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | Environment variables are entirely user-controlled — any user can set `admin=1` before running a command. |
| [Boiler CTF](TryHackme/boilerctf_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | Not every encoded string or elaborate puzzle is a critical clue — sometimes the box is testing whether you'll waste twenty minutes decoding a joke instead of moving on… |
| [Bounty Hacker](TryHackme/bountyhacker_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | GTFOBins is not just a list of fun tricks — it's a mandatory pre-flight checklist before granting `sudo` to any binary. |
| [Brooklyn Nine-Nine](TryHackme/brooklynninenine_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | A correct password attached to a guessed username is indistinguishable from a wrong password until you actually check `/etc/passwd` or `/home` — verify the account name… |
| [Brute It](TryHackme/bruteit_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | `sudo cat` is not a safe alternative to `sudo vim` or `sudo bash` — the ability to read any file as root is all an attacker needs to extract `/etc/shadow`… |
| [Chill Hack](TryHackme/chillhack_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | Access to the Docker socket is equivalent to root access on the host — there is no meaningful distinction. |
| [Cyber Heroes](TryHackme/cyberheros_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Anything that runs in the browser is readable by the user — no amount of string reversal, obfuscation, or "obscure" path names changes that. |
| [Cyborg](TryHackme/cyborg_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Credential reuse across services is a force multiplier for attackers — one cracked hash unlocked both the Squid proxy and the BorgBackup repository. |
| [Fowsniff CTF](TryHackme/fowsniff_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | OSINT isn't just a pre-engagement formality — sometimes it hands you the entire credential set before you've touched a single port. |
| [Ignite](TryHackme/ignite_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | When a discovered credential fails against `sudo`, try it against `su` before discarding it — the two commands check entirely different passwords (the caller's vs. |
| [Lian Yu](TryHackme/lianyu_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | Enumeration isn't just running the same tools with the same flags every time — Lian Yu's key step was adding `.ticket` to Feroxbuster's extension list based on a clue… |
| [Mr Robot CTF](TryHackme/mrrobot_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | Always deduplicate custom wordlists before feeding them to Hydra — 858,160 entries shrunk to 11,451 is a 75× speed improvement for zero information loss. |
| [Mustacchio](TryHackme/mustacchio_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | XXE is not just an academic vulnerability — when a web application parses XML and reflects the output, a single malicious DOCTYPE declaration can turn the XML parser… |
| [Neighbour](TryHackme/neighbour_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | A session cookie proves you're *logged in* — it doesn't automatically prove you're allowed to access every resource on the server. |
| [Overpass](TryHackme/overpass_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Broken authentication doesn't need to be subtle to be devastating — here it was "the server trusts a cookie it never issued." Read client-side JavaScript *before*… |
| [Pickle Rick](TryHackme/picklerick_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | A command filter is only as strong as its list is complete — blocking `cat` while leaving `less`, `more`, `tail`, `strings`, `grep`, `awk`, and a hundred other… |
| [Publisher](TryHackme/publisher_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | AppArmor profiles are only as strong as their coverage — a profile that restricts `ash` but doesn't account for Perl, Python, or other interpreters available on the… |
| [Red](TryHackme/red_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | A sanitization function that only runs once is functionally the same as no sanitization at all — always test traversal filters by nesting the very sequence they claim to… |
| [RootMe](TryHackme/rootme_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | When reviewing SUID binaries, the question isn't "do I recognise this binary" — it's "should this binary *ever* have SUID set?" Interpreters, scripting tools, and… |
| [Silver Platter](TryHackme/silverplatter_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | Passwords passed as command-line arguments are logged everywhere — `sudo` audit logs, shell history, `ps` output, and `/proc`. |
| [Simple CTF](TryHackme/simplectf_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Granting `sudo` access to any program that can run arbitrary commands — editors, interpreters, pagers, file transfer tools — is functionally equivalent to granting… |
| [Source](TryHackme/source_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Supply-chain attacks are particularly insidious because they subvert trust at the source — downloading software from the official website is no longer sufficient if the… |
| [Team](TryHackme/team_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Virtual host enumeration is an underused but essential step — the entire attack surface of this box was hidden behind a hostname that wouldn't resolve without an… |
| [tomghost](TryHackme/tomghost_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | Always check exposed protocols, not just the headline application version — Ghostcat is a connector-level vulnerability that exists independently of which Tomcat version… |
| [w1seGuy](TryHackme/w1seguy_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | XOR is not encryption — it is a bitwise operation. |
| [Watcher](TryHackme/watcher_writeup.md) | <img src="assets/lvl-medium.svg" alt="Medium" /> | Python's import resolution searches the script's local directory before system paths. |
| [Wgel CTF](TryHackme/wgel_writeup.md) | <img src="assets/lvl-easy.svg" alt="Easy" /> | `sudo wget` is not "just a download tool" — `--post-file` turns it into a root-powered file exfiltration primitive. |

---

<p align="center">
  <sub>More coming. Profile: <a href="https://github.com/thecyberfarhan">github.com/thecyberfarhan</a></sub>
</p>
