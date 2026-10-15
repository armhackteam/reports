> 🌐 **Language / Язык:** **🇬🇧 English** | [🇷🇺 Русский](./REPORT.ru.md)

# [AHT-SEC-2026-PKI-01] TECHNICAL SECURITY ASSESSMENT REPORT: RUSSIAN NATIONAL ROOT CA IMPLEMENTATION & TRUST BOUNDARY ARCHITECTURE

> ⚖️ **Legal Notice:** This technical assessment is governed by the [ArmHack-Team Research Disclaimer](../../../DISCLAIMER.md). All conclusions reflect independent threat modeling, observable telemetry, and open standards verification.

| Metadata Attribute | Value / Specification |
| :--- | :--- |
| **Reference ID:** | `AHT-SEC-2026-PKI-01` |
| **Authoring Body:** | `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` |
| **Subject:** | Russian Federation National Root CA (`Russian Trusted Root CA` / Ministry of Digital Development) & Integrated Client Software Ecosystems (`Yandex Browser`) |
| **Version / SHA-256:** | 1. `russian_trusted_root_ca.cer` (SHA-256: `d26d2d0231b7c39f92cc738512ba54103519e4405d68b5bd703e9788ca8ecf31`)<br>2. `russian_trusted_sub_ca.cer` (SHA-256: `2155785036c900dbb5f1bb2a1569c80c55595bd6bf94867a29bbddbc7d88a3f2`)<br>3. Yandex Browser (`com.yandex.browser`, Chromium Engine Integration, Build 24.x–26.x Stable) |
| **Audit Scope:** | Public Key Infrastructure (PKI) Architecture, Operating System Trust Store Modification, Transit-Layer Inspection Surface, Client-Side Cryptographic Validation Bounds |
| **Standards:** | IETF RFC 5280, RFC 8446, RFC 6962 / RFC 9162, CA/Browser Forum Baseline Requirements, OWASP MASVS-NETWORK, NIST SP 800-52 Rev. 2, CWE-295 |
| **Date of Verification:** | October 2026 |
| **Risk Level:** | **CRITICAL** |

---

## 1. Executive Summary

`ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` has conducted an independent cryptographic trust architecture assessment regarding the deployment of the Russian National Certification Authority (`Russian Trusted Root CA`, operated by the Ministry of Digital Development, Communications and Mass Media of the Russian Federation) across endpoint operating systems and pre-configured client software ecosystems. This evaluation examines the structural security implications of installing a non-CA/B-Forum-audited root certificate into operating system trust repositories, alongside the autonomous trust store mechanisms implemented in client software such as Yandex Browser (`com.yandex.browser`).

Under established Public Key Infrastructure (PKI) specifications (RFC 5280, RFC 8446), a trusted root certification authority possesses global mathematical capability to issue valid cryptographic certificates for any domain namespace worldwide unless explicitly constrained by X.509 extensions. Incorporating an unconstrained root certificate into an endpoint operating system trust store structurally alters the cryptographic trust boundaries of Transport Layer Security (TLS) across all applications relying on system validation APIs.

From a defensive PKI threat-modeling standpoint, installing a state-managed Root CA without X.509 `NameConstraints` into an operating system trust store technically empowers the private key holder—or any intermediate authority chained to it—to conduct on-path Man-in-the-Middle (MitM) TLS session termination and re-encryption. Without client-side Subject Public Key Info (SPKI) certificate pinning, end-user authentication sessions traversing domestic transit nodes lack independent end-to-end cryptographic confidentiality guarantees against middlebox inspection.

---

## 2. Threat Modeling & Scope

### 2.1. Systemic Boundaries & Trust Relationships

The Transport Layer Security (TLS 1.2 / TLS 1.3) security model relies on a strictly audited set of root trust anchors maintained within client operating systems and browser engines.

* **Trusted Party (Root Anchor Authority):** The entity holding the private cryptographic key of the root and issuing intermediate certificates (`The Ministry of Digital Development and Communications`, `CN = Russian Trusted Root CA`).
* **On-Path Entities & Network Middleboxes:** Autonomous Systems (AS), upstream transit operators, and inline network inspection middleboxes (e.g., Deep Packet Inspection / TSPU middlebox infrastructure positioned along transit pathways).
* **Endpoint / Client Runtime:** The host operating system trust store (Windows CryptoAPI, Apple Keychain, Linux `/etc/ssl/certs`, Android KeyStore) and autonomous application trust stores (`com.yandex.browser`).

```text
+-----------------------------------------------------------------------------------------+
|                                    TRUST BOUNDARY                                       |
|                                                                                         |
|  [Client Endpoint]               [On-Path Inspection Middlebox]      [Upstream Server]  |
|  (OS / App Trust Store)          (Possesses Sub-CA Key Material)     (Origin Service)   |
|         |                                       |                            |          |
|         |==== Client TLS Handshake =============>                            |          |
|         |     (SNI: service.example.com)        |                            |          |
|         |                                       |==== Origin TLS Handshake ==>          |
|         |<=== Synthesized Leaf Certificate =====|                            |          |
|         |     (Signed by Russian Root CA)       |<=== Genuine Certificate ===|          |
|         |                                       |                            |          |
|         |==== Plaintext HTTP Payload ===========> (Inspects Transit Data)    |          |
|         |                                       |==== Re-encrypted Payload =>|          |
+-----------------------------------------------------------------------------------------+
```

### 2.2. Normative Deviations from International PKI Standards

The architectural parameters of the evaluated hierarchy diverge from standard international PKI governance models in three key areas:

1. **Absence of X.509 Name Constraints (RFC 5280 Section 4.2.1.10):** Neither `russian_trusted_root_ca.cer` nor `russian_trusted_sub_ca.cer` incorporates the `NameConstraints` extension restricting permitted subtrees to domestic top-level domains (`.ru`, `.рф`, `.su`). Consequently, the cryptographic key pair is mathematically capable of validating leaf certificates for any global domain namespace (e.g., `.com`, `.org`, `.net`, `.io`).
2. **CA/Browser Forum Baseline Requirements Scope:** Commercial root authorities included in mainstream browser programs (Mozilla NSS, Microsoft, Apple, Chrome Root Program) undergo recurring independent WebTrust or ETSI audits verifying Hardware Security Module (HSM) key ceremonies and issuance controls. The evaluated hierarchy operates autonomously outside global browser root programs.
3. **Certificate Transparency (CT) Ecosystem Segregation (RFC 6962 / RFC 9162):** While Yandex Browser validates certificates issued under this hierarchy against domestic Certificate Transparency logs, manual installation of the root certificate directly into an operating system trust store bypasses browser-specific CT enforcement for system-wide non-browser applications and standard user agents.

---

## 3. Discrepancy Matrix: Declarations vs. Observable Reality

| Evaluated Parameter | Public / Vendor Declaration | Observable Technical Reality | Cryptographic / Privacy Impact |
| :--- | :--- | :--- | :--- |
| **Domain Issuance Scope** | "Provides uninterrupted and secure HTTPS access to domestic government, banking, and commercial portals." | Certificate omits RFC 5280 `NameConstraints`; mathematically validates leaf certificates for any arbitrary global domain or wildcard namespace. | Enables technical capability for transparent on-path TLS interception (CWE-295) across both domestic and international domains. |
| **OS Trust Store Boundary** | "Standard root installation required for proper operating system and web portal functionality." | Injects an unconstrained root anchor directly into OS-wide trust stores (Windows CryptoAPI, macOS Keychain, Linux CA bundle, Android System/User Store). | Removes cryptographic trust isolation across all OS processes and background clients that rely on system TLS validation APIs. |
| **Client Application Trust (`com.yandex.browser`)** | "Built-in certificate support provides seamless access to domestic sites without manual OS configuration." | Application embeds autonomous trust rules for `Russian Trusted Root CA` tied to domestic CT log verification, decoupled from host OS root policies. | Accepts hierarchy-signed certificates within the browser runtime even on host operating systems where administrators have excluded the root CA. |
| **Global CT Log Visibility** | "Full compliance with modern transport cryptographic protocols and transparency standards." | Certificates issued under this hierarchy are not systematically required by OS-level validators to carry SCTs from globally trusted Chrome/Apple CT logs. | External domain owners monitoring only international CT logs may not detect unauthorized leaf certificate generation for OS-level clients. |
| **Revocation Infrastructure (CRL / OCSP)** | "High-availability certificate status verification via official endpoints." | CRL and OCSP responders (`crl.gosuslugi.ru`, `ocsp.gosuslugi.ru`) reside within domestic routing zones subject to transit-layer traffic management. | If revocation queries time out or are unreachable across network boundaries, soft-fail OCSP clients proceed with handshake completion. |

---

## 4. Technical Analysis & Evidence

### 4.1. Cryptographic Certificate Chain Inspection

Static inspection of the publicly distributed root and intermediate X.509v3 artifacts yields the following deterministic parameters:

#### Root Certificate Artifact (`russian_trusted_root_ca.cer`)
* **Subject DN:** `C = RU, O = The Ministry of Digital Development and Communications, CN = Russian Trusted Root CA`
* **Issuer DN:** `C = RU, O = The Ministry of Digital Development and Communications, CN = Russian Trusted Root CA`
* **Public Key Algorithm:** `rsaEncryption` (4096-bit RSA)
* **Signature Algorithm:** `sha256WithRSAEncryption`
* **SHA-256 Fingerprint:** `d26d2d0231b7c39f92cc738512ba54103519e4405d68b5bd703e9788ca8ecf31`
* **Validity Period:** `2022-03-01 20:59:11 UTC` to `2032-02-27 20:59:11 UTC`
* **X509v3 Basic Constraints:** `critical, CA:TRUE`
* **X509v3 Key Usage:** `critical, Certificate Sign, CRL Sign` (`0x06`)
* **X509v3 Subject Key Identifier (SKI):** `3B:56:59:75:A0:98:89:D6:5D:0C:48:8D:18:2C:94:07:95:C0:CA:63`
* **X509v3 Name Constraints:** `ABSENT` (No permitted or excluded subtree constraints defined)

#### Intermediate Issuing Certificate (`russian_trusted_sub_ca.cer`)
* **Subject DN:** `C = RU, O = The Ministry of Digital Development and Communications, CN = Russian Trusted Sub CA`
* **Issuer DN:** `C = RU, O = The Ministry of Digital Development and Communications, CN = Russian Trusted Root CA`
* **Public Key Algorithm:** `rsaEncryption` (4096-bit RSA)
* **Signature Algorithm:** `sha256WithRSAEncryption`
* **SHA-256 Fingerprint:** `2155785036c900dbb5f1bb2a1569c80c55595bd6bf94867a29bbddbc7d88a3f2`
* **X509v3 Basic Constraints:** `critical, CA:TRUE, pathlen:0`
* **X509v3 Authority Key Identifier (AKI):** `3B:56:59:75:A0:98:89:D6:5D:0C:48:8D:18:2C:94:07:95:C0:CA:63`
* **X509v3 CRL Distribution Points:** `URI:http://crl.gosuslugi.ru/RussianTrustedRootCA.crl`
* **Authority Information Access (AIA):** `CA Issuers - URI:http://reestr-pki.ru/cdp/RussianTrustedRootCA.crt`
* **X509v3 Name Constraints:** `ABSENT`

### 4.2. On-Path TLS Termination Verification (Laboratory Simulation)

In a controlled laboratory topology simulating an on-path inspection middlebox provisioned with an intermediate signing key chained to an OS-installed unconstrained root CA, client connections exhibit transparent termination:

1. **ClientHello Interception:** The client initiates a TLS 1.3 handshake (`ClientHello`) with Server Name Indication (`SNI: service.example.com`).
2. **Dynamic Leaf Synthesis:** The inline middlebox generates an X.509 leaf certificate matching `SAN = DNS:service.example.com` signed by the trusted hierarchy.
3. **Silent Chain Validation:** Because the unconstrained root anchor is present in the OS trust store and no SPKI pinning is enforced by the client, path validation succeeds (`Verify return code: 0 (ok)`).
4. **Application-Layer Visibility:** The middlebox terminates the downstream TLS session, exposing the HTTP/2 stream prior to upstream re-encryption:

```http
POST /api/v2/authenticate HTTP/2
Host: service.example.com
User-Agent: Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko)
Accept: application/json, text/plain, */*
Authorization: Bearer [REDACTED]
X-Client-Device-UUID: [REDACTED]
Cookie: session_auth=[REDACTED]; csrf_token=[REDACTED]
Content-Type: application/json; charset=UTF-8
Content-Length: 128

{
  "account_identifier": "[REDACTED]",
  "auth_credential": "[REDACTED]",
  "client_timestamp": "2026-10-13T03:20:00Z"
}
```

### 4.3. Client Application Trust Store Analysis (`com.yandex.browser`)

Architectural analysis of Yandex Browser across desktop and mobile platforms (`com.yandex.browser`) confirms:
* **Embedded Trust Anchor Logic:** Yandex Browser maintains internal recognition of the `Russian Trusted Root CA` hierarchy out-of-the-box, independent of the host operating system's default certificate store.
* **Divergence from Host OS Security Policy:** On operating systems where the national root CA has not been installed—and where standard browsers such as Chromium, Firefox, or Safari reject the chain with `NET::ERR_CERT_AUTHORITY_INVALID` or `SEC_ERROR_UNKNOWN_ISSUER`—Yandex Browser completes the TLS handshake if the leaf certificate satisfies its internal CT validation rules.

---

## 5. Threat Vectors & Structural Consequences

### 5.1. Threat Vector 1: Transparent On-Path TLS Interception (OS Trust Store Scope)
* **Severity:** **CRITICAL** (`CVSS:3.1/AV:N/AC:L/PR:N/UI:N/S:C/C:H/I:H/A:N`)
* **Precondition:** Manual or GPO-driven installation of `russian_trusted_root_ca.cer` into the operating system root store.
* **Technical Consequence:** Any network middlebox or entity with access to a signing key chained to the unconstrained root can terminate and inspect TLS traffic for any unpinned domain without triggering user-facing browser or OS security warnings.

### 5.2. Threat Vector 2: Cross-Namespace Certificate Issuance (Absence of NameConstraints)
* **Severity:** **CRITICAL**
* **Precondition:** Omission of the X.509 `NameConstraints` extension in both the root and intermediate CA certificates.
* **Technical Consequence:** Rather than being cryptographically restricted to `.ru` or `.рф` namespaces at the X.509 protocol level, the root anchor possesses mathematical validity across all global TLDs (`.com`, `.org`, `.net`, etc.). If an intermediate key is compromised or misused, endpoints trusting the root cannot cryptographically distinguish between legitimate origin certificates and synthesized cross-namespace certificates.

### 5.3. Threat Vector 3: Application-Layer Policy Bypass via Autonomous Trust Stores
* **Severity:** **HIGH**
* **Precondition:** Execution of client software (`com.yandex.browser`) featuring embedded national CA trust anchors on an otherwise unconfigured host OS.
* **Technical Consequence:** Enterprise or individual security postures that rely on keeping the host OS certificate store unmodified are bypassed within the scope of the browser runtime.

---

## 6. Mitigation & Hardening Recommendations

### 6.1. Hardening Measures for End-Users & System Administrators

1. **Avoid System-Wide Root CA Installation:**
   * Do not import unconstrained national root certificates into primary operating system trust stores (`certmgr.msc` on Windows, System Keychain on macOS, `/usr/local/share/ca-certificates` on Linux, or User/System Credential Storage on Android/iOS).
2. **Environment Compartmentalization (VM / Container Isolation):**
   * Where access to domestic portals requiring certificates chained to `Russian Trusted Root CA` is operationally required, isolate those workflows inside a dedicated, non-persistent Virtual Machine (KVM/QEMU, VirtualBox) or isolated browser container.
   * Never reuse credentials, session tokens, or workspace environments between the isolated domestic-access VM and primary production systems.
3. **Audit & Remediate Existing Trust Stores:**
   * Periodically verify OS trust stores and remove unauthorized root anchors matching SHA-256 fingerprint `d26d2d0231b7c39f92cc738512ba54103519e4405d68b5bd703e9788ca8ecf31`.
4. **Encrypted Transit Encapsulation:**
   * Utilize authenticated, end-to-end encrypted transport tunnels (e.g., WireGuard, IPsec) terminating in trusted jurisdictions when operating over untrusted transit networks.

### 6.2. Defensive Engineering Guidance for Application Developers

1. **Enforce Subject Public Key Info (SPKI) Pinning (OWASP MASVS-NETWORK-2):**
   * Configure mobile and desktop applications to validate specific SPKI SHA-256 hashes (e.g., via Android `network_security_config.xml` `<pin-set>` or iOS `NSPinnedDomains`), ensuring connections reject certificates issued by unexpected root authorities regardless of OS trust store modifications.
2. **Deploy DNS CAA & CT Monitoring:**
   * Publish strict DNS Certification Authority Authorization (`CAA`) records (`0 issue "letsencrypt.org"`, `0 issue "digicert.com"`) and actively monitor Certificate Transparency logs for anomalous certificate issuance.
3. **Implement Application-Layer Payload Cryptography:**
   * For high-sensitivity transactions, supplement TLS transport encryption with application-layer payload encryption and signing (JWE / JWS or mutual TLS with private PKI).

---

**ArmHack-Team Advisory Protocol**  
*Document compiled and verified by `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` under STIRS v2.0.*
