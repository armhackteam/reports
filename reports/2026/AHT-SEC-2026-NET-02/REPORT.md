> 🌐 **Language / Язык:** **🇬🇧 English** | [🇷🇺 Русский](./REPORT.ru.md)

# [AHT-SEC-2026-NET-02] TECHNICAL SECURITY & PRIVACY ASSESSMENT REPORT: SIMPLEX CHAT PROTOCOL & ZERO-IDENTIFIER NETWORK ARCHITECTURE

> ⚖️ **Legal Notice:** This technical assessment is governed by the [ArmHack-Team Research Disclaimer](../../../DISCLAIMER.md). All conclusions reflect independent threat modeling, observable telemetry, and open standards verification.

| Metadata Attribute | Value / Specification |
| :--- | :--- |
| **Reference ID:** | `AHT-SEC-2026-NET-02` |
| **Authoring Body:** | `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` |
| **Subject:** | SimpleX Chat Client (`chat.simplex.app`) & SimpleX Messaging Protocol (`SMP` / `XFTP` / `simplexmq`) |
| **Version / SHA-256:** | SimpleX Chat `v6.x / v7.x` Stable (`chat.simplex.app` / `simplexmq` Haskell Core Engine; Reference APK & Desktop Builds Verified) |
| **Audit Scope:** | Unidirectional Queue Topology (`SMP`), Hybrid Post-Quantum Double Ratchet (`sntrup761` + `X25519`/`X448`), Wire Packet Padding (`16,384 Bytes`), `XFTP` Chunking, Local Storage (`SQLCipher`) |
| **Standards:** | IETF RFC 8446 (TLS 1.3), RFC 7748 (X25519/X448), Streamlined NTRU Prime (`sntrup761`), OWASP MASVS-CRYPTO / MASVS-NETWORK, NIST SP 800-52 Rev. 2 |
| **Date of Verification:** | October 2026 |
| **Risk Level:** | **LOW** (`GRADE A+` — Verified Zero-Identifier Architecture; Conditional Operational Vectors Noted) |

---

## 1. Executive Summary

`ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` has conducted an independent technical security evaluation and cryptographic architecture audit of the SimpleX Chat ecosystem (`chat.simplex.app`) and its underlying communication protocols (`SMP` and `XFTP`). The objective of this assessment was to verify the protocol's cryptographic trust boundaries, resistance against passive and active network traffic correlation, post-quantum key encapsulation mechanics, and client-side data sovereignty guarantees.

Conventional end-to-end encrypted (E2EE) messaging platforms protect message plaintext but structurally preserve global user identifiers (telephone numbers, account UUIDs, or static public key hashes) on centralized or federated directory servers. From a network threat-modeling standpoint, persistent identifiers allow server operators and on-path observers to reconstruct the global social graph and correlate communication metadata.

Our technical verification confirms that SimpleX Chat eliminates persistent network-visible user identifiers by implementing a **Zero-Identifier Unidirectional Queue Architecture**. Rather than addressing messages to user accounts, SimpleX routes payloads across ephemeral, unidirectional pairwise message queues (`SMP`). Combined with an inline Hybrid Post-Quantum Double Ratchet combining elliptic-curve Diffie-Hellman (`X25519`/`X448`) with **Streamlined NTRU Prime (`sntrup761`)**, strict `16,384-byte` (16 KB) transport block padding, optional 2-hop onion relay routing, and client-side `SQLCipher` database encryption, the protocol aligns closely with its declared privacy specifications. Residual risks are primarily confined to out-of-band invitation interception, low-throughput timing correlation, and OS-level push notification triggers.

---

## 2. Threat Modeling & Scope

### 2.1. Systemic Boundaries & Unidirectional Queue Topology

The SimpleX Messaging Protocol (`SMP`) decouples bidirectional communication into two independent, unidirectional message queues hosted on user-designated relay servers:

* **Client Endpoints (`Peer A` and `Peer B`):** Autonomous client runtimes storing cryptographic state, contact ratchets, and message histories exclusively within local `SQLCipher` storage.
* **SMP Relay Servers:** Untrusted message transport middleboxes that process ephemeral, per-queue cryptographic credentials without visibility into user identities, cross-queue linkage, or message plaintext.
* **Optional Forwarding Proxy Relay (2-Hop Onion Routing):** An intermediate SMP node operated between the sender and the recipient's destination SMP server to decouple the sender's origin IP address from the destination queue.

```text
+-----------------------------------------------------------------------------------------+
|                        SIMPLEX UNIDIRECTIONAL QUEUE TOPOLOGY                            |
|                                                                                         |
|  [Peer A Endpoint] ──(TLS 1.3 + 16 KB Padded Block)──> [Forwarding SMP Relay (Hop 1)]   |
|         ▲                                                            │                  |
|         │ (Return Queue B -> A)                                      ▼                  |
|  [SMP Relay 1 (Chosen by A)]                           [Destination SMP Relay (Hop 2)]  |
|         ▲                                                            │                  |
|         │                                              (Delivers 16 KB Encrypted Block) |
|         └────────────(TLS 1.3 + 16 KB Padded Block)──── [Peer B Endpoint] <─────────────┘
+-----------------------------------------------------------------------------------------+
```

### 2.2. Cryptographic & Protocol Standards Alignment

1. **Zero Global User Identifiers:** Neither client registration nor peer discovery utilizes phone numbers, email addresses, usernames, or static global public keys. Every pairwise connection generates isolated ephemeral queues and independent cryptographic ratchets.
2. **Inline Post-Quantum Double Ratchet (`sntrup761` + `X25519`/`X448`):** Unlike protocols that apply post-quantum key encapsulation solely during the initial handshake, `simplexmq` integrates **Streamlined NTRU Prime (`sntrup761`)** directly into continuous Double Ratchet steps alongside `X25519`/`X448`, maintaining post-quantum break-in recovery (Post-Compromise Security) throughout the conversation lifecycle.
3. **Uniform Wire Transport Padding (`16,384 Bytes`):** All `SMP` transport blocks and `XFTP` file transfer chunks (capped at `15,780 bytes` of payload per chunk before envelope headers) are strictly padded to `16,384 bytes` on the wire to mitigate payload-size fingerprinting.

---

## 3. Discrepancy Matrix: Declarations vs. Observable Reality

| Evaluated Parameter | Public / Vendor Declaration | Observable Technical Reality | Cryptographic / Privacy Impact |
| :--- | :--- | :--- | :--- |
| **User Identity Model** | "Complete absence of user identifiers—not even random numbers." | Verified in `simplexmq`: connections utilize pairwise unidirectional queue IDs with distinct key pairs per contact; zero global user ID exists on relays. | Eliminates server-side social graph reconstruction across independent conversations (**Verified Compliant**). |
| **Post-Quantum Cryptography** | "Quantum-resistant end-to-end encryption enabled by default for direct chats." | Implements hybrid Double Ratchet combining `X25519`/`X448` with `sntrup761` (Streamlined NTRU Prime) across continuous ratchet steps (since v5.7+). | Protects both initial handshakes and ongoing ratchet recovery against harvest-now, decrypt-later quantum adversaries (**Verified Compliant**). |
| **Wire Packet Size Uniformity** | "Fixed packet sizes prevent traffic size analysis." | All `SMP` wire frames are padded to a constant `16,384 bytes` (16 KB); `XFTP` chunks are capped at `15,780 bytes` to fit inside the 16 KB transport block. | Neutralizes packet-length fingerprinting; passive observers still observe packet timing and volume bursts (**Low Residual Risk**). |
| **Origin IP Concealment** | "Private message routing hides IP addresses from recipient servers." | When 2-hop Private Routing is enabled, the destination SMP relay observes only the forwarding relay's IP; however, single-hop mode exposes sender IP to the destination relay. | Requires explicit verification that 2-hop routing or Tor (`SOCKS5`) is active to prevent single-relay IP correlation (**Configuration Dependent**). |
| **Push Notification Privacy** | "Private push notifications without exposing message content." | iOS (`APNs`) and Google Play (`FCM`) builds transmit encrypted wake-up tokens via notification gateways; OS vendors still observe wake-up timestamps and device tokens. | Metadata of notification timing is visible to Apple/Google unless F-Droid/APK background polling service is used (**Ecosystem Constraint**). |

---

## 4. Technical Analysis & Evidence

### 4.1. Multi-Layered Cryptographic Envelope Verification

Protocol inspection of `simplexmq` confirms that every dispatched message undergoes three nested cryptographic encapsulation layers prior to transit:

1. **Layer 1 — End-to-End Ratchet Payload:** Plaintext and application metadata are encrypted between client endpoints using the Hybrid Post-Quantum Double Ratchet (`X25519`/`X448` + `sntrup761`, `HKDF-SHA512`, `NaCl crypto_secretbox` / `XSalsa20-Poly1305`).
2. **Layer 2 — SMP Queue Envelope:** The E2EE ciphertext is wrapped in an `SMP` agent envelope encrypted with pairwise queue keys (`Ed25519` authentication signatures and ephemeral session keys) and padded to `16,384 bytes`.
3. **Layer 3 — Transport Layer Security:** The `16,384-byte` block is transmitted over an authenticated TLS 1.3 channel (`RFC 8446`) with certificate public-key pinning to the designated SMP relay and TLS session resumption disabled to prevent transport-layer correlation.

#### Sanitized SMP Wire Transport Frame Summary (Laboratory Capture):
```text
[Transport Layer]   TLS 1.3 (ALPN: smp/1.0, Resumption: Disabled, SPKI Pin: [REDACTED])
[SMP Wire Block]    Total Frame Length: 16,384 Bytes (0x4000 Constant Size)
  ├── SMP Header:   Protocol Version, Command Tag (SEND / MSG), Ephemeral Nonce [REDACTED]
  ├── Queue Auth:   Ed25519 Per-Queue Signature [REDACTED]
  ├── Payload:      PQDR Ciphertext (X25519/X448 + sntrup761 Hybrid Ratchet Envelope)
  └── Padding:      Deterministic Pad to 15,780 / 16,384 Byte Boundary
```

### 4.2. SimpleX File Transfer Protocol (`XFTP`) Analysis

Unlike WebRTC-based messengers that default to peer-to-peer UDP hole-punching (which can leak endpoint IP addresses via STUN/TURN), SimpleX routes media and files asynchronously via `XFTP` relays:
* Files are encrypted locally using ephemeral symmetric keys before transmission.
* Encrypted files are split into fixed-size chunks (`<= 15,780 bytes` payload) distributed across multiple `XFTP` queues, encapsulated in `16,384-byte` transport blocks.
* Decryption keys and chunk descriptors are delivered exclusively inside the E2EE `SMP` chat channel, and chunks expire automatically on relays after a short retention window (typically 48 hours).

### 4.3. Endpoint Local Storage Architecture (`SQLCipher`)

Static and runtime analysis of the `chat.simplex.app` client confirms:
* **Encrypted Database Persistence:** All chat histories, contact ratchet states, and queue credentials are stored inside two local SQLite databases (`simplex_v1_chat.db` and `simplex_v1_agent.db`) encrypted via `SQLCipher` (AES-256-CBC with HMAC-SHA512 page authentication and PBKDF2 key derivation).
* **Passphrase Isolation Option:** By default on mobile devices, the database passphrase may be stored in the platform hardware keystore (`Android KeyStore` / `iOS Keychain`) for background notification processing. Users can disable keystore caching and enforce a manual startup passphrase, keeping the database cryptographically locked at rest.

---

## 5. Threat Vectors & Structural Consequences

### 5.1. Threat Vector 1: Out-of-Band Invitation Link Interception (Handshake MitM)
* **Severity:** **MEDIUM** (High if Safety Codes remain unverified)
* **Precondition:** A user shares a 1-time invitation link (`simplex:/...`) over an untrusted or compromised third-party channel (e.g., an unpinned web platform or SMS).
* **Technical Consequence:** Because SimpleX does not rely on a central identity authority, an adversary who intercepts the invitation link before the intended recipient can consume the single-use queue token and establish a man-in-the-middle session unless both peers perform out-of-band **Safety Code (Security Code)** verification.

### 5.2. Threat Vector 2: Timing & Volume Correlation by Global Network Adversaries
* **Severity:** **MEDIUM** (Under targeted Autonomous System / ISP monitoring)
* **Precondition:** An adversary monitors network ingress and egress across low-throughput SMP relays or correlates burst transmissions during active 1-on-1 conversations.
* **Technical Consequence:** While strict `16,384-byte` padding defeats packet-size fingerprinting, SimpleX operates as a low-latency messaging system rather than a high-latency Chaumian mixnet. An observer with visibility into both endpoints' network links can perform statistical timing correlation unless traffic is routed through Tor (`SOCKS5` / `.onion` relays).

### 5.3. Threat Vector 3: OS Push Notification Gateway Metadata Exposure
* **Severity:** **LOW**
* **Precondition:** Usage of standard Apple App Store (`APNs`) or Google Play (`FCM`) client distributions with push notifications enabled.
* **Technical Consequence:** Although push payloads contain zero plaintext or sender identifiers, Apple and Google push infrastructure can observe the timestamp and frequency of wake-up events delivered to a specific device token.

### 5.4. Threat Vector 4: Endpoint Memory Extraction upon Host OS Compromise
* **Severity:** **HIGH** (Inherent to all endpoint applications)
* **Precondition:** Physical device seizure in an unlocked state or execution of privileged root/kernel malware on the host OS.
* **Technical Consequence:** If the database passphrase is cached in the OS keystore or the application is actively unlocked in RAM, a privileged local process can extract decrypted session state.

---

## 6. Mitigation & Hardening Recommendations

### 6.1. Hardening Measures for End-Users & High-Risk Deployments

1. **Enforce 2-Hop Private Routing & Tor (`SOCKS5`) Transport:**
   * Enable **Private Message Routing (2-hop)** across all network profiles and configure the client to route connections through `Orbot` / `Tor` (`SOCKS5` proxy on `127.0.0.1:9050`) using `.onion` SMP and XFTP relays to conceal origin IP addresses.
2. **Mandate Out-of-Band Safety Code Verification:**
   * Immediately after establishing any contact link, verify the cryptographic **Security Code** out-of-band (via voice or in-person comparison) to confirm no out-of-band invitation substitution occurred.
3. **Disable OS Keystore Caching & Set a Dedicated Database Passphrase:**
   * Enable a strong custom **Database Passphrase** in SimpleX Chat settings and disable automatic passphrase storage in the OS Keychain/KeyStore so that the `SQLCipher` database remains unmounted when the app is closed.
4. **Use Background Service Mode instead of FCM (Android):**
   * Deploy the official F-Droid or direct-signed APK build configured with persistent background polling rather than Google Play Services (`FCM`) push notifications.

### 6.2. Guidance for Organizational & Self-Hosted Infrastructure

1. **Deploy Self-Hosted `smp-server` and `xftp-server` Nodes:**
   * Organizations requiring sovereign relay availability should deploy dedicated `smp-server` and `xftp-server` instances behind both TLS 1.3 and Tor v3 `.onion` hidden services, enforcing creation passwords (`-p`) so only authorized organizational clients can provision queues while external peers can still deliver messages to them.

---

**ArmHack-Team Advisory Protocol**  
*Document compiled and verified by `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` under STIRS v2.0.*
