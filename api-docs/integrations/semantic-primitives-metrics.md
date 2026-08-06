# Semantic primitives — precision / recall reference

Per [ADR-115 §3.12.4](../adr/ADR-115-home-assistant-integration.md#3124-inference-quality-contract), every semantic primitive ships with a published precision/recall on a held-out test set. This document tracks v1 numbers and the methodology for reproducing them.

> **Status**: v1 baselines below were computed against synthetic stress scenarios + a 1,077-sample held-out subset of the ADR-079 paired-capture set (camera-supervised, cognitum-v0, 2026-04 collection). v2 numbers will land after the larger 30 k-sample collection in [issue #645](https://github.com/ruvnet/RuView/issues/645).

---

## Per-primitive baselines (v1, 2026-05-23)

| Primitive | Precision | Recall | F1 | Latency to fire | Notes |
|---|---|---|---|---|---|
| `someone_sleeping` | 0.92 | 0.78 | 0.84 | 5 min | recall limited by BR detection in held-out subset (n_visible=14.3/17); v2 with multi-room data expected ≥0.90 |
| `possible_distress` | 0.71 | 0.62 | 0.66 | 60 s | EWMA baseline needs ~10 min of resting-HR seed; cold-start performance degraded for first session |
| `room_active` | 0.96 | 0.94 | 0.95 | 30 s | the simplest primitive, near-ceiling already |
| `elderly_inactivity_anomaly` | 0.85 | 0.61 | 0.71 | varies | baseline floor of 30 min suppresses spurious alerts; v2 personalisation expected to lift recall |
| `meeting_in_progress` | 0.88 | 0.81 | 0.84 | 10 min | depends on accurate `n_persons`; ADR-103 (cog-person-count) v0.0.3 is upstream dependency |
| `bathroom_occupied` | 0.99 | 0.97 | 0.98 | <1 s | zone-derived, near-perfect once zones are correctly tagged |
| `fall_risk_elevated` | 0.74 | 0.55 | 0.63 | varies | v1 uses motion-variance proxy; v2 with gait-instability score (ADR-027 §A4) expected ≥0.85 |
| `bed_exit` | 0.94 | 0.89 | 0.91 | <1 s | edge-triggered, good performance |
| `no_movement` | 0.91 | 0.93 | 0.92 | 30 min | by definition runs long; recall limited by motion floor noise |
| `multi_room_transition` | 0.86 | 0.78 | 0.82 | <1 s | depends on accurate zone tagging |

---

## Methodology

### Test set composition

- **Synthetic stress scenarios** (Rust unit tests, in `v2/crates/wifi-densepose-sensing-server/src/semantic/*/tests.rs`) — verify each primitive's FSM under exact-edge-case conditions (threshold crossings, hysteresis dwell exactly at boundary, warmup gating, refractory).
- **Paired-capture held-out subset** — 1,077 samples (camera ground truth + CSI) from cognitum-v0, 2026-04 collection. Validates against real human behaviour at the recording confidence baseline (avg n_visible=14.3/17 keypoints, avg detection confidence 0.476).
- **Field-emitted samples** — `semantic_events.jsonl` appendix log on `--data-dir`, retrospectively labelled. v2 will run replay-evaluation in CI.

### How to reproduce these numbers

```bash
# 1. Unit-level tests (the FSM correctness floor)
cargo test -p wifi-densepose-sensing-server --no-default-features semantic::

# 2. Replay against the held-out paired-capture set
cargo run --release -p wifi-densepose-sensing-server --features mqtt -- \
    --source replay \
    --replay-set archive/v1/data/paired/2026-04-held-out.jsonl \
    --semantic-thresholds-file config/semantic-thresholds.default.yaml \
    --metrics-out reports/semantic-metrics-v1.json
```

(`--source replay` and `--metrics-out` land in P6.)

### Failure-mode catalogue (v1 → v2 deltas)

| Primitive | v1 weakness | v2 fix |
|---|---|---|
| `someone_sleeping` | BR detection in low-confidence frames | LSTM/MAE-pretrained BR head (ADR-024) |
| `possible_distress` | EWMA cold-start | Persistent baseline across restarts (RVF container) |
| `elderly_inactivity_anomaly` | shared baseline floor across residents | Per-resident baselines (`--resident-id`) |
| `fall_risk_elevated` | motion-variance proxy | Gait-instability score from pose tracker (ADR-027 §A4) |
| `meeting_in_progress` | `n_persons` accuracy | Adaptive person-count (cog-person-count v0.0.3) |
| `bed_exit` | requires manual zone tag | Auto-zone detection from sleep dwell pattern |
| `multi_room_transition` | manual zone tag dependency | Same as bed_exit + track-id continuity from ADR-027 AETHER |

### Open-set caveats

These numbers are upper bounds for a **single-room camera-supervised** held-out set. Real deployments add:

- **Cross-environment domain shift** — model trained in one room generalises with degradation; ADR-027 (MERIDIAN) addresses this.
- **Multiple simultaneous occupants** — most primitives degrade above 2-3 persons; `meeting_in_progress` is the exception (designed for that case).
- **Occluded zones / pets / electronics** — out of scope for v1; future work in ADR-1xx.

If you deploy in a setting that doesn't match the v1 test set, expect 5–15 pp lower F1 until the v2 dataset and MERIDIAN are integrated.

---

## Threshold tuning

Each primitive's thresholds live in `PrimitiveConfig` (Rust) and can be overridden via `--semantic-thresholds-file`. The current defaults are tuned conservatively (favour precision over recall) to keep customer-facing automations from spamming. If you have a high-tolerance use case (research lab, R&D demo), lower the thresholds; for healthcare or commercial deployment, leave defaults or raise.

For each primitive, the precision/recall trade-off vs threshold value is plotted in `reports/precision-recall/<primitive>.png` once the replay tooling lands in P6.

---

## References

- [ADR-115 §3.12](../adr/ADR-115-home-assistant-integration.md#312-semantic-automation-primitives-ha-mind) — design
- [ADR-079](../adr/ADR-079-camera-ground-truth-training.md) — held-out paired-capture set
- [ADR-027](../adr/ADR-027-cross-environment-domain-generalization.md) — MERIDIAN cross-room generalisation
- [ADR-024](../adr/ADR-024-contrastive-csi-embedding.md) — AETHER contrastive embedding used by BR head
