# Agar.io Protocol v23

**Date:** February 13, 2026  
**Author:** @LDAP.SH  
**Server version tested:** 25.5.2  
**Client:** `agario.core.wasm` (C/C++ compiled to WebAssembly via Emscripten)

---

## Table of Contents

1. [Overview](#1-overview)
2. [Client Architecture](#2-client-architecture)
3. [WebSocket Connection](#3-websocket-connection)
4. [Encryption System (The Breakthrough)](#4-encryption-system-the-breakthrough)
5. [Protocol : Client → Server](#5-protocol--client--server)
6. [Protocol : Server → Client](#6-protocol--server--client)
7. [World Update Packet Parsing](#7-world-update-packet-parsing)
8. [WASM Integrity System](#8-wasm-integrity-system)
9. [Anti-Bot Protections](#9-anti-bot-protections)

---

## 1. Overview

This fully reverse was made using the help of ClaudeAI reflexion's.<br>
The official Agar.io client uses a binary protocol over WebSocket with **three layers of protection**:

```
┌──────────────────────────────────────────────────────┐
│  Layer 3: Cloudflare (TLS fingerprinting, 403/400)   │
├──────────────────────────────────────────────────────┤
│  Layer 2: XOR cipher (4-byte cycling key = DKS ⊕ EKS)│
│           + LZ4 compression on every packet          │
├──────────────────────────────────────────────────────┤
│  Layer 1: Standard v13 opcodes inside                │
│           (WorldUpdate, Border, Ping, etc.)           │
└──────────────────────────────────────────────────────┘
```

**Key discovery:** Protocol v23 wraps the legacy v13 binary format inside a combined encryption + compression layer. Once decrypted and decompressed, the inner packets are identical to the well-documented v13 protocol.

---

## 2. Client Architecture

The Agar.io client is **NOT** obfuscated JavaScript. It is C/C++ compiled to **WebAssembly** via **Emscripten**.

```
┌────────────────────────────────────────────────────────┐
│  BROWSER                                               │
│                                                        │
│  ┌──────────────┐     ┌──────────────────────────────┐ │
│  │ agar.io HTML  │────>│ agario.core.js (Emscripten)  │ │
│  │ + MC object   │     │                              │ │
│  └──────────────┘     │  ┌──────────────────────────┐ │ │
│                        │  │ agario.core.wasm (~306KB) │ │ │
│                        │  │                          │ │ │
│                        │  │ Contains:                │ │ │
│                        │  │  • Network protocol      │ │ │
│                        │  │  • Integrity checks      │ │ │
│                        │  │  • 2D Canvas rendering   │ │ │
│                        │  │  • Game logic            │ │ │
│                        │  │  • Packet parsing        │ │ │
│                        │  └──────────────────────────┘ │ │
│                        └──────────────────────────────┘ │
└────────────────────────────────────────────────────────┘
```

### Exported WASM Functions

| WASM Function | JS Wrapper | Purpose |
|---|---|---|
| `_ac_connect(ptr)` | `core.connect(url)` | Connect to server |
| `_ac_disconnect()` | `core.disconnect()` | Disconnect |
| `_ac_set_player_name(ptr)` | `core.sendNick(name)` | Set nickname |
| `_ac_set_mouse_position(x,y)` | `core.setTarget(x,y)` | Mouse movement |
| `_ac_split()` | `core.split()` | Split cell |
| `_ac_eject()` | `core.eject()` | Eject mass |
| `_ac_spectate()` | `core.sendSpectate()` | Spectate mode |
| `_ac_zoom(v)` | `core.playerZoom(v)` | Zoom |
| `_ac_set_integrity_checks(v)` | `core.disableIntegrityChecks(!v)` | Toggle integrity |
| `_ac_send_facebook_data(ptr)` | `core.sendFacebookData(str)` | Facebook auth |
| `_ac_get_game_state()` | `core.getGameState()` | Current state |
| `_ac_player_has_cells()` | `core.playerHasCells()` | Is player alive |

### WebSocket Bridge

The WebSocket lives in JS-land, not WASM. The Emscripten glue code uses a `cp5.sockets[]` array with an event queue system:

- **Event 1** = MESSAGE (data copied to WASM heap via `_malloc` + `writeArrayToMemory`)
- **Event 2** = OPEN
- **Event 3** = ERROR
- **Event 4** = CLOSE

WASM polls events via `_cp5_check_ws()` and sends data by writing to its heap then calling `ws.send(HEAP8.subarray(...))`.

### MC Object Callbacks

The WASM communicates back to the front-end JS via `window.MC`:

| Callback | When |
|---|---|
| `MC.onAgarioCoreLoaded()` | Core initialized |
| `MC.onPlayerSpawn()` | Player spawned |
| `MC.onPlayerDeath(...)` | Player died (9 params) |
| `MC.onDisconnect()` | Disconnected |
| `MC.onConnect()` | Connected |
| `MC.getHost()` | Get server URL |
| `MC.showOutdatedClientDialog()` | Client too old (packet 0x80) |
| `MC.updateServerVersion(v)` | Server version received |

---

## 3. WebSocket Connection

### Handshake Sequence

```
Client                              Server
  │                                    │
  │──── WSS connect ──────────────────>│  (Cloudflare TLS)
  │<─── 101 Switching Protocols ──────│
  │                                    │
  │──── 0xFE [u32 proto=23] ─────────>│  Establish
  │──── 0xFF [i64 EKS] ─────────────>│  ConnectionKey
  │──── 0x00 [name\0] ───────────────>│  Spawn
  │                                    │
  │<─── 0xF1 [i32 DKS][ver\0] ──────│  DKS2 (cleartext)
  │<─── encrypted stream ────────────│  All subsequent packets
  │                                    │
  │──── 0x10 [x][y][integrity] ─────>│  SetTarget (~50ms)
  │<─── 0x10 WorldUpdate ────────────│  (encrypted+compressed)
```

### Required HTTP Headers

```
Host: <server-hostname>
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36
            (KHTML, like Gecko) Chrome/131.0.0.0 Safari/537.36
Origin: https://agar.io
Cache-Control: no-cache
Pragma: no-cache
Accept-Language: en-US,en;q=0.9
Accept-Encoding: gzip, deflate, br, zstd
Sec-WebSocket-Extensions: permessage-deflate; client_max_window_bits
```

Missing or incorrect headers cause Cloudflare to reject with 400 or 403.

### ConnectionKey Format Change (v13 → v23)

```python
# v13: struct.pack("<Bi", 0xFF, eks_value)    → 5 bytes (i32)
# v23: struct.pack("<Bq", 0xFF, eks_value)    → 9 bytes (i64)
```

---

## 4. Encryption System (The Breakthrough)

This was the hardest part to crack. The server encrypts ALL packets after sending `0xF1` (DKS2).

### The Key

The XOR key is **NOT** just DKS. It is the **XOR of DKS and EKS**:

```
XOR_KEY = DKS ⊕ EKS
```

Where:
- **EKS** = `0x00007999` — static, hardcoded in WASM as obfuscated XOR chain (func 341)
- **DKS** = 4-byte value received from server in packet `0xF1` — changes per session

Example from a live session:
```
DKS received:   0xB7CFB688
EKS hardcoded:  0x00007999
XOR_KEY:         0xB7CFCF11  →  bytes: [11, CF, CF, B7] (little-endian)
```

### Full Decryption Pipeline

Every server packet after `0xF1` must be processed as follows:

```
Raw packet (on wire)
        │
        ▼
   XOR decrypt with 4-byte cycling key (DKS ⊕ EKS)
        │
        ▼
   0xFF compressed wrapper
   [0xFF][u32 uncompressed_size][LZ4 data...]
        │
        ▼
   LZ4 decompress
        │
        ▼
   Standard v13 opcode packet
   [opcode][payload...]
```

```python
def decrypt_server_packet(data, xor_key):
    result = bytearray(len(data))
    for i in range(len(data)):
        result[i] = data[i] ^ xor_key[i & 3]
    return bytes(result)

# After decryption: first byte is always 0xFF
# Next 4 bytes = uncompressed size (u32 LE)
# Remaining = LZ4 compressed payload
decrypted = decrypt_server_packet(raw, xor_key)
assert decrypted[0] == 0xFF
uncomp_size = struct.unpack_from("<I", decrypted, 1)[0]
inner = lz4.block.decompress(decrypted[5:], uncompressed_size=uncomp_size)
real_opcode = inner[0]  # 0x10, 0x40, 0x20, 0xE2, etc.
```

### Verified Results from Live Session

| Raw encrypted | After XOR + LZ4 | Inner opcode | Parsed content |
|---|---|---|---|
| `0xEE...` (93B) | 135B inner | `0x10` WorldUpdate | 3 cells with names ">.<", valid coords |
| `0xEE...` (41B) | 34B inner | `0x40` Border | `[-4503, -4500] → [9638, 9641]` |
| `0xEE...` (61B) | 54B inner | `0x10` WorldUpdate | Position updates, 3 cells moving |
| `0xEE...` (80B) | 73B inner | `0x10` WorldUpdate | New food cell spawned (flags2=0x01) |
| `0xF3...` (3B) | — | `0xE2` Ping | 2 bytes echo data |

### Exceptions (Not Encrypted)

- `0xF1` (DKS2) — sent in cleartext since it provides the key itself

### How It Was Discovered

1. XOR with DKS alone → opcode `0x66` for most packets (unknown, nothing parsed)
2. XOR with **DKS ⊕ EKS** → opcode `0xFF` (compressed wrapper)
3. LZ4 decompression succeeded → standard v13 opcodes with valid data
4. WorldUpdate cells had reasonable coordinates (1968, 2457), realistic radii (1361), actual player names

---

## 5. Protocol — Client → Server

All client packets are currently sent **unencrypted** (integrity checks disabled on official servers). When integrity is enabled, packets go through MurmurHash3 rolling XOR (see Section 8).

| Opcode | Name | Format | Size |
|---|---|---|---|
| `0x00` | Spawn | `[0x00][name\0]` | Variable |
| `0x01` | Spectate | `[0x01]` | 1 |
| `0x10` | SetTarget | `[0x10][i32 x][i32 y][i32 integrity]` | 13 |
| `0x11` | Split | `[0x11]` | 1 |
| `0x15` | Eject | `[0x15]` | 1 |
| `0xE3` | Pong | `[0xE3][echo_data]` | Variable |
| `0xFE` | Establish | `[0xFE][u32 protocol_version]` | 5 |
| `0xFF` | ConnectionKey | `[0xFF][i64 eks_value]` | 9 |

### SetTarget Integrity Field

- When integrity disabled: `integrity = 0`
- When integrity enabled: `integrity = last DKS value` (read from WASM address 36208)

---

## 6. Protocol — Server → Client

After decryption + LZ4 decompression, inner packets use standard v13 opcodes:

| Opcode | Name | Description |
|---|---|---|
| `0x10` | WorldUpdate | Cell positions, sizes, eat events, deletions |
| `0x11` | Viewport | Camera position/zoom |
| `0x12` | Reset | Clear all cells |
| `0x20` | OwnedCell | Your cell ID (`u32`) |
| `0x32` | LeaderboardRGB | Team mode leaderboard |
| `0x35` | LeaderboardList | FFA leaderboard |
| `0x36` | LeaderboardListAlt | Leaderboard with friend indicators |
| `0x40` | Border | Map boundaries (`4× f64`) + gamemode (`u32`) + server name |
| `0x55` | CaptchaRequest | reCAPTCHA required |
| `0x67` | LoggedIn | Facebook auth confirmed |
| `0x80` | Outdated | Client version too old |
| `0xA1` | RemoveArrow | Remove UI arrow |
| `0xE2` | Ping | Server keepalive |
| `0xF1` | DKS2 | Encryption key + server version string (**CLEARTEXT**) |

### New v23 Opcodes (Not Yet Fully Decoded)

| Opcode | Size | Frequency | Notes |
|---|---|---|---|
| `0x45` | ~262B | ~2/sec | Likely extended leaderboard or world state |
| `0xF2` | 5B | Once at start | Handshake acknowledgement? |

---

## 7. World Update Packet Parsing

`0x10` WorldUpdate is the most complex and frequent packet:

```
[0x10]
[u16 eat_count]
  for each eat:
    [u32 eater_id][u32 victim_id]       ← 8 bytes per eat event
[cell updates until sentinel u32 == 0]
  for each cell:
    [u32 cell_id]                        ← 0 = end of cell list
    [i32 x][i32 y]                       ← world coordinates
    [u16 radius]                         ← cell size
    [u8 flags]                           ← see below
    if flags & 0x80:  [u8 flags2]        ← extended flags
    if flags & 0x02:  [u8 r][u8 g][u8 b] ← RGB color
    if flags & 0x04:  [utf8 skin\0]      ← skin URL
    if flags & 0x08:  [utf8 name\0]      ← player name
    if flags2 & 0x04: [u32 account_id]   ← logged-in account
[u16 delete_count]
  for each delete:
    [u32 cell_id]                        ← removed cell
```

### Cell Flags (byte 1)

| Bit | Hex | Meaning |
|---|---|---|
| 0 | `0x01` | Virus |
| 1 | `0x02` | Has color (RGB follows) |
| 2 | `0x04` | Has skin (string follows) |
| 3 | `0x08` | Has name (string follows) |
| 4 | `0x10` | Agitated |
| 5 | `0x20` | Ejected mass |
| 6 | `0x40` | Other eject |
| 7 | `0x80` | Extended flags byte follows |

### Cell Flags2 (byte 2, only if flags & 0x80)

| Bit | Hex | Meaning |
|---|---|---|
| 0 | `0x01` | Food pellet |
| 1 | `0x02` | Friend |
| 2 | `0x04` | Has account ID (u32 follows) |

### Border Packet (0x40)

```
[0x40]
[f64 left][f64 top][f64 right][f64 bottom]
[u32 gamemode]    ← optional
[utf8 server\0]   ← optional
```

Typical: `[-7071, -7071, 7071, 7071]` for standard FFA.

### Mass Calculation

```
mass = (radius * radius) / 100
```

---

## 8. WASM Integrity System

Fully reverse-engineered from `agario.core.wasm` disassembly (306KB binary, 705 functions, 130K lines WAT).

### Key WASM Memory Addresses

| Address | Size | Purpose |
|---|---|---|
| `27182` | 1 byte | Integrity ON/OFF flag |
| `36184` | struct | Connection object base |
| `36189` | 1 byte | Client encryption toggle (always 0 = off) |
| `36208` | 4 bytes | DKS value from server (0xF1 packet) |
| `36212` | 4 bytes | MurmurHash3 rolling key state (conn_obj + 28) |
| `37336` | 4 bytes | Mouse X position |
| `37340` | 4 bytes | Mouse Y position |

### Key WASM Functions

| Function | Export Name | Purpose |
|---|---|---|
| func 341 | — | Compute EKS constant (returns `0x00007999`) |
| func 611 | `_ac_set_integrity_checks` | Set flag at address 27182 |
| func 609 | `_ac_set_mouse_position` | Store x,y at 37336/37340 |
| func 217 | — | Build SetTarget (0x10) packet with integrity field |
| func 62 | — | MurmurHash3 XOR encryption + WebSocket send |
| func 353 | — | Main network tick loop, builds ConnectionKey |
| func 148 | — | Initialize connection object, set seed=0 |

### EKS Computation (func 341)

EKS is a compile-time constant hidden behind an obfuscated 5-way XOR:

```python
c0 = 0xE85E696E
c1 = 0x5FB9F97F
c2 = 0x35A506C8
c3 = 0xF717DD0B
c4 = 0xDB611D6C ^ 0xAE342F27  # = 0x7555324B

EKS = c0 ^ c1 ^ c2 ^ c3 ^ c4 = 0x00007999
```

### Client-Side Encryption Algorithm (func 62)

When integrity is enabled, outgoing packets are XOR-encrypted with a MurmurHash3 rolling key:

```python
def murmur_mix(key):
    C = 0x5BD1E995   # MurmurHash2 constant
    SEED = 0x06D0FF67
    key = (key * C) & 0xFFFFFFFF
    key = ((key >> 24) ^ key) & 0xFFFFFFFF
    key = (key * C) & 0xFFFFFFFF
    key = (key ^ SEED) & 0xFFFFFFFF
    key = ((key >> 13) ^ key) & 0xFFFFFFFF
    key = (key * C) & 0xFFFFFFFF
    key = ((key >> 15) ^ key) & 0xFFFFFFFF
    return key

# Usage:
rolling_key = 0  # initial seed (from func 148 init)
for packet in outgoing_packets:
    old_key = rolling_key
    rolling_key = murmur_mix(old_key)
    key_bytes = struct.pack("<I", old_key)
    encrypted = bytes(b ^ key_bytes[i & 3] for i, b in enumerate(data))
    ws.send(encrypted)
```

### MurmurHash3 Constants

| Constant | Value | Purpose |
|---|---|---|
| Multiplier | `0x5BD1E995` (1540483477) | MurmurHash2 mixing |
| Seed | `0x06D0FF67` (114296087) | XOR after multiply |
| Shift 1 | 24 | First right-shift |
| Shift 2 | 13 | Second right-shift |
| Shift 3 | 15 | Third right-shift |
| Initial key | `0x00000000` | From func 148 (`i64.const 673720360` stores `0x28282828` at 36208 and `0x00000000` at 36212) |

### DKS Initial Value

The DKS field at address 36208 is initialized to `0x28282828` by func 148. This default is overwritten when the server sends the `0xF1` packet with the session-specific DKS value.

### Encryption Toggle

Address `36189` controls whether client-side encryption is active. In all observed WASM builds, this flag is **always 0** (disabled). The JS call `core.disableIntegrityChecks(true)` sets address `27182` to 0, which skips the entire encryption branch in func 62.

---

## 9. Anti-Bot Protections

### Cloudflare TLS Fingerprinting

- WebSocket upgrade requests are fingerprinted via JA3/JA4
- Python's default TLS stack gets rejected with 400/403
- **Mitigation:** `tls-client` library for Chrome JA3 fingerprint spoofing, or residential proxies

### reCAPTCHA

- Server sends `0x55` (CaptchaRequest) for suspicious IPs
- Datacenter/VPS IPs trigger CAPTCHA almost immediately
- **Mitigation:** Residential IP or CAPTCHA solving service (2captcha, etc.)

### Protocol Version Enforcement

- Wrong version in Establish packet → server sends `0x80` (Outdated) and disconnects
- Must send exactly `23`

### Integrity Verification

- Server CAN verify the integrity field in SetTarget matches the DKS value
- Server CAN verify MurmurHash3 client-side encryption
- **Current observation:** sending `0` for integrity works — server does not currently enforce

---

### Complete Constants Reference

```
Protocol version:        23
Server version:          25.5.2

EKS (static):            0x00007999 (31129)
DKS (initial default):   0x28282828 (673720360)
DKS (per-session):       received in packet 0xF1
Server XOR key:          DKS ^ EKS  (4 bytes, cycling)

MurmurHash multiplier:   0x5BD1E995 (1540483477)
MurmurHash XOR seed:     0x06D0FF67 (114296087)
MurmurHash shifts:       24, 13, 15
Rolling key initial:     0x00000000

WASM addresses:
  Integrity flag:        27182
  Encryption toggle:     36189
  Connection object:     36184
  DKS storage:           36208
  Rolling key state:     36212
  Mouse X:               37336
  Mouse Y:               37340
```

---

## Appendix A — Packet Hex Examples

### DKS2 Packet (cleartext, on wire)
```
F1 88 B6 CF B7 32 35 2E 35 2E 32 00
│  │           │                    │
│  └ DKS i32   └ "25.5.2\0"        │
└ opcode                            └ null terminator
```

### Encrypted WorldUpdate (on wire → decrypted → decompressed)
```
On wire (93 bytes):
  EE 48 CF B7 11 30 E3 A7 11 CF A3 13 F7 56 ...

After XOR with key [11 CF CF B7]:
  FF 87 00 00 ...  (0xFF = compressed wrapper, size=135)

After LZ4 decompression (135 bytes):
  10 00 00 6C A4 E6 99 B0 07 00 00 99 09 00 00 ...
  │  │     │           │        │        │
  │  └u16  └u32 cellID └i32 x   └i32 y   └u16 radius
  │  eat=0
  └ 0x10 WorldUpdate

Parsed: cell 2582029420 at (1968, 2457) r=1361 name=">.<"
```
