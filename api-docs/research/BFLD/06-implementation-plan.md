# BFLD Implementation Plan

## 1. New Crate: wifi-densepose-bfld

Location: `v2/crates/wifi-densepose-bfld/`

This crate slots between `wifi-densepose-signal` (BFI normalization, temporal windowing)
and `wifi-densepose-sensing-server` (MQTT/HA integration). It does not depend on the
training pipeline (`wifi-densepose-train`) or the neural-network inference crate
(`wifi-densepose-nn`) in the default build — feature flags activate those paths.

### 1.1 Module Layout

```
v2/crates/wifi-densepose-bfld/
    Cargo.toml
    src/
        lib.rs              # Public API: BfldPipeline, BfldFrame, BfldEvent
        frame.rs            # BfldFrame struct, serialization, CRC32, magic bytes
        extractor.rs        # BFI packet capture interface, Phi/Psi parsing,
                            #   802.11ac/ax CBFR format decoder
        features.rs         # Feature computation: mean_angle_delta,
                            #   subcarrier_variance, temporal_entropy,
                            #   doppler_proxy, path_stability,
                            #   cross_antenna_correlation, burst_motion_score,
                            #   stationarity_score, identity_separability_score
        identity_risk.rs    # identity_risk_score formula, EmbeddingRingBuf,
                            #   in-RAM-only lifecycle enforcement
        privacy_gate.rs     # privacy_class assignment, field masking,
                            #   #[must_classify] lint check
        emitter.rs          # BfldEvent construction, JSON serialization
        mqtt.rs             # MQTT topic publishing, ACL, per-class topic routing
    tests/
        frame_roundtrip.rs  # BfldFrame serialization + CRC32 determinism
        privacy_gate.rs     # Per-class field suppression assertions
        hash_rotation.rs    # Cross-site isolation + daily rotation proofs
        identity_risk.rs    # Risk score bounded [0,1], local-only embedding
        acceptance.rs       # All 7 acceptance criteria as named tests
    benches/
        pipeline_throughput.rs  # Frame processing at 40 Hz
```

### 1.2 Public API Sketch

```rust
// lib.rs — primary entry points

pub struct BfldPipeline {
    config: BfldConfig,
    extractor: BfiExtractor,
    feature_engine: FeatureEngine,
    identity_risk: IdentityRiskEngine,
    privacy_gate: PrivacyGate,
    emitter: BfldEmitter,
}

impl BfldPipeline {
    pub fn new(config: BfldConfig) -> Result<Self, BfldError>;
    pub fn process_frame(&mut self, raw: RawBfiCapture) -> Option<BfldEvent>;
    pub fn current_privacy_class(&self) -> PrivacyClass;
    pub fn enable_privacy_mode(&mut self);  // forces class 3
}

pub struct BfldEvent {
    pub timestamp_ns: u64,
    pub presence: bool,
    pub motion: f32,                    // 0.0..1.0
    pub person_count: u8,
    pub identity_risk_score: Option<f32>, // None if privacy_class >= 2
    pub rf_signature_hash: Option<[u8; 32]>, // None if privacy_class >= 2
    pub zone_id: Option<ZoneId>,
    pub confidence: f32,
    pub privacy_class: PrivacyClass,
}

#[repr(u8)]
pub enum PrivacyClass {
    Raw       = 0,
    Derived   = 1,
    Anonymous = 2,
    Restricted = 3,
}
```

---

## 2. Reuse Map: Existing Crates and Modules

### 2.1 RuvSense Modules (wifi-densepose-signal)

Path: `v2/crates/wifi-densepose-signal/src/ruvsense/`

| Module | Used by BFLD | Purpose |
|--------|-------------|---------|
| `coherence_gate.rs` | `identity_risk.rs` | Accept/reject frame based on coherence score; gates embeddings fed into risk calculation |
| `multistatic.rs` | `features.rs` | Attention-weighted fusion for cross_perspective_consistency component of risk score |
| `cross_room.rs` | `privacy_gate.rs` | Environment fingerprinting — confirms that the site_salt corresponds to the current room geometry |
| `longitudinal.rs` | `identity_risk.rs` | Welford stats for temporal_stability component |
| `adversarial.rs` | `extractor.rs` | Physically-impossible signal detection — flags frames that may be from a compromised AP (A5 threat) |

Not used by BFLD: `pose_tracker.rs`, `intention.rs`, `gesture.rs`, `tomography.rs`,
`field_model.rs` — these operate above the identity-risk layer.

### 2.2 RuVector v2.0.4 Crates

| Crate | BFLD Usage | Rationale |
|-------|-----------|-----------|
| `ruvector-attention` | `identity_risk.rs` | Spatial attention over subcarrier dimension for embedding computation |
| `ruvector-mincut` | `features.rs` | Person separation score as input to person_count feature |
| `ruvector-temporal-tensor` | `extractor.rs` | Temporal windowing + compression of BFI angle sequences |

Not used: `ruvector-attn-mincut`, `ruvector-solver` — spectrogram and sparse
interpolation are not needed in the BFI pipeline.

### 2.3 Cross-Viewpoint Fusion (wifi-densepose-ruvector)

Path: `v2/crates/wifi-densepose-ruvector/src/viewpoint/`

| Module | BFLD Usage |
|--------|-----------|
| `coherence.rs` | Cross-viewpoint phase coherence for cross_perspective_consistency risk component |
| `geometry.rs` | Fisher Information / Cramer-Rao bounds for confidence estimation |
| `attention.rs` | GeometricBias-weighted attention for multi-AP BFI fusion |
| `fusion.rs` | MultistaticArray aggregate root — BFLD subscribes to domain events here |

---

## 3. ESP32 Firmware Additions

### 3.1 ESP32-S3 BFI Capability Assessment

The ESP32-S3's WiFi driver (`csi_collector.c` in `firmware/esp32-csi-node/main/`)
uses `esp_wifi_csi_set_config()` and the `wifi_csi_cb_t` callback. This produces
Espressif HT20 CSI in a vendor-specific format — amplitude + phase per subcarrier,
not the VHT/HE Compressed Beamforming frames (CBFR) that contain Phi/Psi angles.

The ESP32-S3 does NOT have a public API to generate or capture CBFR frames. Espressif's
802.11 implementation does receive and process CBFR frames internally (for beamforming
its own transmissions), but these are not exposed via the CSI callback.

**Consequence**: BFI capture for BFLD requires host-side sniffing, not ESP32 firmware
modification.

### 3.2 Host-Side BFI Capture Path

Recommended capture hardware: Raspberry Pi 5 with BCM43456 chip running Nexmon CSI
patch. This is already present in the fleet as `cognitum-v0` (Pi 5, Tailscale IP
100.77.59.83 per CLAUDE.local.md).

Capture path:
1. Nexmon monitor mode captures all 802.11 frames on the target channel.
2. A filter pass extracts CBFR frames (frame type = Action, subtype = VHT/HE CBFR).
3. The rvcsi adapter (`vendor/rvcsi/`) already handles Nexmon PCap format; add a
   BFI extractor alongside the existing CSI extractor.
4. Frames are forwarded to the BFLD pipeline via the existing UDP stream path
   (`stream_sender.c` / sensing-server).

### 3.3 Firmware Changes Required (Minimal)

The only firmware change needed in `firmware/esp32-csi-node/main/` is to the
`stream_sender.c` protocol: add a packet type byte to the stream header to distinguish
CSI frames from BFI frames. The BFI frames originate on the Pi-side host, not the
ESP32; the ESP32 stream is unchanged.

```c
// stream_sender.h — add packet type
#define STREAM_PKT_TYPE_CSI  0x01
#define STREAM_PKT_TYPE_BFI  0x02  // new: BFI frames from host capture
```

---

## 4. Test Plan: 7 Acceptance Criteria Mapped to Rust Tests

| AC | Criterion | Test in `acceptance.rs` |
|----|-----------|------------------------|
| AC1 | Commodity WiFi 5/6 capture (80/160 MHz, 2×2 MIMO minimum) | `ac1_commodity_wifi_capture`: assert BfiExtractor parses 80 MHz VHT CBFR sample fixture |
| AC2 | Presence detection latency ≤ 1s from first non-empty BFI frame | `ac2_presence_latency`: replay 10-frame window, assert first `BfldEvent` with `presence=true` within 1,000 ms wall time |
| AC3 | Motion score published at ≥ 1 Hz on `motion/state` topic | `ac3_motion_hz`: mock MQTT sink, run at 5 Hz input, assert ≥ 1 motion event per second |
| AC4 | Raw BFI bytes never appear in serialized output | `ac4_raw_bfi_absent`: fuzz 1,000 random BfiCaptures, assert no bfi_matrix bytes in serialized BfldFrame for any privacy_class |
| AC5 | Privacy-mode suppresses all identity-derived fields | `ac5_privacy_mode`: enable privacy_mode, assert BfldEvent fields identity_risk_score and rf_signature_hash are None |
| AC6 | Deterministic frame hash for identical inputs | `ac6_deterministic_hash`: run same BfiCapture 100 times, assert all output hashes identical |
| AC7 | CSI-optional fusion: pipeline runs without csi_matrix | `ac7_csi_optional`: run BfldPipeline with None csi_matrix, assert no panic and presence event produced |

Additionally, `tests/hash_rotation.rs` must include:
- `cross_site_isolation`: two BfldPipelines with different site_salts, identical inputs → hashes must differ
- `daily_rotation`: same salt, frames 1 second before/after midnight → hashes must differ

---

## 5. Phased Rollout

### P1 — Frame Format + Extractor Stub (2 weeks)

Deliverables:
- `frame.rs`: `BfldFrame` struct, serialization, CRC32, magic, version
- `extractor.rs`: CBFR parser for 802.11ac VHT + 802.11ax HE formats
- AC1, AC6 tests passing
- `Cargo.toml` with workspace integration

Effort: 1 engineer, 2 weeks.

### P2 — Feature Extraction + Identity Risk (3 weeks)

Deliverables:
- `features.rs`: all 9 named features (mean_angle_delta through identity_separability_score)
- `identity_risk.rs`: risk formula, EmbeddingRingBuf, coherence gate integration
- AC4, AC7 tests passing (raw-absent, CSI-optional)
- Integration with `ruvector-attention` and `ruvector-temporal-tensor`

Effort: 1 engineer, 3 weeks.

### P3 — Privacy Gate + MQTT (2 weeks)

Deliverables:
- `privacy_gate.rs`: privacy_class assignment, field masking, `#[must_classify]` lint
- `mqtt.rs`: per-class topic routing, discovery payloads, ACL documentation
- AC2, AC3, AC5 tests passing (latency, Hz, privacy-mode)
- Hash rotation: `hash_rotation.rs` tests passing
- Deterministic proof bundle: `verify_bfld.py` equivalent

Effort: 1 engineer, 2 weeks.

### P4 — Home Assistant Integration (1 week)

Deliverables:
- MQTT discovery payloads for all 6 entities
- 3 HA blueprints
- `sensor.bfld_identity_risk` marked diagnostic + hidden by default
- Update `wifi-densepose-sensing-server` to include BFLD event routing

Effort: 0.5 engineer, 1 week.

### P5 — Matter Exposure (1 week)

Deliverables:
- `cog-ha-matter` crate updated to filter BfldFrame → Matter attribute reports
- OccupancySensing cluster populated from `presence`
- Rejection list for identity fields enforced at Matter boundary

Effort: 0.5 engineer, 1 week.

### P6 — cognitum Federation (1 week)

Deliverables:
- Topic routing in `mqtt.rs` for federated vs local topics
- Documentation for cognitum-rvf-agent BFLD event subscription
- End-to-end test: Pi 5 (cognitum-v0) receives federated events, identity fields absent

Effort: 0.5 engineer, 1 week.

**Total estimate**: ~10.5 engineer-weeks across 6 phases, approximately 3 calendar months
with one engineer.
