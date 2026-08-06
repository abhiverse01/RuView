# BFLD Benchmarks and Evaluation Strategy

## 1. Datasets

### 1.1 BFId Dataset (Primary)

**Reference**: Todt, Morsbach, Strufe; KIT. ACM CCS 2025.
https://dl.acm.org/doi/10.1145/3719027.3765062
https://ps.tm.kit.edu/english/bfid-dataset/index.php

197 individuals. BFI and CSI recorded simultaneously. Multiple sessions, multiple AP
angles. Available to researchers for non-commercial use on request from KIT.

**Use in BFLD evaluation**: The BFId dataset provides the ground-truth identity labels
needed to calibrate `identity_risk_score`. Specifically: given BFId's known re-ID
accuracy as a function of time window, BFLD's identity_risk_score should correlate
with BFId's success rate. High-risk frames (score > 0.7) should correspond to windows
where BFId achieves > 80% accuracy; low-risk frames (score < 0.2) should correspond
to windows where BFId accuracy approaches chance.

### 1.2 Wi-Pose and MM-Fi (Context)

**MM-Fi**: Multi-modal WiFi sensing dataset used by this project (ADR-015). Contains
synchronized WiFi CSI, mmWave, and camera pose data. Does not contain BFI separately,
but can be used to validate BFLD's CSI-optional path (AC7).

**Wi-Pose**: Academic benchmark for WiFi pose estimation. CSI only; used for
person_count and motion accuracy baselines.

### 1.3 Proposed In-House Multi-Site Capture Protocol

**Purpose**: Validate cross-site isolation (Invariant 3) and daily rotation.

**Setup**:
- Site A: ruvultra (RTX 5080 workstation, Tailscale 100.104.125.72) with USB WiFi
  adapter in monitor mode.
- Site B: cognitum-v0 (Pi 5, Tailscale 100.77.59.83) with Nexmon monitor mode.
- Subject pool: 5–10 volunteers.
- Protocol: Each subject walks a fixed path at each site on 3 consecutive days.
  BFI captured simultaneously at both sites using Wi-BFI.

**Analysis**:
1. Can the BFId classifier re-identify subjects within a site? (Baseline — should
   confirm BFId's published results.)
2. Can any classifier re-identify subjects across sites using BFLD's
   rf_signature_hash? (Should fail — cross-site isolation test.)
3. Can any classifier re-identify across days using BFLD's rf_signature_hash? (Should
   fail — daily rotation test.)

---

## 2. Metrics

### 2.1 Presence Detection

| Metric | Definition | Target |
|--------|-----------|--------|
| Latency p50 | Time from first non-empty BFI frame to first `presence=true` event | < 500 ms |
| Latency p95 | | < 1000 ms (AC2) |
| False positive rate | Presence=true when room is confirmed empty | < 5% |
| False negative rate | Presence=false when person confirmed present | < 2% |

Measurement method: camera ground-truth (ruvultra webcam via MediaPipe Pose, same
as ADR-079 collection protocol) for empty/occupied labels.

### 2.2 Motion Score

| Metric | Definition | Target |
|--------|-----------|--------|
| MAE vs ground truth | Mean absolute error of motion score vs camera-derived motion magnitude | < 0.1 |
| Hz at sustained operation | Events published per second on `motion/state` | >= 1 Hz (AC3) |
| Latency p95 | Time from motion onset (camera) to motion event | < 750 ms |

### 2.3 Person Count

| Metric | Definition | Target |
|--------|-----------|--------|
| Count accuracy | Fraction of windows where BFLD person_count == camera count | > 85% for 1–3 persons |
| Count MAE | |  < 0.5 for counts 1–4 |

Person count is harder than presence. The target is achievable with MinCut separation
(`ruvector-mincut`) but requires multi-AP coverage for 4+ persons.

### 2.4 Identity Risk Calibration

This is BFLD's novel evaluation dimension — no prior system has explicitly quantified
this.

**Calibration definition**: Let `r(t)` = BFLD's identity_risk_score at time t.
Let `acc(t)` = BFId classifier's re-identification accuracy when trained on frames
around time t. The identity_risk_score is *calibrated* if:

    E[acc(t) | r(t) = v] is monotonically increasing in v

In other words: higher risk scores should correspond to frames where identity inference
is genuinely easier.

**Evaluation protocol**:
1. Run BFId classifier in sliding 5-second windows on the BFId dataset.
2. Record per-window BFId accuracy (using leave-one-out cross-validation).
3. Run BFLD's identity_risk_score computation on the same windows.
4. Compute Spearman correlation between risk scores and BFId accuracy.
5. Target: Spearman rho > 0.5 (positive monotonic correlation).

### 2.5 Privacy-Mode False Positive Rate

When `privacy_mode` is enabled (privacy_class = 3), all identity-correlated fields
should be suppressed. The false positive rate is the fraction of outbound events
that inadvertently include an identity-correlated field despite privacy_mode being
active.

**Target**: 0% (this is a hard correctness requirement, not a statistical target).
Verified by the AC5 fuzz test in `acceptance.rs`.

---

## 3. Red-Team Protocol

### 3.1 Hash Re-identification Attack

**Question**: Can an attacker re-identify a person across rotated hashes?

**Setup**:
- Run BFLD pipeline for person X across 3 days.
- Collect `rf_signature_hash` values for each day: H_1, H_2, H_3.
- Adversary has access to H_1, H_2, H_3 and knows they are from the same site.
- Adversary attempts to confirm H_1, H_2, H_3 are from the same person.

**Success condition**: adversary achieves confirmation rate > chance (1/N for N subjects).

**Expected result**: FAIL (by construction of the hash rotation with site_salt).
Since day_epoch changes daily and site_salt is fixed but unknown to the adversary,
the hash function is a keyed PRF. The adversary has three random-looking 32-byte
values with no structural relationship. Success rate should be indistinguishable from
random guessing.

**Quantitative target**: success rate <= 1/N + 0.05 (within 5% of chance).

### 3.2 Cross-Site Re-identification Attack

**Question**: Can an attacker confirm person X visited both site A and site B?

**Setup**: Same as Section 1.3 in-house protocol. Adversary has BFLD event streams
from both sites.

**Method**: Attempt to match rf_signature_hash values from site A and site B on the
same day. Alternatively, train a classifier on BFI features (using the raw angle
sequences from the captured data) and attempt cross-site re-ID.

**Expected result**: Hash-based matching fails by construction. Classifier-based
re-ID may succeed if the adversary has raw angle data (which BFLD does not publish)
but not using BFLD's published output.

**Success condition**: hash-based cross-site match rate <= 1/N + 0.05.

### 3.3 Timing Side-Channel Attack

**Question**: Can an attacker infer a person's schedule by monitoring
identity_risk_score over time?

**Method**: Record identity_risk_score time series. Correlate with known schedule
(person X leaves at 8am, returns at 6pm). Compute mutual information between
schedule and risk score time series.

**Expected result**: Some correlation exists (risk score rises when person enters),
but the attacker learns "someone is present" — equivalent to the presence sensor —
not identity. This is acceptable: presence information is already published at
class 2.

---

## 4. Comparison Baselines

| Baseline | Description | Presence F1 | Motion MAE | Identity leak |
|----------|-------------|------------|-----------|--------------|
| Raw CSI pipeline | Existing wifi-densepose pipeline (no BFLD) | ~0.95 (est.) | ~0.08 (est.) | Unquantified — no risk gating |
| BFI-only (no BFLD) | Wi-BFI + threshold presence | ~0.82 (from LeakyBeam) | N/A | Angle matrices published |
| BFI+CSI fusion (no BFLD) | Combined pipeline, ungated | ~0.97 (est.) | ~0.06 (est.) | Unquantified |
| **BFLD (BFI+CSI, class 2)** | Full BFLD with anonymous privacy class | target 0.93 | target 0.10 | 0% (class 2 gate) |
| BFLD (BFI-only, class 2) | BFLD without CSI input (AC7) | target 0.85 | target 0.12 | 0% (class 2 gate) |

The BFLD privacy-class guarantee reduces the raw sensing accuracy by a small margin
versus an ungated BFI+CSI pipeline (target F1 0.93 vs estimated 0.97). This is the
explicit trade-off: identity safety for a modest utility cost.

---

## 5. Continuous Evaluation in CI

Three tests run on every PR that touches the BFLD crate:

1. **Deterministic hash test** (AC6): same input → same output across platforms.
2. **Privacy-mode field suppression fuzz** (AC5): 1,000 random inputs → no identity
   fields in class-2 output.
3. **Latency smoke test** (AC2): 100-frame replay → first presence event < 200 ms
   (tighter than the 1s AC target, to keep CI fast).
