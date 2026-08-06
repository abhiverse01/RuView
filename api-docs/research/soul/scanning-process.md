# Soul Signature — Scanning Process

**Status:** Research Specification (Pre-Implementation)
**Date:** 2026-05-24
**Author:** ruv

---

## 1. Hardware Prerequisites

### 1.1 Full Protocol (N ≥ 3 Nodes)

| Component | Minimum | Recommended | Notes |
|---|---|---|---|
| Sensing nodes | 3 × ESP32-S3 (ADR-028) | 5+ nodes | Multi-node triangulation reduces angle-dependent blind spots; ADR-029 multistatic mesh |
| Compute appliance | Cognitum Seed (Pi 5 + Hailo) | Same | Runs the field model, AETHER inference, vitals pipeline |
| Network link | 2.4 GHz or 5 GHz AP | Dedicated sensing AP | Shared AP with user traffic degrades CSI frame rate |
| Firmware version | ADR-110 v0.7.0+ | Same | Ed25519 witness chain required for attestation |
| Clock sync | 802.15.4 time-sync (ESP32-C6) or NTP fallback | 802.15.4 preferred | ±100 µs alignment per ADR-110; NTP gives ±5 ms |

### 1.2 Degraded Mode (1 Node)

A single-node enrollment produces an incomplete signature:
- Skeletal proportions: degraded (single-angle view)
- Subcarrier reflection profile: single orientation only (3-orientation protocol collapses to 1)
- AETHER embedding: usable but lower confidence
- Cardiac / respiratory: unaffected (single-node sufficient)
- Gait timing: usable if node placement allows bidirectional walk

Single-node signatures MUST be tagged `degraded_mode: true` in the manifest. The
match score uses only the channels that met minimum confidence thresholds. The
soul signature is technically valid but should be re-enrolled with multi-node
hardware when possible.

### 1.3 ESP32-C6 Uplift (Wi-Fi 6 HE-LTF)

When at least one ESP32-C6 node is present (ADR-110), the subcarrier count
expands from 52 (HT-LTF, S3) to up to 242 (HE-LTF, C6). The MERIDIAN
HardwareNormalizer (ADR-027) maps all nodes to a canonical 56-subcarrier
representation for the AETHER backbone. The full 242-subcarrier profile is
preserved in the SubcarrierReflectionProfile node for higher-fidelity matching
when available. The C6's 802.15.4 time-sync (±100 µs) also improves multistatic
coherence relative to NTP-only S3 meshes.

---

## 2. Structured 60-Second Enrollment Protocol

The enrollment protocol produces exactly one `.rvf` soul signature file. The
protocol is structured into five phases with exact timing. A human-readable
prompt sequence should be delivered to the subject via audio or display.

### Phase 0 — Empty-Room Field Recalibration (T+0 to T+10)

Before the subject enters the sensing zone, the room must be empty and the
ADR-030 field model must be current.

```
T+0s   : System checks field model age. Maximum age: 4 hours.
          If stale or absent → run field recalibration:
            Collect 1,200 CSI frames at 20 Hz (60 seconds of empty room)
            Compute per-link Welford mean and covariance
            Run SVD on covariance matrix → top-K=8 eigenmode vectors
            Store in field_model.rs::FieldNormalMode

T+0–10s: Quiet sampling of empty-room field state. No subject present.
          Operator prompt: "Please ensure the room is empty."
          System: verifies presence score < 0.1 (ADR-039 Tier 2 presence detection).
          Failure: if presence score ≥ 0.1, abort and report FAIL_ROOM_NOT_EMPTY.
```

This phase is skipped (not aborted) if the field model was updated within the
last 4 hours AND the current empty-room sampling confirms presence score < 0.05.

### Phase 1 — Deep Breathing Baseline (T+10 to T+25)

Subject enters the sensing zone and performs five deep breathing cycles.

```
T+10s  : Subject enters scan zone. System detects presence.
          Operator prompt: "Please stand still and breathe slowly and deeply."

T+10–25s: Subject stands at zone center, facing node cluster.
           Five complete breath cycles, each ≥ 4 seconds.
           System collects:
             - ADR-021 BreathingExtractor: baseline_bpm, depth_amplitude,
               inspiration_expiration_ratio, HRV_RSA
             - ADR-021 HeartRateExtractor: initial HR, HRV_SDNN (partial)
             - AETHER embedding: accumulates over 300 CSI frames (20 Hz × 15s)
           Quality gate: BreathingExtractor VitalCoherenceGate must emit
             PERMIT for ≥ 10 of the 15 seconds. Failure → FAIL_POOR_BREATHING_SIGNAL.
```

### Phase 2 — Seated Rest (T+25 to T+35)

Subject sits to minimize motion and allow cardiac signal isolation.

```
T+25s  : Operator prompt: "Please sit down and rest quietly."

T+25–35s: Subject seated, minimal movement.
           System collects:
             - HeartRateExtractor: HR baseline, HRV_SDNN, HRV_RMSSD,
               LF/HF ratio, sinus rhythm classification
             - Cardiac_Waveform_Morphology: 64-coefficient wavelet decomposition
               of bandpass-filtered cardiac phase signal (0.8–2.0 Hz)
           Quality gate: HR confidence ≥ 0.6 for ≥ 7 of 10 seconds.
             Failure → FAIL_POOR_CARDIAC_SIGNAL (soft failure: cardiac nodes
             marked low-confidence; signature proceeds without them if AETHER
             and gait nodes pass their own thresholds).
```

### Phase 3 — Gait Walk (T+35 to T+50)

Subject walks a 2-meter line twice in each direction.

```
T+35s  : Operator prompt: "Please walk a straight line of 2 meters back and
          forth twice at your natural pace."

T+35–50s: Subject walks: A→B, B→A, A→B, B→A (four transits, ≥ 8 strides total).
           System collects (via pose_tracker.rs, ADR-029 Sect 2.7):
             - GaitTimingNode: cadence, stride_period_variance,
               double_support_pct, asymmetry_index, step_width_m
             - SkeletalProportionsNode: torso/limb ratios from 17-keypoint
               trajectory accumulated over ≥ 8 strides
             - AETHER embedding: continues accumulating (300 more frames)
           Quality gate: ≥ 8 strides detected with confidence ≥ 0.7 per stride.
             Failure → FAIL_INSUFFICIENT_GAIT_DATA.
           Note: the ruvector-mincut DynamicPersonMatcher must confirm only one
           person is tracked. If two tracks are active → FAIL_MULTIPLE_SUBJECTS.
```

### Phase 4 — Standing Orientation Scan (T+50 to T+60)

Subject stands at three orientations to capture the subcarrier reflection profile.

```
T+50s  : Operator prompt: "Please stand facing the wall. I will ask you to
          rotate in place twice."

T+50–53s: Orientation 0° (subject faces primary node cluster).
           System collects: SubcarrierReflectionProfile at 0°
           (ADR-030 field-subtracted, 56 subcarriers, amplitude + phase).

T+53s  : Operator prompt: "Please turn 90 degrees to your right."

T+53–56s: Orientation 90°.
           System collects: SubcarrierReflectionProfile at 90°.

T+56s  : Operator prompt: "Please turn 90 degrees to your right again."

T+56–60s: Orientation 180°.
           System collects: SubcarrierReflectionProfile at 180°.
           Body_Field_Coupling: computed from AETHER attention map weighted
           by ADR-030 top-K=8 eigenvectors (final computation at T=60s).

T+60s  : Enrollment window closes.
          AETHER embedding finalized: mean pool over all ~1,200 accumulated frames.
          All node confidence values computed.
```

---

## 3. Quality Gates

The enrollment FAILS and emits a structured error code if any of the following
conditions are met. Failed enrollments do not produce a stored `.rvf` file.

| Gate | Condition for FAIL | Error code |
|---|---|---|
| Room occupied | Presence score ≥ 0.1 at Phase 0 end | `FAIL_ROOM_NOT_EMPTY` |
| Multiple subjects | ≥ 2 active pose tracks during Phases 1–4 | `FAIL_MULTIPLE_SUBJECTS` |
| Intermittent presence | Subject exits sensing zone for > 3 consecutive seconds | `FAIL_SUBJECT_LEFT_ZONE` |
| AETHER confidence low | Final embedding confidence < 0.6 (HNSW search confidence) | `FAIL_AETHER_LOW_CONFIDENCE` |
| Breathing signal absent | VitalCoherenceGate PERMIT rate < 67% during Phase 1 | `FAIL_POOR_BREATHING_SIGNAL` |
| Gait data insufficient | Fewer than 8 strides detected with confidence ≥ 0.7 | `FAIL_INSUFFICIENT_GAIT_DATA` |
| Field model dirty | Field model age > 4 hours and recalibration refused | `FAIL_STALE_FIELD_MODEL` |
| Adversarial detection | RuvSense adversarial.rs flags physically impossible signal | `FAIL_ADVERSARIAL_SIGNAL` |
| Node count below minimum | Fewer than 2 nodes online during Phases 3–4 | `WARN_DEGRADED_MODE` (not a hard fail; produces degraded signature) |

Soft failures (cardiac signal only) do not abort the enrollment; they mark those
nodes as low-confidence and reduce the match weight for those channels at
recognition time.

---

## 4. Fast Scan (10-Second Degraded Identification)

A fast scan produces a partial query embedding, not a stored profile. It is used
for recognition of already-enrolled subjects, not for new enrollment.

```
T+0s   : System checks whether field model is current (age < 4 hours).
          If stale: recognition accuracy degraded; warn operator.

T+0–10s: Subject stands still at zone center, natural breathing.
          System collects: AETHER embedding (200 frames, 10s at 20 Hz).
          Cardiac HR: partial (confidence typically < 0.5).
          Gait: not available.
          Subcarrier reflection: 1 orientation only.

T+10s  : Query issued against all stored profiles in HNSW index.
          Match score computed using available channels only.
          Cardiac, gait, and skeletal proportions excluded from denominator
          (availability factor = 0 for absent channels).
```

Fast scan is acceptable for:
- Returning resident recognition (already enrolled, low-friction use case)
- Home automation triggers (occupancy attribution per ADR-115 HA-MIND)

Fast scan is NOT acceptable for:
- Initial enrollment
- High-assurance access control
- Healthcare identification

---

## 5. Continuous Mode — Implicit Signature Refinement

In continuous operating mode, the system incrementally updates the online
aggregator for enrolled persons as they go about their normal activities. The
stored profile is re-published from the aggregator every 90 days (or on the
re-scan cadence, whichever comes first). This means a deployed system becomes
more accurate over time, not less.

Convergence property: the Welford online statistics in the aggregator are
numerically stable and converge to the true population mean/variance as
observation count increases. The AETHER embedding accumulated over thousands
of natural-activity windows is more representative than a single 60-second
enrollment. The stored profile is replaced (not amended) on each re-publish; the
old profile is archived (not deleted) per the forward-secrecy requirements in
`security.md`.

The continuous mode raises a consent concern: a person is effectively being
re-enrolled continuously without explicit action. This is addressed in
`security.md §4` (Consent Architecture).

---

## 6. Multi-Room Enrollment

When a person moves across multiple sensing zones (e.g., living room and bedroom
each with a Cognitum Seed node cluster), the cross-room signature works as follows:

1. Full 60-second enrollment is performed in the primary room. This produces the
   initial stored profile with `environment_normalized: false` in the manifest.

2. When the MERIDIAN domain generalization layer (ADR-027) is active, the
   HardwareNormalizer maps the enrollment embedding to the environment-invariant
   subspace. The stored profile is updated to `environment_normalized: true`.

3. In subsequent rooms, a fast scan (10s) is sufficient to attribute identity. The
   MERIDIAN-normalized AETHER embedding handles the room shift.

4. For healthcare deployments requiring room-by-room re-enrollment for regulatory
   reasons, a per-room enrollment protocol runs in each room and the signatures
   are linked by the opaque `person_id` field (never by raw PII).

---

## 7. Re-Scan Cadence

| Deployment context | Re-scan interval | Rationale |
|---|---|---|
| Healthy adult (residential) | 90 days | Anatomy stable; continuous mode refines continuously |
| Child (growing skeleton) | 30 days | Skeletal proportions change; gait timing changes |
| Healthcare / clinical | Per clinical event | Post-surgery, post-illness, post-significant weight change |
| Post-exercise monitoring | 7 days during active programs | Body composition changes affect RF backscatter |
| Any | On drift alert from longitudinal.rs (ADR-030 Tier 4) | System-initiated; shown to user as "calibration recommended" |

The `longitudinal.rs` module monitors five drift metrics (GaitSymmetry,
StabilityIndex, BreathingRegularity, MicroTremor, ActivityLevel) using Welford
statistics over daily observations. When any metric exceeds 2-sigma deviation
sustained for 3 consecutive days, a `DriftAlert` is emitted. The system
displays this as "signature drift detected — re-scan recommended," not as a
health diagnosis.

---

## 8. Output Artifact

On successful completion, the enrollment pipeline produces:

1. `signature-<sha256>.rvf` — the binary soul signature container. Content-addressed.
   Encrypted with the person's key (see `security.md §5`) before writing to disk.

2. `signature-<sha256>.json` — the JSON-LD sidecar for human inspection and audit.
   Does not contain raw vector data. Safe to log.

3. A row in the local HNSW index (`ruvector-core::VectorIndex`, `person_track`
   subindex per ADR-024 §2.4) linking the person_id to the AETHER embedding.
   This index is used for O(log n) recognition queries.

4. An Ed25519 witness entry per ADR-110, signing
   `(rvf_sha256 || timestamp_ns || enrolled_by_device_id)`. Stored in the
   RVF SEG_WITNESS segment AND in the node's local audit log.

The enrollment process does NOT:
- Transmit raw CSI or raw biometrics to any external server.
- Publish the soul signature to MQTT or Matter unless explicitly configured with
  `--privacy-mode disabled` (see `security.md §6`).
- Store PII (name, email, account linkage) in the `.rvf` file. The `person_id`
  field is an opaque u64. PII linkage, if any, lives in the application layer
  and is governed by separate access control.
