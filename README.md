# 🔒 Zero-Trace Terminal

> End-to-end encrypted P2P chat, right in your browser. No server, no accounts, no stored history.

**by Hardlint Cybersecurity Team**

---

## ⚠️ Important Notice

This project is distributed **for educational and security research purposes**. It does not guarantee network-level anonymity: it protects the **content** of conversations, not necessarily **who** is connecting. Read the [Known Limitations](#-known-limitations) section before using it for sensitive communications.

**Use of a trustworthy VPN on both devices is strongly recommended.**

---

## ✨ Features

- 🔐 **End-to-end encryption** — AES-GCM 256-bit, key derived via PBKDF2 (100,000 iterations)
- 🌐 **True P2P connection** — direct WebRTC link between the two devices, no central server relaying messages
- 🚫 **Zero persistence** — no cookies, no localStorage, no database: close the tab and nothing remains
- 🔑 **Single shared secret** — a randomly generated Room Key (100 characters), no manual technical configuration required
- 🧹 **Panic Purge** — one button instantly wipes keys, connection state, and visible chat history
- 📡 **Reliable connectivity** — 18 STUN/TURN servers configured as fallbacks to work even behind restrictive NATs (4G/5G, corporate networks)

---

## 🚀 How to Use

1. **Open the page** (must be served over HTTPS — e.g. via GitHub Pages, not opened as a local file)
2. **Host:** click `[1] INITIALIZE ROOM` → copy the generated Room Key
3. Send the Room Key to your contact **through a different channel** (in person, voice call, another encrypted app)
4. **Guest:** click `[2] CONNECT TO ROOM` → paste the received Room Key
5. Wait for the connection (usually a few seconds) → the chat opens
6. If the connection isn't established within 2 minutes, the Room Key expires automatically: generate a new one with the dedicated button

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| Cryptography | Web Crypto API (AES-GCM, PBKDF2, SHA-256) |
| P2P | WebRTC via [PeerJS](https://peerjs.com/) |
| NAT Traversal | STUN + TURN (18 configured endpoints) |
| Frontend | Plain HTML/CSS/JS, no framework |
| Hosting | Static (GitHub Pages) |

No proprietary backend: the entire logic runs in the browser.

---

## 📋 Requirements

- Modern browser with WebRTC and Web Crypto API support (recent Chrome, Firefox, Edge, Safari)
- Internet access on both devices
- The page **must** be served over HTTPS (Secure Context is required for Web Crypto API and WebRTC) — it does not work when opened as a local file

---

## ⚠️ Known Limitations

This project protects the **content** of messages (real end-to-end encryption, verifiable by reading the code), but **does not hide connection metadata**:

- The hosting service (GitHub Pages), the signaling broker (PeerJS), and the TURN server can see the **IP addresses** and **timestamps** of both devices connecting
- They **never see the message content**, which stays end-to-end encrypted at every stage
- To reduce metadata exposure, using a trustworthy VPN is recommended

For the full technical breakdown (cryptographic model, attack surface, detailed architecture), see [`DOCUMENTAZIONE_TECNICA.md`](./DOCUMENTAZIONE_TECNICA.md).

---

## 📜 License and Disclaimer

This software is provided "as is", without warranties of any kind. The developers are not responsible for any improper or illegal use of this tool. Users are solely responsible for complying with applicable laws in their jurisdiction.

---

<div align="center">

**Hardlint Cybersecurity Team**

</div>
