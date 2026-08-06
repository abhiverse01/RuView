# ADR-118: BFLD — Beamforming Feedback Layer for Detection

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-24 |
| **Deciders** | ruv |
| **Codename** | **BFLD** — Beamforming Feedback Layer for Detection |
| **Relates to** | [ADR-024](ADR-024-contrastive-csi-embedding-model.md) (AETHER), [ADR-027](ADR-027-cross-environment-domain-generalization.md) (MERIDIAN), [ADR-028](ADR-028-esp32-capability-audit.md) (witness), [ADR-029](ADR-029-ruvsense-multistatic-sensing-mode.md) (multistatic), [ADR-030](ADR-030-ruvsense-persistent-field-model.md) (field model), [ADR-031](ADR-031-ruview-sensing-first-rf-mode.md) (sensing-first), [ADR-032](ADR-032-multistatic-mesh-security-hardening.md) (mesh security), [ADR-095](ADR-095-rvcsi-edge-rf-sensing-platform.md) (rvCSI), [ADR-115](ADR-115-home-assistant-integration.md) (HA), [ADR-116](ADR-116-cog-ha-matter-seed.md) (Matter), [ADR-117](ADR-117-pip-wifi-densepose-modernization.md) (pip) |
| **Sub-ADRs** | [ADR-119](ADR-119-bfld-frame-format-and-wire-protocol.md) (frame), [ADR-120](ADR-120-bfld-privacy-class-and-hash-rotation.md) (privacy), [ADR-121](ADR-121-bfld-identity-risk-scoring.md) (risk), [ADR-122](ADR-122-bfld-ruview-ha-matter-exposure.md) (RuView), [ADR-123](ADR-123-bfld-capture-path-nexmon-and-esp32.md) (capture) |
| **Research bundle** | [`docs/research/BFLD/`](../research/BFLD/) (11 files, 13,544 words) |
| **Companion research** | [`docs/research/soul/`](../research/soul/) — Soul Signature multi-modal biometric. BFLD is the policy-enforcement and compliance layer for Soul Signature; the two share the AETHER encoder (ADR-024), the witness chain (ADR-110/028), the RVF container, and `cross_room.rs` (ADR-030). |
| **Tracking issue** | TBD |

---

## 1. Context

### 1.1 The plaintext BFI problem

IEEE 802.11ac and 802.11ax beamforming feedback (BFI) is exchanged between client stations (STA) and access points (AP) in **unencrypted management-plane frames**. The STA compresses the channel response into a Givens-rotation angle matrix (Φ/ψ) and transmits it as a VHT/HE Compressed Beamforming Report (CBFR). Any device in WiFi monitor mode within range can passively sniff these frames without joining the network.

Two independent 2024–2025 research results establish the severity of this exposure:

1. **BFId** (KIT, ACM CCS 2025) — re-identifies 197 individuals from BFI alone with >90% accuracy from 5 s of capture. https://publikationen.bibliothek.kit.edu/1000185756
2. **LeakyBeam** (NDSS 2025) — detects occupancy through walls at 20 m with 82.7% TPR / 96.7% TNR using only plaintext BFI. https://www.ndss-symposium.org/wp-content/uploads/2025-5-paper.pdf

Capture tooling is freely available: **Wi-BFI** (pip-installable), **PicoScenes**, **Nexmon BFI patches** for BCM43455c0 (Raspberry Pi 5 / 4 / 3B+).

### 1.2 Gap in the existing RuView pipeline

The wifi-densepose / RuView pipeline processes CSI via the rvCSI runtime (ADR-095/096) and emits presence, pose, vitals, and zone-activity events. **No layer in the existing pipeline measures whether the data it is processing is capable of identifying individuals.** All CSI is treated as equivalent from a privacy standpoint regardless of operating regime.

This gap becomes a compliance and liability issue at deployment scale. An operator placing RuView in a care home, hotel, shared office, or rental property has no instrument to verify that the system is operating anonymously.

### 1.3 BFI as a sensing signal

BFI is not only a threat vector — its compressed angle matrices carry multipath geometry useful for presence and motion detection, particularly in single-AP deployments where MIMO CSI is unavailable. BFLD treats BFI as an **optional input alongside CSI**, not a replacement.

### 1.4 Relationship to the Soul Signature research

The Soul Signature research (`docs/research/soul/`) defines a 7-channel multi-modal biometric for **consent-based** passive re-identification of enrolled individuals. Where Soul Signature *intentionally produces* identity (with a 60-second enrollment protocol), BFLD *measures and gates* identity leakage from the same sensing substrate. The two systems are complementary by design:

| Concern | Soul Signature | BFLD |
|---------|----------------|------|
| Intent | Create a biometric for enrolled persons | Measure and gate identity leakage |
| Consent model | Explicit enrollment, GDPR/HIPAA modes | Default-deny, all unenrolled persons |
| Operating class | Must run at `privacy_class = 1` (derived) | Defaults to class 2 (anonymous) |
| Shared assets | AETHER encoder (ADR-024), WitnessChain (ADR-110/028), RVF container, `cross_room.rs` (ADR-030) | Same |
| ID space | Long-lived opaque `person_id` per enrolled subject | Rotating `rf_signature_hash` per day per unenrolled person |

BFLD becomes Soul Signature's enforcement layer: the `identity_risk_score` gates whether a zone is leaky enough to enroll, the witness bundle is the regulator-facing audit artifact, and the structural privacy invariants (I1/I2/I3) ensure unenrolled bystanders stay anonymous even in zones where Soul Signature is actively matching enrolled persons. See ADR-120 §2.7 and ADR-121 §2.7 for the integration points.

### 1.5 What this ADR is *not*

- Not a removal of the CSI pipeline. ADR-095/096 rvCSI stays authoritative for CSI.
- Not a port of any external sniffer into the repo. The Nexmon capture path lives in a separate adapter (see ADR-123).
- Not a Matter SDK ship — Matter exposure is filtered through the ADR-116 `cog-ha-matter` boundary.

---

## 2. Decision

Create a new Rust crate **`wifi-densepose-bfld`** in `v2/crates/` that:

1. **Ingests** BFI angle matrices (Φ/ψ) from CBFR frames, optionally fused with CSI.
2. **Computes** nine named features and an `identity_risk_score` (separability × temporal_stability × cross_perspective_consistency × sample_confidence).
3. **Gates** all output through a `privacy_class` byte that **structurally prevents** identity-correlated data from being published at classes 2 (anonymous) and 3 (restricted).
4. **Emits** `BfldEvent` JSON over MQTT under `ruview/<node_id>/bfld/*` with per-class topic routing.
5. **Enforces three invariants structurally, not by policy**:
   - **I1**: Raw BFI never exits the node.
   - **I2**: Identity embedding is in-RAM-only (no disk, no network).
   - **I3**: Cross-site identity correlation is cryptographically impossible via per-site keyed BLAKE3 hash rotation with a daily epoch.

The umbrella implementation is decomposed into five sub-ADRs:

| Sub-ADR | Scope |
|---------|-------|
| **ADR-119** | `BfldFrame` wire format, magic `0xBF1D_0001`, deterministic serialization, CRC32 |
| **ADR-120** | `privacy_class` semantics, BLAKE3 hash rotation, default-deny field classification |
| **ADR-121** | Identity risk scoring formula, coherence gate, leakage estimator |
| **ADR-122** | RuView surface: HA entities, Matter cluster boundary, MQTT topic ACL |
| **ADR-123** | Capture path: Pi 5 / Nexmon adapter + ESP32-S3 BFI feasibility |

### 2.1 Crate module layout

```
v2/crates/wifi-densepose-bfld/
├── Cargo.toml
└── src/
    ├── lib.rs
    ├── frame.rs             # BfldFrame (ADR-119)
    ├── extractor.rs         # CBFR parser → BfiCapture
    ├── features.rs          # 9 features
    ├── identity_risk.rs     # risk score (ADR-121)
    ├── privacy_gate.rs      # privacy_class enforcement (ADR-120)
    ├── hash_rotation.rs     # BLAKE3 per-site rotation (ADR-120)
    ├── emitter.rs           # BfldEvent → MQTT
    ├── mqtt.rs              # topic routing (ADR-122)
    └── ffi.rs               # PyO3 bindings (ADR-117 pattern)
```

### 2.2 Reuse map

| BFLD module | Depends on |
|---|---|
| `features.rs` | `wifi-densepose-signal/src/ruvsense/coherence.rs`, `multistatic.rs` |
| `identity_risk.rs` | `wifi-densepose-ruvector/src/viewpoint/attention.rs`, `coherence.rs` |
| `privacy_gate.rs` | (new) — no upstream dependency |
| `hash_rotation.rs` | `blake3 = "1.5"` (keyed mode) |
| `extractor.rs` | `vendor/rvcsi/crates/rvcsi-adapter-nexmon` (ADR-095/096) |

---

## 3. Consequences

### Positive

- First explicit, auditable RF-layer privacy primitive in the wifi-densepose ecosystem.
- `identity_risk_score` doubles as an anomaly signal (sudden spike → new AP firmware / nearby attacker-grade sniffer / unusual propagation).
- BFI fusion augments presence/motion in single-AP deployments.
- Deterministic frame hashes extend the ADR-028 witness-bundle pattern to the new surface.
- Cross-site isolation is **structural, not policy-dependent** — a stronger guarantee than ACLs.

### Negative

- ESP32-S3 cannot directly capture CBFR via the Espressif WiFi API. Full BFLD pipeline requires a Pi 5 / Nexmon host sniffer (cognitum-v0 available; see ADR-123).
- `identity_risk_score` calibration requires the KIT BFId dataset (non-commercial research agreement).
- Estimated effort: ~10.5 engineer-weeks across the six ADRs.

### Neutral

- BFLD does not prevent passive BFI capture by an external attacker (LeakyBeam-class). It only ensures the **node's own output** is non-identifying. Operators must understand this distinction.
- Daily hash rotation prevents multi-day analytics correlating individual signatures across the day boundary. Acceptable for privacy goals; may surprise analytics use-cases.

---

## 4. Alternatives Considered

### Alt 1: Skip BFI entirely (CSI-only)

Rejected because: (a) leaves the identity-leakage gap open for the CSI pipeline; (b) as BFI tooling becomes ubiquitous (Wi-BFI, PicoScenes), the absence of a privacy layer becomes more conspicuous for operators.

### Alt 2: Publish `identity_risk_score` publicly by default

Rejected: the risk score itself is privacy-sensitive (reveals presence via timing correlation). Default is opt-in.

### Alt 3: Cloud ML on raw BFI

Rejected: violates I1. Cloud training creates an off-node store of angle matrices reconstructible into identity profiles.

### Alt 4: Differential privacy noise on BFI at ingress

Deferred to a follow-up ADR. DP sensitivity analysis and its interaction with `identity_risk_score` calibration are not yet complete. Current design achieves privacy through structural impossibility, not noise injection.

---

## 5. Acceptance Criteria

- [ ] **AC1**: Extractor parses BFI from 802.11ac and 802.11ax captures, 20/40/80/160 MHz, 2×2 through 4×4 MIMO.
- [ ] **AC2**: Presence detection latency ≤ 1 s p95 from first non-empty BFI frame.
- [ ] **AC3**: Motion score published at ≥ 1 Hz on `ruview/<node_id>/bfld/motion/state`.
- [ ] **AC4**: Raw BFI bytes never present in any serialized `BfldFrame` payload at any `privacy_class` value.
- [ ] **AC5**: With `privacy_mode` enabled, all identity-derived fields are absent from outbound events.
- [ ] **AC6**: Identical `BfiCapture` inputs produce bit-identical `BfldFrame` serialization (deterministic hash).
- [ ] **AC7**: Pipeline produces valid `BfldEvent` outputs without `csi_matrix` (BFI-only mode).

Per-sub-ADR acceptance criteria are defined in ADR-119 through ADR-123.

---

## 6. Phased Rollout

| Phase | ADR | Scope | Effort |
|-------|-----|-------|--------|
| **P1** | 119 | Frame format + extractor stub | 1.5 wk |
| **P2** | 121 | Features + identity_risk_score | 2.0 wk |
| **P3** | 120 | Privacy gate + hash rotation | 1.5 wk |
| **P4** | 122 (a) | MQTT emitter + HA discovery | 1.5 wk |
| **P5** | 122 (b) | Matter cluster boundary in `cog-ha-matter` | 1.5 wk |
| **P6** | 123 | Pi 5 / Nexmon capture adapter | 2.5 wk |
| **Total** | | | **10.5 wk** |

---

## 7. Related ADRs

See header table. Cross-references in body cite the structural reuse of:
- ADR-024 (AETHER embedding for identity_risk computation)
- ADR-027 (MERIDIAN's no-cross-site assumption is now structurally enforced by I3)
- ADR-028 (witness-bundle extends to BFLD surface)
- ADR-029/030 (`multistatic.rs`, `cross_room.rs` reused)
- ADR-095/096 (rvCSI Nexmon adapter for BFI capture)
- ADR-115 (HA surface extension)
- ADR-116 (`cog-ha-matter` boundary filter)
- ADR-117 (PyO3 bindings pattern)
