# Zero-Trace Terminal — Technical Documentation

**Project:** Hardlint Cybersecurity Team
**Type:** Browser-based, end-to-end encrypted P2P chat
**Last updated:** September 2026

---

## 1. Architectural Overview

Zero-Trace Terminal is a static web application (HTML/CSS/JS, no proprietary backend) that allows two devices to establish a **direct peer-to-peer connection** via WebRTC, exchanging end-to-end encrypted text messages.

**Main components:**

| Component | Role | Technology |
|---|---|---|
| User interface | Retro terminal UI | Plain HTML/CSS |
| Signaling | Makes the two peers "find" each other | PeerJS (public cloud broker) |
| Data transport | Encrypted P2P channel | WebRTC DataChannel |
| NAT traversal | Punching through firewalls/NAT | STUN + TURN (ICE) |
| Encryption | Message content protection | AES-GCM 256-bit + PBKDF2 |
| Hosting | Code distribution | GitHub Pages (static) |

There is no proprietary application server: the code runs entirely in each user's browser. The only external infrastructure involved is used to "introduce" the two devices to each other (signaling) and, if needed, to relay traffic when a direct connection isn't possible (TURN).

---

## 2. Operational Flow

### 2.1 Room Key Generation

When a user clicks **"INITIALIZE ROOM (HOST)"**:

1. A random **100-character** string is generated (`generate100CharCode()`), using `crypto.getRandomValues()` — a cryptographically secure random number generator (not `Math.random()`, which is unsuitable for cryptographic purposes).
2. The character set includes uppercase/lowercase letters, digits, and special symbols (`-_!@#$%^&*`), maximizing entropy within 100 characters.

This string (the **Room Key**) is the only shared secret the two parties need to exchange, out-of-band (e.g. voice message, in person, another encrypted channel).

### 2.2 Deriving Keys from the Room Key

Two independent values, each with a different purpose, are derived from the Room Key:

**A. Message encryption key (PBKDF2 → AES-GCM)**
```
PBKDF2(
  password = Room Key,
  salt = "p2p-zero-trace-salt-v1" (fixed, hardcoded),
  iterations = 100,000,
  hash = SHA-256
) → 256-bit AES-GCM key
```

**B. PeerJS identifier (truncated SHA-256)**
```
SHA-256(Room Key) → first 32 hex characters, prefixed with "ztt-"
```
This ID is used solely so that Host and Guest can "find" each other on the PeerJS signaling broker, without exchanging anything beyond the Room Key. It plays no cryptographic role.

> **Note on the fixed salt:** the PBKDF2 salt is hardcoded and identical across all sessions. This is acceptable because the "password" (Room Key) already has very high entropy (100 random characters) — a fixed salt only weakens security in scenarios involving weak, reused passwords, which does not apply here.

### 2.3 Signaling Phase (PeerJS)

1. The Host creates a `Peer` object, registering with the public PeerJS cloud broker using the ID derived from the Room Key.
2. The Guest, after pasting the same Room Key, computes the same ID and calls `peer.connect(id)`.
3. The PeerJS broker only mediates this initial exchange (who wants to talk to whom) — **it never sees or transmits message content**, which by that point travels over a separate WebRTC channel.

### 2.4 ICE Negotiation (NAT Traversal)

Once the two Peers have "introduced" themselves, WebRTC starts ICE negotiation to find a valid network path:

1. **Host candidates** — the device's local IP addresses
2. **Server-reflexive (srflx) candidates** — public IP discovered via STUN
3. **Relay candidates** — allocated via TURN, used only if a direct connection fails

**Configured ICE servers (in priority order):**

- **Dedicated Metered.ca TURN** (own credentials, not shared) — `stun.relay.metered.ca` / `global.relay.metered.ca`
- 7 public STUN fallbacks (Google ×3, Cloudflare, Twilio, Nextcloud, stunprotocol.org, freestun)
- 10 additional public TURN fallback endpoints (OpenRelay, freestun, numb.viagenie, ExpressTurn) — used only if the dedicated TURN also fails

The browser automatically tries every combination and selects the first one that establishes a working channel (standard ICE algorithm, handled internally by WebRTC).

### 2.5 Timeout and Session Expiry

- If the connection isn't established within **120 seconds**, the session is considered expired:
  - The `Peer` and `DataConnection` are destroyed (`peer.destroy()`, `conn.close()`)
  - The status shows `[EXPIRED] Room Key no longer valid`
  - A button appears to generate a new Room Key (Host) or enter a new one (Guest)
- This prevents a Room Key from remaining "listening" indefinitely on the public broker.

---

## 3. Message Cryptographic Model

Every message is individually encrypted before being sent over the DataChannel:

```
1. Generate a random 12-byte IV (crypto.getRandomValues)
2. ciphertext = AES-GCM-Encrypt(key, IV, plaintext)
3. payload = IV || ciphertext   (concatenated, IV in plaintext at the front)
4. Send payload as a Uint8Array via conn.send()
```

On receipt:
```
1. Extract the first 12 bytes as the IV
2. The rest is the ciphertext (includes the 16-byte GCM authentication tag at the end)
3. plaintext = AES-GCM-Decrypt(key, IV, ciphertext)
```

**Security properties guaranteed by AES-GCM:**
- **Confidentiality** — nobody without the key can read the content
- **Integrity/authenticity** — any tampering with the packet in transit causes decryption to fail (`[ERR: DECRYPTION_FAILED]`), rather than silently producing corrupted output

**What this scheme does NOT cover:**
- **Forward secrecy across sessions** — if the same Room Key were reused across multiple sessions (not the normal flow, which generates a new one every time), all those sessions would share the same derived key
- **Peer identity authentication** — anyone who knows the Room Key can connect; there is no cryptographic verification of "who" is on the other end beyond possession of the shared key

---

## 4. Data Persistence (Client-Side)

| Data | Persistence | Notes |
|---|---|---|
| Room Key | None | Only in a JS variable, gone on close/reload |
| Derived AES-GCM key | None | Same, never written to disk |
| Chat messages | None | Only live in the DOM/RAM, no localStorage/IndexedDB |
| Cookies | None | The project uses none at all |
| Application logs | Local DevTools console only | Never sent anywhere, gone when the tab closes |

The **"PANIC: PURGE SESSION"** button explicitly forces:
- Closure of the `PeerConnection`/`DataConnection`
- Zeroing of the encryption key in memory
- Wiping of all UI fields and the displayed message history

---

## 5. What External Infrastructure Can See (Metadata)

Key point to understand: **encryption protects content, not connection metadata**.

| Service | What it can see | What it CANNOT see |
|---|---|---|
| **GitHub Pages** | IP and timestamp of whoever loads the page | Message content, Room Key |
| **PeerJS broker** (public cloud) | IP of Host and Guest, when they connect, their Peer ID (a hash of the Room Key, not the Key itself) | Message content |
| **Metered.ca TURN** (if used as relay) | Source/destination IP, ports, amount of data transferred | Message content (already travels encrypted) |
| **Each user's ISP** | That a connection is being made to github.io / metered.ca / a PeerJS server | Message content |

**Recommended mitigation (outside the code):** use a trustworthy VPN (e.g. Mullvad, with an anonymously created account) on both devices, to avoid exposing real IP addresses to these third-party services. The project displays an explicit warning to this effect on the splash screen.

---

## 6. Attack Surface and Known Limitations

| Risk | Description | Mitigated? |
|---|---|---|
| Message content interception | MITM on TURN/network traffic | ✅ Yes — end-to-end AES-GCM |
| Message tampering in transit | Packet manipulation | ✅ Yes — GCM authentication tag |
| Deanonymization via IP metadata | IP↔identity correlation through third-party logs | ⚠️ Partial — requires a VPN client-side, not solved by the code itself |
| Metered.ca account compromise | TURN credentials are in the public code (base64-obfuscated, not encrypted) | ⚠️ Minimal deterrent, not real security |
| Dependency on third-party services | GitHub Pages, PeerJS broker, Metered TURN — if suspended, the app stops working | ⚠️ Not mitigated (would require full self-hosting) |
| Room Key reuse | Would compromise forward secrecy across sessions that reuse it | ✅ Not applicable in normal flow (a new Key every session) |

---

## 7. Full Technology Stack

- **Frontend:** HTML5, CSS3 (no framework)
- **Cryptography:** Browser-native Web Crypto API (`crypto.subtle`) — PBKDF2, AES-GCM, SHA-256
- **P2P/Signaling:** [PeerJS](https://peerjs.com/) v1.5.4 (a wrapper library over native WebRTC), loaded from a public CDN (unpkg.com)
- **NAT Traversal:** WebRTC ICE (STUN/TURN) — 18 endpoints configured in total
- **Hosting:** GitHub Pages (static, automatic HTTPS)
- **Browser requirements:** WebRTC support, Web Crypto API, ES6+ — requires a secure context (HTTPS); does not work from `file://` or `content://`

---

## 8. Requirements for Correct Operation

- Both devices need **internet access** (to reach the PeerJS broker and ICE servers)
- The page must be served over **HTTPS** (GitHub Pages guarantees this automatically) — it **will not work** if opened as a local file (`content://`, `file://`) due to Secure Context restrictions on the Web Crypto API and WebRTC
- Both parties must have the page open **at the same time** during the connection attempt (signaling is not persistent/asynchronous)
- The Room Key must be copied **in full, exactly 100 characters**, with no extra spaces or line breaks introduced by copy-paste

---

## 9. Changelog of Major Versions

1. **v1** — Native WebRTC with manual SDP exchange (copy/paste offer/answer)
2. **v2** — Migrated to PeerJS, automatic connection based on the Room Key, removed manual SDP fields
3. **v3** — Added dedicated Metered.ca TURN + multiple public STUN/TURN fallbacks
4. **v4** — Detailed ICE diagnostics (candidate logging, connection states) — fixed a bug that overwrote PeerJS's internal event handlers
5. **v5** — Timeout extended to 120s, Room Key expiry system with manual regeneration, VPN warning on splash screen, TURN credential obfuscation

---

*This document is provided for internal technical documentation purposes only. It does not constitute legal advice regarding regulatory compliance, privacy, or liability for use.*
