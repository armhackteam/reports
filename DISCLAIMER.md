> 🌐 **Language / Язык:** **🇬🇧 English** | [🇷🇺 Русский](./DISCLAIMER.ru.md)

# ArmHack-Team — Legal & Ethical Research Disclaimer (Safe Harbor Framework)

**Document Standard:** STIRS v2.0 (Standardized Technical Information & Risk Specification)  
**Authoring Body:** `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)`

---

## 1. Purpose and Nature of Publications

All technical security assessment reports, cryptographic evaluations, and architectural advisories published in this repository constitute independent academic, cryptographic, and defensive engineering research. They are compiled and published exclusively for educational, informational, privacy-awareness, and threat-mitigation purposes.

All technical conclusions presented within this repository are derived strictly from:
1. Publicly available international technical standards and open specifications (including but not limited to IETF RFC 5280, RFC 8446, RFC 6962, RFC 9162, CA/Browser Forum Baseline Requirements, OWASP MASVS / MASTG, and NIST SP 800-52 Rev. 2).
2. Publicly auditable binary artifacts, X.509 certificate chains, open-source repositories, and cryptographic parameters.
3. Observable client-side runtime telemetry and protocol behavior captured within controlled, isolated laboratory environments.

---

## 2. Architectural Capability vs. Attribution of Intent

**No Assertion of Unlawful Conduct or Malicious Intent:**  
`ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` does **not** state, imply, accuse, or assert that any evaluated vendor, software developer, telecommunications operator, certificate authority, or state entity is actively engaging in unlawful mass surveillance, unauthorized data exfiltration, criminal conduct, or malicious exploitation.

In accordance with defensive systems engineering and formal threat modeling methodologies:
- Every finding is documented strictly as an **inherent architectural capability**, **structural threat vector**, **cryptographic trust-boundary condition**, or **technical discrepancy between declared policy and observable protocol mechanics**.
- Evaluating what a system is *mathematically or architecturally capable of performing* (e.g., on-path TLS termination when a root anchor lacking `NameConstraints` is installed in an operating system trust store) describes an objective property of the cryptographic topology, not an accusation regarding the operational motives of the key holder.

---

## 3. Safe Harbor & Defensive Research Ethics

All assessments conducted by `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` adhere to strict defensive research ethics:
- **Zero Disruption:** No active exploitation, denial-of-service, or unauthorized intrusion against third-party production infrastructure is ever performed during testing.
- **Data Sanitization:** All captured protocol headers, network payloads, session tokens, device identifiers, and laboratory IP addresses are systematically scrubbed and replaced with `[REDACTED]` placeholders prior to publication.
- **Reproducibility & Verification:** Artifact hashes (`SHA-256`), build identifiers, and standard specifications are documented to enable independent verification by security engineers and academic peers.

---

## 4. Limitation of Liability & No Warranty

The information in this repository is provided **"AS IS"**, without warranty of any kind, express or implied, including but not limited to warranties of completeness, accuracy, fitness for a particular purpose, or non-infringement. In no event shall `ArmHack-Team (Cybersecurity Assessment & Defensive Engineering Unit)` be liable for any direct, indirect, incidental, special, exemplary, or consequential damages arising out of the use or inability to use the technical analyses or hardening recommendations contained herein.

---

## 5. Trademarks & Intellectual Property

All product names, package identifiers, logos, brands, and registered trademarks mentioned in these reports remain the sole property of their respective owners. Their inclusion is strictly nominative and used solely for technical identification of the evaluated software or infrastructure artifacts.
