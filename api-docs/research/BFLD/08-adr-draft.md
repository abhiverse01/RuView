# ADR-118: BFLD — Beamforming Feedback Layer for Detection

> This file is a draft. When approved, copy to:
> `docs/adr/ADR-118-bfld-beamforming-feedback-layer-for-detection.md`

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-24 |
| **Deciders** | ruv |
| **Codename** | **BFLD** — Beamforming Feedback Layer for Detection |
| **Relates to** | [ADR-024](ADR-024-contrastive-csi-embedding-model.md) (AETHER contrastive embedding), [ADR-027](ADR-027-cross-environment-domain-generalization.md) (MERIDIAN cross-environment), [ADR-028](ADR-028-esp32-capability-audit.md) (capability audit / witness), [ADR-029](ADR-029-ruvsense-multistatic-sensing-mode.md) (RuvSense multistatic), [ADR-030](ADR-030-ruvsense-persistent-field-model.md) (persistent field model), [ADR-031](ADR-031-ruview-sensing-first-rf-mode.md) (sensing-first RF mode), [ADR-032](ADR-032-multistatic-mesh-security-hardening.md) (mesh security hardening), [ADR-095](ADR-095-rvcsi-edge-rf-sensing-platform.md) (rvCSI platform), [ADR-115](ADR-115-home-assistant-integration.md) (HA integration), [ADR-116](ADR-116-cog-ha-matter-seed.md) (Matter seed packaging), [ADR-117](ADR-117-pip-wifi-densepose-modernization.md) (pip modernization) |
| **Tracking issue** | TBD |

---

## 1. Context

### 1.1 The Plaintext BFI Problem

IEEE 802.11ac and 802.11ax beamforming feedback information (BFI) is exchanged between
client stations (STA) and access points (AP) in unencrypted management-plane frames.
The STA compresses the channel response into a matrix of Givens rotation angles (Phi/Psi)
and transmits them in a VHT/HE Compressed Beamforming Report (CBFR) frame. These frames
are passively sniffable by any device in WiFi monitor mode without any access to the
target network.

Two independent 2024–2025 research papers establish the severity of this exposure:

1. **BFId** (Todt, Morsbach, Strufe; KIT; ACM CCS 2025,
   https://dl.acm.org/doi/10.1145/3719027.3765062): demonstrates re-identification of
   197 individuals using BFI alone, with >90% accuracy from 5 seconds of capture.
2. **LeakyBeam** (Xiao et al.; Zhejiang U., NTU, KAIST; NDSS 2025,
   https://www.ndss-symposium.org/ndss-paper/lend-me-your-beam-privacy-implications-of-plaintext-beamforming-feedback-in-wifi/):
   demonstrates occupancy detection through walls at 20 m range using BFI, with 82.7%
   TPR and 96.7% TNR.

Tooling for passive BFI capture is freely available. Wi-BFI
(https://arxiv.org/abs/2309.04408) is pip-installable and supports 802.11ac/ax,
SU/MU-MIMO, 20/40/80/160 MHz channels.

### 1.2 Gap in Existing Pipeline

The wifi-densepose sensing pipeline processes CSI via the rvCSI runtime (ADR-095/096)
and produces presence, pose, vitals, and zone-activity events. No layer explicitly
measures whether the data being processed is capable of identifying specific individuals.
The pipeline treats all CSI as equivalent from a privacy standpoint, regardless of
whether it is operating in a high-separability (identity-leaky) or low-separability
(anonymous) regime.

This gap becomes a compliance and liability issue as WiFi sensing deployments scale.
An operator deploying this system in a care facility, hotel, or shared office has no
instrument to verify that the system is operating anonymously.

### 1.3 The BFI Opportunity

BFI is not only a threat vector — it is a complementary sensing signal. Because BFI
encodes the channel response as a structured compressed matrix, it carries multipath
geometry that can augment CSI-based presence and motion detection, particularly in
scenarios where only one AP is available (fewer antenna pairs than a full MIMO CSI
capture). The BFLD design treats BFI as an optional input alongside CSI, not as a
replacement.

---

## 2. Decision

We will create a new crate `wifi-densepose-bfld` (to live in `v2/crates/`) that:

1. **Ingests** raw BFI (Phi/Psi angle matrices from CBFR frames) as input and optionally
   fuses CSI when available.
2. **Computes** nine named features and derives an `identity_risk_score` using a
   separability × temporal_stability × cross_perspective_consistency × sample_confidence
   formula.
3. **Gates** all output through a `privacy_class` mechanism that structurally prevents
   identity-correlated data from being published at privacy classes 2 and 3.
4. **Emits** `BfldEvent` structs on MQTT topics under `ruview/<node_id>/bfld/` with
   per-class topic routing.
5. **Enforces** three invariants structurally (not by policy):
   - Raw BFI never exits the node.
   - Identity embedding is in-RAM-only.
   - Cross-site identity correlation is made cryptographically impossible via per-site
     keyed BLAKE3 hash rotation with a daily epoch.

The `BfldFrame` wire format carries magic `0xBF1D_0001`, a version byte, hashed AP/STA
identifiers, a quantization byte, a privacy_class byte, compressed feature payload, and
a CRC32.

Matter exposure is limited to: OccupancySensing (presence), MotionSensor (motion),
PeopleCount (person_count). Identity fields are rejected at the Matter boundary in the
`cog-ha-matter` crate.

---

## 3. Consequences

### Positive

- Operators gain an explicit, auditable measure of privacy compliance at the RF layer —
  the first such primitive in the wifi-densepose ecosystem.
- The identity_risk_score doubles as an anomaly signal: unexpected spikes indicate
  environmental changes (new AP firmware, nearby attacker-grade sniffer, unusual
  propagation geometry) that warrant investigation.
- BFI fusion augments presence and motion accuracy in single-AP deployments, partially
  compensating for lower CSI antenna counts.
- The crate's deterministic frame hashes enable the ADR-028 witness-bundle pattern to
  extend to the new sensing surface, preserving the existing audit trail model.
- Cross-site identity isolation is structural, not policy-dependent. This is a stronger
  guarantee than access-control rules.

### Negative

- BFI capture on ESP32-S3 hardware is not directly possible via the Espressif WiFi API.
  The full BFLD pipeline requires a Pi 5 / Nexmon host-side sniffer (cognitum-v0 is
  available for this purpose, but it adds a fleet dependency for the BFI path).
- The identity_risk_score calibration (correlation with actual re-ID success rate)
  requires the BFId dataset, which requires non-commercial research agreement with KIT.
- ~10.5 engineer-weeks of implementation effort.

### Neutral

- BFLD does not prevent passive BFI capture by an external attacker (A1 / LeakyBeam
  threat). It only ensures the node's own output is non-identifying. Operators should
  be informed of this distinction.
- The daily hash rotation means that occupant-counting analytics that span multiple
  days cannot correlate individual signatures across the day boundary. This is a privacy
  benefit that some analytics use-cases may find inconvenient.

---

## 4. Alternatives Considered

### Alt 1: Skip BFI entirely, CSI-only pipeline

The rvCSI pipeline (ADR-095/096) already handles CSI without BFI. This alternative
requires no new crate and no change to the ESP32 firmware.

**Rejected because**: (a) it leaves the identity-leakage detection gap open for the
existing CSI pipeline, and (b) as BFI capture tooling becomes more widespread (Wi-BFI,
PicoScenes), the absence of a privacy layer becomes more conspicuous for operators.

### Alt 2: Publish identity_risk_score publicly (default-on)

Treat the risk score as a diagnostic metric that operators and the public can observe.

**Rejected because**: the risk score is itself a privacy-sensitive signal (it reveals
when a specific person is present via timing correlation). The default should be
opt-in, with the operator explicitly acknowledging the trade-off.

### Alt 3: Use raw BFI in cloud ML training

Send raw BFI angle matrices to a cloud training service to improve model quality.

**Rejected because**: this violates Invariant 1. Cloud training on raw BFI would
create an off-node store of angle matrices that could be reconstructed into identity
profiles. The on-device-only constraint is not negotiable.

### Alt 4: Differential privacy noise injection on BFI before any processing

Add calibrated Laplace/Gaussian noise to the angle matrices at ingress to provide
epsilon-differential privacy on all downstream computations.

**Rejected for this ADR** (noted as future extension): DP noise calibration requires
sensitivity analysis that is not yet complete, and the interaction between DP noise
and the identity_risk_score formula requires separate validation. The current design
achieves privacy through structural impossibility (local-only, hash rotation) rather
than noise injection.

---

## 5. Acceptance Criteria

- [ ] **AC1**: The extractor parses BFI from commodity WiFi 5 (802.11ac) and WiFi 6
  (802.11ax) captures, supporting 20/40/80/160 MHz channel bandwidth and 2×2 through
  4×4 MIMO configurations.
- [ ] **AC2**: Presence detection latency is ≤ 1s p95 from the first non-empty BFI
  frame in a new occupancy event.
- [ ] **AC3**: Motion score is published at ≥ 1 Hz on the `ruview/<node_id>/bfld/motion/state`
  MQTT topic during sustained occupancy.
- [ ] **AC4**: Raw BFI bytes (Phi/Psi angle matrices) are never present in any
  serialized `BfldFrame` payload at any `privacy_class` value.
- [ ] **AC5**: When `privacy_mode` is enabled, all identity-derived fields
  (`identity_risk_score`, `rf_signature_hash`, `identity_embedding`) are absent from
  all outbound events.
- [ ] **AC6**: Given identical `BfiCapture` inputs, the `BfldFrame` serialization
  produces bit-identical output (deterministic hash) across runs and across platforms.
- [ ] **AC7**: The pipeline produces valid `BfldEvent` outputs when `csi_matrix` is
  absent (BFI-only mode), without panic or degraded presence/motion reporting beyond
  the documented accuracy bounds.

---

## 6. Related ADRs

- **ADR-024**: AETHER contrastive CSI embedding — BFLD reuses the AETHER embedding
  infrastructure for identity_risk computation.
- **ADR-027**: MERIDIAN cross-environment — BFLD's cross-site isolation instantiates
  the "no cross-site correlation" assumption that MERIDIAN requires.
- **ADR-028**: Witness verification — BFLD extends the deterministic proof pattern.
- **ADR-029**: RuvSense multistatic — BFLD uses `multistatic.rs` for
  cross_perspective_consistency.
- **ADR-030**: Persistent field model — BFLD uses `cross_room.rs` for
  environment fingerprinting in the hash rotation.
- **ADR-031**: Sensing-first RF mode — BFLD is a new sensing primitive alongside
  the CSI-based sensing.
- **ADR-032**: Mesh security hardening — BFLD's threat model is a superset.
- **ADR-095/096**: rvCSI platform — BFLD shares the BFI capture path with rvCSI's
  Nexmon adapter.
- **ADR-115**: HA integration — BFLD extends the 21-entity HA surface with 6 new
  entities.
- **ADR-116**: Matter seed packaging — BFLD's Matter boundary filter is implemented
  in `cog-ha-matter`.
- **ADR-117**: pip modernization — BFLD's Python bindings (PyO3) will follow the
  pattern established in ADR-117.
