# Soul Signature — Security, Privacy, and Threat Model

**Status:** Research Specification (Pre-Implementation)
**Date:** 2026-05-24
**Author:** ruv

---

## 1. Scope

This document defines the threat model, mitigations, cryptographic primitive
choices, privacy architecture, and open security research items for the Soul
Signature system. It is intended to be reviewed by a security engineer or
privacy counsel before any production deployment.

The soul signature is a passive biometric system. The security bar is:
**attacker cost to achieve a false accept must exceed the value of the
protected resource for the relevant threat model**. The soul signature does
not claim to be unbreakable. It claims to be hard enough.

---

## 2. What We Explicitly Do NOT Claim

- Not equal to fingerprint scanners on FBI-tier datasets in EER terms. RF
  biometrics are a younger discipline. No independent benchmark with the soul
  signature's specific multi-channel fusion exists yet.
- Not legal evidence. Passive RF biometric identification has no established
  legal precedent in any jurisdiction.
- Not a replacement for explicit consent in regulated contexts (healthcare,
  employment, border control).
- Not unbreakable under a nation-state adversary with full physical access to
  the sensing infrastructure.
- Not validated at scale beyond the constituent ADR baselines. The AETHER
  channel (ADR-024) targets >80% mAP at 5 subjects; at 100+ subjects the
  false-accept rate is open research.

---

## 3. Threat Model

### 3.1 Attacker: Passive Eavesdropper on the WiFi Medium

**Capability:** An attacker near the WiFi sensing zone can observe CSI of any
person who passes through. With enough CSI, the attacker could construct an
unauthorized soul signature enrollment of an unconsenting bystander.

**Impact:** Unauthorized enrollment → unauthorized recognition → attribution of
presence to a person who did not consent.

**Mitigation:**
- Ambient CSI capture does NOT trigger enrollment. Enrollment requires the
  explicit 60-second structured protocol. Ambient bystander CSI produces
  `unauthenticated` pose tracks tagged as `person_id: NULL`.
- Unauthenticated RVF nodes are pruned from the HNSW index after 24 hours.
- The enrollment protocol requires presence confirmation from at least two
  sensing nodes simultaneously, making drive-by enrollment geometrically
  harder to achieve without physical proximity.

**Residual risk:** An attacker who can be physically present in the scanning
zone for 60 seconds, under the observation of the scanning protocol, can cause
enrollment of a fake person. This requires physical co-location and is
equivalent to the threat model for any in-person biometric registration.

### 3.2 Attacker: Active Replay

**Capability:** An attacker records a CSI stream from a legitimate enrollment
or recognition event and replays it to a sensing node to impersonate the
enrolled person.

**Impact:** False positive recognition; unauthorized access or presence attribution.

**Mitigation:**
- Each enrollment is bound to the room's ADR-030 field model eigenstate at
  enrollment time. The `environment_id` field in every vector node is a
  SHA-256 of the field model's eigenmode matrix. A replay in a different room
  produces a different `environment_id` and a dramatically different
  Subcarrier_Reflection_Profile — the cross-validation between these two
  signed fields fails.
- The Ed25519 witness chain (ADR-110) includes a monotonic timestamp
  (`timestamp_ns`). A replay of an old signature is detected by the timestamp
  freshness check at recognition time (configurable; default: reject any
  signature older than 7 days for high-assurance contexts).
- The ADR-030 field model continuously updates. Even if the replay is in the
  same room, the field model's eigenstate changes as furniture is moved or
  temperature shifts the propagation medium; cross-validation degrades over
  time.

**Residual risk:** Replay within the same room within a short time window
(< 4 hours, before the field model rotates) by an attacker who has recorded the
original CSI with high fidelity remains a plausible attack vector. This is not
defended against by the current architecture. It requires a future ADR for
challenge-response liveness detection.

### 3.3 Attacker: Phased-Array Vest / RF Body Emulator

**Capability:** An attacker wears a device capable of emitting RF signals that
mimic another person's backscatter profile, allowing them to be recognized as
the enrolled person.

**Impact:** The strongest impersonation attack; if successful, bypasses all
electromagnetic biometric channels simultaneously.

**Mitigation:**
- The RuvSense `adversarial.rs` module (ADR-030 Tier 7) enforces four
  physics-based consistency checks:
  1. Multi-link consistency: a real body perturbs all mesh links passing
     through its location. A vest emitting signals affects only the targeted
     link(s). Detection: at least 4 links must show correlated perturbation.
  2. Field model constraints: the perturbation must lie within the span of
     the room's eigenmode structure. Artificially injected signals produce
     perturbations inconsistent with room geometry.
  3. Temporal continuity: real movement is smooth in embedding space; injected
     signals can produce discontinuities flagged by the embedding velocity
     monitor.
  4. Energy conservation: total perturbation energy across all links must be
     consistent with the number and geometry of bodies present.
- The adversarial detector fires `FAIL_ADVERSARIAL_SIGNAL` before the soul
  signature match is considered.

**Residual risk:** A sophisticated attacker with a calibrated phased-array
system who also knows the room's eigenmode structure and the enrolled person's
exact multi-link backscatter pattern could in principle construct a convincing
emulation. This is a high-capability, high-cost attack. Practical countermeasure:
require multi-node confirmation (ADR-029 multistatic) which raises the
geometric complexity of the emulation exponentially with node count.

### 3.4 Attacker: Insider with Broker Access

**Capability:** A privileged operator or compromised service with read access
to the stored `.rvf` files and the HNSW person_track index.

**Impact:** Exfiltration of biometric signatures; linkage of person_id to PII
if linkage tables also accessible; replay or cross-site re-enrollment.

**Mitigation:**
- At-rest encryption: all `.rvf` files are encrypted with ChaCha20-Poly1305
  using a key derived via Argon2id from a user-provided passphrase (or a FIDO2
  hardware token binding). The Cognitum Seed appliance NEVER stores the
  decryption key; it is re-derived from the passphrase on each access.
- The opaque `person_id` (u64) in the `.rvf` file is not PII. PII linkage, if
  any, requires access to a separate application-layer database not stored on
  the sensing appliance.
- The HNSW index stores only the 128-dim AETHER embedding, not raw CSI or full
  soul signatures. Exfiltration of the index exposes the embedding but not the
  full biometric record.
- Differential privacy (ADR-106 DP-SGD) applies at training time when AETHER
  is fine-tuned on enrolled-person data, preventing membership inference attacks
  that could recover training samples from model weights.

**Residual risk:** If the passphrase is weak or the FIDO2 token is compromised,
the at-rest encryption fails. Key management is a deployment responsibility.

### 3.5 Attacker: Manufacturer / Firmware Supply Chain

**Capability:** A malicious firmware update to the ESP32 node or Cognitum Seed
appliance could silently exfiltrate soul signatures or CSI streams.

**Impact:** Large-scale passive surveillance; biometric data exfiltration across
all installed appliances.

**Mitigation:**
- All firmware releases are signed with Ed25519 (ADR-100 cog packaging) and
  verified by the appliance before installation. A Dilithium-3 post-quantum
  co-signature is added in the transition window (ADR-109).
- The Ed25519 witness chain (ADR-110) signs each CSI frame bundle at the
  sensor level. A firmware change that alters the witness chain is detectable
  by downstream audit.
- Network egress from the Cognitum Seed in `--privacy-mode` is blocked for
  raw CSI and soul signatures by default. Only MQTT auto-discovery messages
  (ADR-115) and OTA metadata are permitted outbound.
- Open-source firmware. The ESP32 firmware and Cognitum Seed Rust crates are
  open source (this repository). Independent audit is possible.

**Residual risk:** A zero-day exploit in the ESP-IDF WiFi stack or the Rust
codebase could bypass these controls. This is mitigated by regular security
audits (run `npx @claude-flow/cli@latest security scan` per CLAUDE.md) but not
eliminated.

---

## 4. Consent Architecture

### 4.1 The Enrollment-vs-Recognition Distinction

The soul signature system enforces a hard distinction:

| Action | Consent required | Mechanism |
|---|---|---|
| Enrollment | Explicit, active | 60-second protocol with operator confirmation; produces signed `.rvf` |
| Recognition of enrolled person | Implicit (enrollment = consent for recognition) | Continuous mode; HNSW match |
| Ambient sensing of unenrolled person | No — but data is transient and pruned | Unauthenticated tracks; 24h TTL |
| Updating stored profile from continuous mode | Implicit (set at enrollment time) | Aggregator auto-refresh; configurable |

The system operator is responsible for obtaining appropriate consent from
persons before performing enrollment. The technical system enforces that
enrollment cannot happen accidentally or from drive-by sensing.

### 4.2 Bystander Protection

Persons who pass through a sensing zone without being enrolled are sensed but
not persistently identified. Their data flow:
1. Pose tracker produces a track tagged `person_id: NULL`.
2. AETHER embedding is computed for motion detection and occupancy counting
   (ADR-115 HA-MIND).
3. The embedding is written to the `temporal_baseline` HNSW index with a 24-hour
   TTL and `authenticated: false`.
4. After 24 hours, the entry is automatically pruned by the `EmbeddingIndex::prune()`
   method (ADR-024 §2.4).
5. No `.rvf` file is created. No persistent record exists.

This architecture satisfies the GDPR principle of data minimization (Article 5(1)(c))
for bystander data: the retention period is bounded, the data is not linked to
an identity, and the storage is proportionate to the functional purpose
(occupancy counting).

### 4.3 GDPR / HIPAA Mode

When `--privacy-mode enabled` (from ADR-115 HA-MIND §privacy):

1. Soul signatures are computed and stored locally only. They are NEVER
   published to MQTT topics, Matter clusters, or any external endpoint.
2. The local REST API for accessing soul signatures requires a valid bearer
   token (ADR-028 bearer_auth.rs). No unauthenticated endpoint exposes
   biometric data.
3. The JSON-LD sidecar is written to the local encrypted store only. It is not
   included in MQTT auto-discovery payloads.
4. The longitudinal drift metrics (ADR-030 Tier 4) are published to MQTT in
   aggregated form only (e.g., `drift_detected: true`, never raw metric values
   that could be used for medical inference).
5. A data deletion endpoint must be implemented: `DELETE /api/v1/persons/{id}`
   removes the `.rvf` file, the HNSW index entry, the JSON-LD sidecar, and all
   longitudinal Welford statistics for that person_id.

---

## 5. Cryptographic Primitives

All primitives are chosen from NIST-approved or widely-audited standards.

| Purpose | Primitive | Rationale |
|---|---|---|
| Content integrity (per-segment) | CRC32 (IEEE 802.3) | Already implemented in `rvf_container.rs:line 70`. Corruption detection, not security. |
| Content addressing | SHA-256 | File name derivation; pre-image resistance prevents name collisions |
| Ed25519 signatures | Ed25519 (RFC 8032) | ADR-110 witness chain; 64-byte signatures; 128-bit security |
| At-rest encryption | ChaCha20-Poly1305 (RFC 8439) | AEAD; software-friendly; no timing-attack surface like AES-CBC; 256-bit key |
| Key derivation from passphrase | Argon2id (RFC 9106) | Memory-hard KDF; resistant to GPU/ASIC brute-force; recommended by NIST SP 800-132 draft (2024) |
| DP-SGD noise | Gaussian N(0, σ²C²I) per ADR-106 | (ε, δ)-DP per Abadi et al. 2016 Moments Accountant |
| Post-quantum key exchange (future) | Kyber-768 (NIST FIPS 203, 2024) | ADR-108; ~AES-192 security; NIST CNSA 2.0 recommended |
| Post-quantum signatures (future) | Dilithium-3 (NIST FIPS 204, 2024) | ADR-109; hybrid mode with Ed25519 during transition window |

### 5.1 Argon2id Parameters

Default parameters for soul signature key derivation:

```
m_cost = 65536 (64 MB memory)
t_cost = 3     (3 iterations)
p_cost = 4     (4 parallel lanes)
output_len = 32 bytes (256-bit key for ChaCha20-Poly1305)
salt = 16 random bytes stored alongside encrypted blob (NOT the person_id)
```

These parameters provide ~100ms KDF time on a Pi 5, which is acceptable for
enrollment (one-time) and recognition (HNSW match precedes decryption, so
decryption is only triggered after a candidate match).

### 5.2 Forward Secrecy

Old soul signature files are NOT keys for new ones. Compromise of a 90-day-old
`.rvf` file does not unlock the current profile. The key is derived from the
user's passphrase each time, not derived from the previous file.

Archived files (kept for audit purposes) are re-encrypted on passphrase rotation
if the operator elects to do so via the `soul-signature re-encrypt --all` CLI
command (not yet implemented; specified here for future ADR).

---

## 6. Privacy Mode Integration (ADR-115)

The `--privacy-mode` flag defined in ADR-115 HA-MIND §9 is extended to cover
soul signature data:

| Privacy mode | MQTT publish | REST API | Local storage | HNSW index |
|---|---|---|---|---|
| `disabled` (default for home users) | Aggregated presence/count only | Authenticated bearer required | Encrypted at rest | Local only |
| `enabled` | Nothing biometric | Authenticated bearer required | Encrypted at rest | Local only |
| `research` (explicit opt-in) | Full soul signature nodes (anonymized person_id) | Open (for research deployments only) | Encrypted at rest | Exportable |

The `research` mode requires a separate `--research-consent-token` flag and is
intended for academic data collection under IRB approval. It must never be the
default.

---

## 7. Open Research and Outstanding Security Work

The following items are known security gaps or open research questions. Each
warrants a future ADR before production deployment at scale.

**7.1 Challenge-Response Liveness Detection**
Replay attacks within a short time window (see §3.2 residual risk) are not
defended against. A future mechanism should issue a random challenge (e.g.,
"please raise your left hand") and verify the CSI response matches the challenge
before accepting a recognition. This eliminates replay as a practical attack
vector. Future ADR: ADR-120 (proposed).

**7.2 False-Accept Rate at Scale (N > 20 subjects)**
The AETHER baseline (ADR-024) is tested at 5 subjects (>80% mAP). For household
deployments this is sufficient. For building-scale deployments (50-500 subjects),
the FAR is open research. Independent benchmarking on a dataset of 20+ subjects
with the full 7-channel fusion is required before building-scale deployment can
be recommended. Publication target: co-locate with ADR-027 MERIDIAN evaluation.

**7.3 Side-Channel Leakage from Encrypted RVF Files**
The file size of an encrypted `.rvf` blob is observable by an attacker with
filesystem access. File size is a function of the number of nodes present, which
reveals whether the cardiac channel was captured (high-SNR enrollment vs
low-SNR enrollment). This is a minor information leak. Mitigation: pad all
`.rvf` files to a fixed 64 KB boundary. Future ADR: append to ADR-106.

**7.4 Membership Inference in Continuous Mode**
In continuous mode, the AETHER model is fine-tuned on the enrolled person's
data over months. An adversary with access to the model weights before and after
a re-train cycle could infer that a specific enrollment occurred, even without
the soul signature file, via membership inference (Shokri et al. 2017).
ADR-106 DP-SGD mitigates this for federation round deltas but not for local
single-device fine-tuning. Extension of DP-SGD to the local continuous-mode
update is required. Future ADR: extend ADR-106.

**7.5 Physical Access to Sensing Nodes**
An attacker with physical access to an ESP32 node can extract the firmware and
attempt to reverse the Ed25519 signing key (if the key is stored in ESP32
NVS without protection). ADR-110 uses NVS for key storage. A future ADR should
mandate secure element storage (e.g., ATECC608A co-processor on the Cognitum
Seed) for the signing key. Future ADR: ADR-121 (proposed).

**7.6 Federated Learning Linkability**
When AETHER is retrained via federated learning (ADR-105), the LoRA weight
deltas carry information about enrolled persons. ADR-106 applies DP-SGD to
these deltas, but the post-quantum migration path (ADR-108 Kyber-768) is not
yet integrated with the federation protocol. Until ADR-108 Phase 2 ships, the
federation link is classically encrypted and vulnerable to harvest-now-decrypt-later
attacks by quantum-capable adversaries. Assessed risk: low until 2027.

---

## 8. Summary Security Properties Table

| Property | Status | Evidence |
|---|---|---|
| At-rest encryption | Specified (ChaCha20-Poly1305 + Argon2id) | This document §5 |
| Ed25519 attestation | Implemented | ADR-110 witness chain |
| Replay resistance (cross-room) | Implemented | ADR-030 field model environment_id binding |
| Replay resistance (same-room, short window) | Open gap | §7.1 |
| Anti-spoofing (single-link injection) | Implemented | adversarial.rs multi-link consistency |
| Anti-spoofing (phased-array vest) | Partial | adversarial.rs + energy conservation; residual risk documented |
| Bystander protection | Specified | 24h TTL on unauthenticated tracks; §4.2 |
| DP-SGD training privacy | Implemented (federation) | ADR-106 |
| DP-SGD training privacy (local continuous mode) | Open gap | §7.4 |
| GDPR data deletion | Specified | §4.3 `DELETE /api/v1/persons/{id}` |
| Post-quantum migration path | Specified (Kyber-768, Dilithium-3) | ADR-108, ADR-109 |
| Firmware supply chain integrity | Implemented (Ed25519 cog signing) | ADR-100, ADR-109 hybrid |
| False-accept rate at scale | Open research | §7.2 |
| Liveness detection | Open gap | §7.1 |
| Secure element key storage | Open gap | §7.5 |
