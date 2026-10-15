> 🌐 **Language / Язык:** **🇬🇧 English** | [🇷🇺 Русский](./REPORT.ru.md)

# [AHT-SEC-2026-APP-03] TECHNICAL SECURITY & PRIVACY ASSESSMENT REPORT: ELEMENT MESSENGER CLIENT & MATRIX FEDERATED PROTOCOL ARCHITECTURE

> ⚖️ **Legal Notice:** This technical assessment is governed by the [ArmHack-Team Research Disclaimer](../../../DISCLAIMER.md). All conclusions reflect independent threat modeling, observable telemetry, and open standards verification.

| Metadata Attribute | Value / Specification |
| :--- | :--- |
| **Reference ID:** | `AHT-SEC-2026-APP-03` |
| **Authoring Body:** | `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` |
| **Subject:** | Element Messenger Client (`im.vector.app` / Element X `io.element.android.x`) & Matrix Open Federated Communication Protocol |
| **Version / SHA-256:** | Element X (`io.element.android.x` v24.x–v26.x) / Element Classic (`im.vector.app`) & `vodozemac` Pure-Rust Cryptographic Engine (`matrix-rust-sdk`) |
| **Audit Scope:** | `Olm` / `Megolm` Cryptography (`vodozemac`), Replicated Event DAG State, Native Sliding Sync (`MSC4186` / `MSC3575`), Authenticated Media Repository (`MXC`), Homeserver Federation Metadata Surface |
| **Standards:** | Matrix Spec v1.11+, IETF RFC 8446 (TLS 1.3), RFC 7748 (`Curve25519`/`Ed25519`), OWASP MASVS-CRYPTO / MASVS-STORAGE, NIST SP 800-52 Rev. 2 |
| **Date of Verification:** | October 2026 |
| **Risk Level:** | **MEDIUM** (`GRADE B+` — Robust Content Encryption; Inherent Federated DAG Metadata Replication) |

---

## 1. Executive Summary

`ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` has conducted an independent technical security evaluation and cryptographic architecture audit of the Element Messenger client ecosystem (`im.vector.app` and Element X `io.element.android.x`) and the underlying decentralized Matrix communication protocol. This assessment evaluates the cryptographic boundaries of Matrix's modern pure-Rust `vodozemac` encryption library (`Olm` and `Megolm` ratchets), analyzes metadata persistence across federated homeservers, and models the structural trust assumptions of Eventual-Consistency Directed Acyclic Graph (DAG) replication.

Unlike zero-identifier queue architectures, the Matrix protocol operates as an open, federated state-replication network. Every participating homeserver in a room maintains a cryptographically signed, synchronized copy of the room state and event DAG history. While end-to-end encryption (E2EE) protects message body text and media attachments from untrusted homeserver operators, the federation protocol structurally replicates operational metadata—including global Matrix User IDs (`MXIDs`), room membership graphs, event timestamps, device key registries, and server-to-server network origins—across all homeservers participating in a room.

Our technical verification confirms that Element and Matrix deliver strong payload confidentiality via the audited `vodozemac` implementation of `Olm` (1:1 Double Ratchet) and `Megolm` (Group Ratchet), supplemented by interactive cross-signing device verification. However, from a metadata threat-modeling perspective, decentralized DAG state replication, unpadded JSON event envelopes, and `Megolm`'s forward-only group ratcheting trade-offs require deliberate administrative and client-side hardening to mitigate metadata harvesting in high-sensitivity deployments.

---

## 2. Threat Modeling & Scope

### 2.1. Systemic Boundaries & Federated DAG State Replication

The Matrix protocol models room history as a decentralized, eventual-consistency Directed Acyclic Graph (DAG) synchronized across participating homeservers over HTTPS Federation APIs:

* **Client Endpoints (`im.vector.app` / `io.element.android.x`):** Client applications managing `vodozemac` E2EE ratchet states, cross-signing keys, and local `SQLCipher` / `IndexedDB` stores, communicating with their home server via the Matrix Client-Server API and Native Sliding Sync (`MSC4186`).
* **Originating Homeserver (`Synapse` / `Dendrite` / `Conduit`):** The authoritative server for a given user's `MXID` (`@user:domain.tld`), responsible for signing outgoing DAG events, hosting device public key directories, and storing encrypted media blobs (`mxc://`).
* **Remote Federated Homeservers:** External server instances joined to shared rooms via the Matrix Server-to-Server (S-S) Federation API, receiving full replicas of all room state events and encrypted timeline payloads.

```text
+-----------------------------------------------------------------------------------------+
|                         MATRIX FEDERATED DAG REPLICATION TOPOLOGY                       |
|                                                                                         |
|  [Element Client A] ──(Matrix C-S API / Native Sliding Sync MSC4186)──> [Homeserver 1]  |
|  (vodozemac E2EE)                                                       (Origin DAG)    |
|                                                                               │         |
|                                                                (Federation S-S API/TLS) |
|                                                                               ▼         |
|  [Element Client B] <─(Matrix C-S API / Native Sliding Sync MSC4186)─── [Homeserver 2]  |
|  (vodozemac E2EE)                                                       (Replicated DAG)|
+-----------------------------------------------------------------------------------------+
```

### 2.2. Cryptographic & Protocol Standards Alignment

1. **Transition from `libolm` to `vodozemac`:** Modern Element clients (`io.element.android.x`, Element Web/Desktop, and updated mobile clients) replace the deprecated C-based `libolm` implementation with `vodozemac`, an independently audited pure-Rust implementation of the `Olm` (Triple Diffie-Hellman `3DH` + Double Ratchet) and `Megolm` cryptographic specifications.
2. **Native Sliding Sync (`MSC4186` superseding `MSC3575`):** To resolve the bandwidth and metadata overhead of legacy `/sync` endpoints—and eliminate the intermediate proxy trust boundary of experimental `MSC3575`—modern Matrix deployments implement Simplified Native Sliding Sync (`MSC4186`) directly within the homeserver.
3. **Authenticated Media Endpoints (`MSC3916`):** Matrix v1.11+ deprecates unauthenticated `mxc://` media retrieval in favor of authenticated media endpoints, restricting arbitrary third-party scraping of encrypted media blobs while still exposing attachment size and access timing to participating homeservers.

---

## 3. Discrepancy Matrix: Declarations vs. Observable Reality

| Evaluated Parameter | Public / Vendor Declaration | Observable Technical Reality | Cryptographic / Privacy Impact |
| :--- | :--- | :--- | :--- |
| **Decentralized Privacy Posture** | "Decentralized, end-to-end encrypted communication with no single point of control." | Room DAG state, membership lists, event timestamps, and sender `MXIDs` are permanently replicated across every homeserver participating in a room. | Any untrusted or compromised homeserver in a federated room receives a complete copy of the room's social graph and timing metadata (**High Metadata Exposure**). |
| **Group Chat Ratchet Security (`Megolm`)** | "End-to-end encryption powered by Olm and Megolm secures group conversations." | `Megolm` uses a forward-only symmetric ratchet (`AES-256-CBC` + `HMAC-SHA-256`) shared via 1:1 `Olm` channels to scale to large rooms without $O(N^2)$ overhead. | Provides Forward Secrecy, but lacks intra-session Post-Compromise Security (PCS) until a new `Megolm` session rotation occurs (**Architectural Trade-off**). |
| **Identity & Profile Partitioning** | "Users can set custom per-room display names and avatars." | Display aliases are local to room state events, but the underlying `@user:domain.tld` `MXID` and `Device ID` remain globally immutable across all joined rooms. | Trivially correlates user participation across disparate public and private rooms sharing a common homeserver (**Persistent Global ID**). |
| **Wire Payload Size Concealment** | "Encrypted JSON event payloads (`m.room.encrypted`) protect message contents." | `m.room.encrypted` JSON structures do not enforce uniform fixed-size block padding on the wire. | Passive network observers and homeserver operators can infer approximate plaintext length and media types via ciphertext size analysis (**Traffic Fingerprinting Vector**). |
| **Push Notification Architecture (`Sygnal`)** | "Configurable push notifications supporting custom gateways and UnifiedPush." | Default mobile installations route push triggers through Matrix's `Sygnal` gateway to `APNs` / `FCM`, transmitting room IDs, event IDs, or unread counts unless set to `event_id_only`. | Default push configurations expose notification timing and room activity frequency to push gateway operators (**Configuration Dependent**). |

---

## 4. Technical Analysis & Evidence

### 4.1. Cryptographic Engine Inspection (`vodozemac`: `Olm` & `Megolm`)

Inspection of the `vodozemac` cryptographic stack confirms two distinct ratchet layers:

1. **Pairwise `Olm` Channels (Device-to-Device):**
   * Establishes 1:1 authenticated channels between user devices using `Curve25519` identity and one-time prekeys (`3DH` agreement) and `Ed25519` fingerprint keys.
   * Advances symmetric keys via `HKDF-SHA-256` on every message exchange, providing both Forward Secrecy and Post-Compromise Security (PCS).
2. **Group `Megolm` Sessions (Room Payload Encryption):**
   * Each sender initializes an outbound `Megolm` ratchet consisting of a 4-component 128-bit counter/PRNG state (`R_0, R_1, R_2, R_3`) hashed via `HMAC-SHA-256` to derive `AES-256-CBC` encryption keys and `HMAC-SHA-256` MAC keys.
   * The initial `Megolm` session key is distributed to authorized recipient devices over pairwise `Olm` `m.room_key` to-device events.
   * **Rotation Boundary:** Because `Megolm` does not perform Diffie-Hellman ratcheting per group message, healing from a compromised session state requires an explicit session rotation (triggered by membership changes, device revocation, or reaching configured message/time limits).

### 4.2. Federated Event DAG Structure & Unencrypted Envelope Metadata

While the `ciphertext` inside an `m.room.encrypted` event is opaque to homeservers, the surrounding Matrix event envelope remains visible in plaintext within the homeserver database and across federation S-S streams:

#### Sanitized Federated Room Event Envelope (Laboratory Capture):
```json
{
  "type": "m.room.encrypted",
  "event_id": "$[REDACTED_EVENT_HASH]",
  "room_id": "![REDACTED_ROOM_ID]:matrix.example.org",
  "sender": "@[REDACTED_USER]:origin.example.com",
  "origin": "origin.example.com",
  "origin_server_ts": 1791936000000,
  "content": {
    "algorithm": "m.megolm.v1.aes-sha2",
    "ciphertext": "[REDACTED_VARIABLE_LENGTH_BASE64_CIPHERTEXT]",
    "device_id": "[REDACTED_DEVICE_ID]",
    "sender_key": "[REDACTED_CURVE25519_KEY]",
    "session_id": "[REDACTED_MEGOLM_SESSION_ID]"
  },
  "hashes": {
    "sha256": "[REDACTED_CONTENT_HASH]"
  },
  "signatures": {
    "origin.example.com": {
      "ed25519:[REDACTED_KEY_ID]": "[REDACTED_HOMESERVER_SIGNATURE]"
    }
  }
}
```

As demonstrated in the captured event structure, every federated homeserver in the room permanently records:
* The exact sender `MXID` (`sender`) and originating `device_id`.
* The millisecond-precision timestamp (`origin_server_ts`).
* The active `Megolm` `session_id` and exact `ciphertext` byte length.
* The cryptographic signature of the originating homeserver (`signatures`), which cryptographically proves domain origin.

### 4.3. Encrypted Media Repository (`MXC`) & Key Backup Verification

* **Attachment Handling:** Client applications encrypt media files locally using `AES-256-CTR` before uploading the ciphertext blob to the homeserver's content repository (`/_matrix/media/v3/upload`). The `AES-256-CTR` key, initialization vector (`iv`), and `SHA-256` digest are transmitted inside the E2EE `Megolm` room event. However, homeservers retain visibility into exact file byte sizes and client IP addresses requesting `mxc://` downloads.
* **Server-Side Key Backup (`m.megolm_backup.v1.curve25519-aes-sha2`):** Historical `Megolm` keys backed up to the homeserver are encrypted client-side using a recovery key / Security Passphrase derived via `PBKDF2-HMAC-SHA-512` before upload.

---

## 5. Threat Vectors & Structural Consequences

### 5.1. Threat Vector 1: Metadata Harvesting via Untrusted Federated Homeservers
* **Severity:** **HIGH** (For anonymity-sensitive operations)
* **Precondition:** Joining federated rooms that include users hosted on third-party or public homeservers (e.g., `matrix.org` or unknown external domains).
* **Technical Consequence:** Because Matrix replicates the full room DAG to every participating homeserver, any single server in a room receives real-time metadata of all room members (`MXIDs`), join/leave events, message timestamps, and ciphertext sizes. Furthermore, deletion/redaction requests (`m.room.redaction`) rely on cooperative server compliance and cannot cryptographically force an untrusted remote homeserver to purge historical metadata.

### 5.2. Threat Vector 2: Rogue Device Key Injection by a Compromised Origin Homeserver
* **Severity:** **HIGH** (If Cross-Signing Verification is bypassed or ignored)
* **Precondition:** An attacker compromises a user's own homeserver database or administrative API, and room participants allow unverified devices to receive group keys.
* **Technical Consequence:** Since the homeserver controls the user device directory (`/_matrix/client/v3/keys/query`), a compromised homeserver could attempt to append a rogue device key to a target user's identity. If sending clients do not enforce strict cross-signing verification ("Never send encrypted messages to unverified sessions"), they may automatically share subsequent `Megolm` outbound session keys with the injected device.

### 5.3. Threat Vector 3: Intra-Session `Megolm` Key Compromise (Absence of Per-Message PCS)
* **Severity:** **MEDIUM**
* **Precondition:** Temporary compromise or memory extraction of a single active `Megolm` inbound session key (`session_id`).
* **Technical Consequence:** Because `Megolm` ratchets forward deterministically via `HMAC-SHA-256` without per-message Diffie-Hellman entropy injection, an adversary possessing a session key at index $i$ can compute all subsequent message keys $i+1, i+2, \dots, n$ within that specific `session_id` until the sender rotates the `Megolm` session.

### 5.4. Threat Vector 4: Push Gateway (`Sygnal`) & Network Traffic Timing Correlation
* **Severity:** **MEDIUM**
* **Precondition:** Use of default push notification gateways (`Sygnal` -> `FCM`/`APNs`) and unpadded JSON event transport.
* **Technical Consequence:** Push gateways observe device wake-up timing and room identifiers, while passive network observers can correlate variable-length HTTPS payload bursts between clients and homeservers.

---

## 6. Mitigation & Hardening Recommendations

### 6.1. Hardening Measures for End-Users & Client Configurations

1. **Enforce Strict Cross-Signing Verification & Block Unverified Devices:**
   * Complete interactive emoji/QR cross-signing verification (`SAS`) for all personal and contact devices.
   * Enable the client setting **"Exclude insecure devices" / "Never send encrypted messages to unverified sessions"** so that `Megolm` keys are never distributed to unverified device keys injected by a homeserver.
2. **Avoid Large Public Homeservers for Sensitive Communications:**
   * Do not register sensitive operational accounts on centralized public instances (such as `matrix.org`), where thousands of federated rooms concentrate social graph metadata in a single database.
3. **Configure Ephemeral &Pseudonymous `MXID` Accounts over Tor (`SOCKS5`):**
   * Route Element client traffic through `Tor` / `Orbot` (`SOCKS5`) or an always-on WireGuard tunnel to prevent homeservers and `mxc://` media hosts from logging client origin IP addresses.
4. **Minimize Push Notification Payload Exposure:**
   * Configure mobile clients to use `event_id_only` push notification format or `UnifiedPush` with a self-hosted distributor rather than default `FCM`/`APNs` payload forwarding.

### 6.2. Defensive Engineering Guidance for Homeserver Administrators

1. **Deploy Private, Federation-Restricted Homeservers:**
   * For internal organizational deployments, host dedicated `Synapse` or `Conduit` instances with public registration disabled (`enable_registration: false`) and enforce strict federation allow-lists (`federation_domain_whitelist`) or completely disable external S-S federation.
2. **Tune Aggressive `Megolm` Session Rotation Parameters:**
   * Configure room encryption state events (`m.room.encryption`) to reduce `rotation_period_msgs` (e.g., from the default `100` messages down to `20–50` messages) and `rotation_period_ms` (e.g., from `7 days` down to `24 hours`), minimizing the exposure window of any single `Megolm` session key.
3. **Enforce Authenticated Media (`MSC3916`) & Database Retention Policies:**
   * Enable authenticated media endpoints and configure automated database retention/purging jobs (`retention` module in `Synapse`) to regularly prune historical DAG state and remote media caches.

---

**ArmHack-Team Advisory Protocol**  
*Document compiled and verified by `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` under STIRS v2.0.*
