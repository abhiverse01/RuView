# ADR-121: BFLD Identity Risk Scoring and Coherence Gate

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-24 |
| **Deciders** | ruv |
| **Parent** | [ADR-118](ADR-118-bfld-beamforming-feedback-layer-for-detection.md) |
| **Relates to** | [ADR-024](ADR-024-contrastive-csi-embedding-model.md) (AETHER), [ADR-027](ADR-027-cross-environment-domain-generalization.md) (MERIDIAN), [ADR-029](ADR-029-ruvsense-multistatic-sensing-mode.md) (multistatic fusion), [ADR-086](ADR-086-edge-novelty-gate.md) (novelty gate precedent), [ADR-120](ADR-120-bfld-privacy-class-and-hash-rotation.md) (privacy class) |
| **Companion research** | [`docs/research/soul/`](../research/soul/) — risk score doubles as Soul Signature enrollment-quality signal; §2.7 defines the Recalibrate exemption. |
| **Tracking issue** | TBD |

---

## 1. Context

BFLD's distinguishing primitive is the `identity_risk_score` — a scalar that says **"is this capture window currently capable of identifying a specific person?"**. The score has two consumers:

1. **The operator** — exposed as an HA diagnostic sensor (ADR-122). A spike from the long-term baseline indicates the RF environment has shifted toward a higher-leakage regime (new AP firmware, denser MIMO, attacker-grade sniffer in range).
2. **The privacy gate** (ADR-120) — when the score crosses a configurable threshold, the gate downgrades the active `privacy_class` automatically (e.g., 2 → 3) until the score recovers.

The score must be:
- **Bounded** in `[0, 1]` for HA gauge entities.
- **Calibrated** against actual re-ID success rate, ideally on the KIT BFId dataset.
- **Computable on-device** at ≥ 1 Hz on a Pi 5 core or an aarch64 cognitum-v0.
- **Stable** — small environmental changes should not produce wild swings; the score is for slow-moving regime detection, not per-frame chatter.

ADR-086 (edge novelty gate) establishes a precedent for an on-device gate primitive. BFLD's risk scoring borrows the gate-pattern but with identity leakage as the trigger condition.

---

## 2. Decision

### 2.1 Nine features (from BFLD spec §5)

The features are computed over a sliding window of `W = 32` BFI frames (≈3 s at 10 Hz):

| Feature | Definition | Source |
|---------|------------|--------|
| `mean_angle_delta` | mean( ‖ Φ_t − Φ_{t-1} ‖ over subcarriers ) | extractor |
| `subcarrier_variance` | var( ‖ Φ ‖ over subcarrier axis ) | extractor |
| `temporal_entropy` | Shannon entropy of angle-bin histogram over W | extractor |
| `doppler_proxy` | FFT peak magnitude of mean-angle time series | features.rs |
| `path_stability` | 1 − ‖ Φ_t − median(Φ_{t-W..t}) ‖ / scale | features.rs |
| `cross_antenna_correlation` | mean Pearson correlation across n_tx × n_rx pairs | features.rs |
| `burst_motion_score` | high-pass-filtered angular velocity, soft-thresholded | features.rs |
| `stationarity_score` | 1 − rolling KL divergence over W/2 vs W | features.rs |
| `identity_separability_score` | top-1 cosine to nearest AETHER cluster centroid | identity_risk.rs |

The first eight are sensing features (also used by the presence/motion pipeline). Only the ninth depends on the AETHER embedding and therefore on `identity_class >= 1`.

### 2.2 Identity risk formula

```rust
pub fn identity_risk_score(
    sep: f32,    // identity_separability_score, [0, 1]
    stab: f32,   // temporal_stability, [0, 1] = ema(path_stability, alpha=0.1)
    consist: f32,// cross_perspective_consistency, [0, 1] = multistatic.rs
    conf: f32,   // sample_confidence, [0, 1] = f(SNR, n_subcarriers, n_rx)
) -> f32 {
    // Clamp inputs, then multiplicative combination — any factor near 0 dominates.
    let s = sep.clamp(0.0, 1.0);
    let t = stab.clamp(0.0, 1.0);
    let p = consist.clamp(0.0, 1.0);
    let c = conf.clamp(0.0, 1.0);
    (s * t * p * c).clamp(0.0, 1.0)
}
```

Multiplicative combination is chosen so that **any** weak factor (e.g., very low SNR ⇒ low `conf`) collapses the score toward 0. This matches the privacy intent: when the system is uncertain, the score should be low and the operator should not be alarmed.

### 2.3 Calibration target

The score is calibrated against re-ID success rate on a held-out test split of the KIT BFId dataset. A piecewise-linear isotonic regression maps raw scores into a calibrated `[0, 1]` band where `score ≥ 0.8` corresponds to `>80%` re-ID accuracy on a 5-second window in the calibration dataset.

Calibration parameters live in `v2/crates/wifi-densepose-bfld/data/risk_calibration.toml` and are versioned independently of the code. A regression update is a content-only PR.

### 2.4 Coherence gate

The coherence gate (per ADR-029 `coherence_gate.rs` pattern) consumes the risk score and emits one of four actions:

```rust
pub enum GateAction {
    Accept,           // score < 0.5, publish normally
    PredictOnly,      // 0.5 <= score < 0.7, publish but flag confidence
    Reject,           // 0.7 <= score < 0.9, drop the event
    Recalibrate,      // score >= 0.9, drop AND rotate site_salt
}
```

The `Recalibrate` action triggers a forced site-salt rotation — an aggressive response to a sustained high-risk regime. It costs the operator continuity of long-term aggregate analytics but is the right answer to an attacker-grade sniffer arriving in range.

### 2.5 Hysteresis

To prevent oscillation around the gate thresholds, the gate uses ±0.05 hysteresis and a 5-second debounce. A score must cross the boundary by the hysteresis margin and persist for the debounce window before the gate action changes.

### 2.6 Soul Signature interaction — Recalibrate exemption and enrollment-quality gate

Soul Signature (`docs/research/soul/`) intentionally exists in a high-separability regime — the whole point of its 60-second enrollment protocol is to push `identity_separability_score` toward 1.0. The default coherence gate (§2.4) would therefore fire `Recalibrate` constantly inside Soul Signature zones, rotating `site_salt` every few seconds and breaking enrollment.

Two integrations resolve this:

1. **Recalibrate exemption.** When the gate is about to fire `Recalibrate`, it consults a `SoulMatchOracle` (provided by the Soul Signature crate when compiled with `--features soul-signature`). If the oracle reports that the current high-separability cluster matches an enrolled `person_id` above the Soul Signature acceptance threshold, the gate downgrades to `PredictOnly` instead. The high score is the *intended* outcome of a successful match, not an attack indicator. Without the `soul-signature` feature, the oracle is a no-op stub returning `MatchOutcome::NotEnrolled`, so the gate behaves exactly per §2.4.

2. **Enrollment-quality gate.** Soul Signature's enrollment protocol (`scanning-process.md` §3) requires that the sensing zone meet a minimum identity-leakage regime — too low, and the resulting signature is unreliable. The BFLD `identity_risk_score` is exactly the right signal. Soul Signature gates enrollment on `score >= ENROLL_MIN` (default `0.65`) sustained over the 60-second window. If the score drops below threshold mid-enrollment, the protocol aborts and the operator is prompted to re-attempt in better RF conditions.

The exemption is asymmetric: it suppresses `Recalibrate` only for known-enrolled matches. Unknown high-separability clusters (a real attacker-grade sniffer, or an unenrolled person whose identity is unexpectedly leaky) still trigger `Recalibrate` as designed.

### 2.7 Compute budget

| Stage | Target latency | Implementation |
|-------|----------------|----------------|
| Feature extraction (8 features) | < 3 ms per window | ndarray + nalgebra; vectorized over subcarriers |
| Separability (cosine to centroids) | < 5 ms per window | RuVector RaBitQ index (ADR-085) over ≤ 1k centroids |
| Risk score | < 0.1 ms | scalar multiplicative |
| Gate decision + hysteresis | < 0.1 ms | scalar |

Total p95 ≤ 10 ms per window on a Pi 5 core (8 ms target). Headroom on cognitum-v0 (Pi 5 + Hailo) is ample; ESP32-S3 hosts only the extraction stage (features computed; risk score is host-side per ADR-123). The `SoulMatchOracle` lookup (§2.6) adds < 1 ms when the `soul-signature` feature is enabled (RaBitQ index over enrolled centroids).

---

## 3. Consequences

### Positive

- The risk score becomes a first-class diagnostic surface for operators and a structural input to the privacy gate — both consumers from a single computation.
- Multiplicative combination is conservative under uncertainty; the system is biased toward "report low risk when unsure", which is the right default.
- Calibration is a content-only update — no recompile needed when the calibration file changes.
- The recalibration gate action gives the system a self-healing response to a sniffer arrival without operator intervention.

### Negative

- Calibration requires the KIT BFId dataset; without it the score is uncalibrated and serves only as an internal trigger, not a publishable signal.
- Multiplicative scoring can be dominated by `sample_confidence`, which is sensitive to channel conditions. A persistent low-SNR environment will keep the published score near 0 even when the underlying separability is high — an under-reporting failure mode that the documentation must call out.
- The recalibrate action breaks historical hash continuity by design; an operator who wants long-term aggregates needs to know they will see a discontinuity on recalibrate events.

### Neutral

- The nine features overlap with the existing CSI pipeline. BFLD computes them on BFI; the CSI pipeline computes them on CSI. Both can be fused via `cross_perspective_consistency`.

---

## 4. Alternatives Considered

### Alt 1: Additive scoring (`(s + t + p + c) / 4`)

Rejected: a sample with high separability but very low confidence would still produce a moderate score, which over-reports risk in degraded RF conditions.

### Alt 2: Maximum scoring (`max(s, t, p, c)`)

Rejected: over-reports risk because any single high factor pins the output, even if the others contradict it.

### Alt 3: Learned scoring (a small MLP)

Rejected for this ADR: introduces an opaque model whose output cannot be audited from first principles. The multiplicative formula is simple, conservative, and directly explainable to operators. A learned model is a future option once enough calibration data is in hand.

### Alt 4: Per-feature thresholds instead of a continuous score

Rejected: continuous score is needed for the HA gauge entity and for downstream calibration. Per-feature thresholds would force operators to interpret nine separate binaries.

---

## 5. Acceptance Criteria

- [ ] **AC1**: All nine features are computed in `< 8 ms` p95 per window on a Pi 5 core.
- [ ] **AC2**: `identity_risk_score` is monotonic non-decreasing in any single input when the other three are held constant.
- [ ] **AC3**: Calibration regression on the KIT BFId test split: `score ≥ 0.8` corresponds to ≥ 80% re-ID accuracy ± 5%.
- [ ] **AC4**: The coherence gate emits `Recalibrate` if score is ≥ 0.9 for ≥ 5 seconds.
- [ ] **AC5**: Hysteresis prevents action oscillation across ± 0.05 of a threshold within a 5-second window.
- [ ] **AC6**: At `privacy_class = 3`, the risk score is computed but not published to MQTT (kept local for the gate only).
- [ ] **AC7**: A reproducible 1,000-frame synthetic fixture produces a deterministic score sequence (bit-identical across runs).

---

## 6. References

- ADR-118 (umbrella)
- ADR-024 (AETHER encoder for separability)
- ADR-029 (`coherence_gate.rs` precedent)
- ADR-086 (edge novelty gate pattern)
- ADR-120 §2.4 (class transition consumed by gate)
- KIT BFId dataset: https://publikationen.bibliothek.kit.edu/1000185756
