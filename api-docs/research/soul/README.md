# Soul Signature — Research Specification

**Status:** Research Specification (Pre-Implementation)
**Date:** 2026-05-24
**Maintainer:** ruv

---

## What Is a Soul Signature

A Soul Signature is a fused multi-modal biometric identity vector derived entirely
from passive electromagnetic measurement of a person inside a room equipped with
WiFi-DensePose / RuView sensing nodes. No wearable, no camera, no explicit
scan-time consent moment is required for recognition once a person has enrolled.

The word "soul" is deliberate product framing for a scientifically defensible concept:
the same relationship a fingerprint bears to identity in forensic science, or FaceID
to phone authentication, but extended to a new sensing dimension — passive RF at
distance, through walls, at room scale. Seven orthogonal electromagnetic observables,
fused into a single content-addressed RVF graph file, constitute the signature.

The claim is not mystical. Every channel is grounded in published physics and prior
WiFi sensing literature. Every assertion about discriminative power either cites a
peer-reviewed result or is explicitly marked "open research; baseline TBD."

---

## What a Soul Signature Is NOT

- It is NOT a replacement for fingerprint scanners, iris scanners, or FaceID on
  accuracy-per-attempt measures. Current RF biometrics are less mature than those
  modalities. See `security.md` for the honest error-rate picture.
- It is NOT a single number, hash, or deterministic bit string. It is a
  probabilistic match against a stored graph with a calibrated false-accept rate.
- It is NOT medically diagnostic. It detects biophysical proxies, not conditions.
  "Gait asymmetry increased 18% over 14 days" is the output, never "Parkinson's."
- It is NOT equivalent to explicit-consent biometrics in regulated contexts. GDPR
  and HIPAA modes are defined and mandatory for healthcare deployments.
- It is NOT currently deployable as a legal evidence instrument.
- It is NOT snake oil, energy healing, or anything outside measurable electrophysics.

---

## Document Map

| File | Contents |
|------|----------|
| `specification.md` | Typed RVF graph schema; all node types, edge types, serialization format; aggregator vs stored profile distinction |
| `scanning-process.md` | Structured 60-second enrollment protocol; hardware requirements; quality gates; fast-scan and continuous modes; re-scan cadence |
| `security.md` | Full threat model; five adversaries; mitigations; cryptographic primitive choices; GDPR/HIPAA mode; open research items |
| `references.md` | All cited ADRs, papers, datasets, standards |

---

## Conceptual Graph (ASCII)

The following depicts one example soul signature as a graph stored in a single
RVF container. Each box is an RVF node (a SEG_EMBED or SEG_META segment). Each
arrow is a typed edge stored in the graph manifest.

```
  +-----------------------+
  |   AETHER_Embedding    |  128-dim f32, L2-normalized (ADR-024)
  |   contrastive CSI     |  HNSW-searchable via ruvector-core
  |   backbone embedding  |
  +----------+------------+
             |  derived_from
             v
  +-----------+-----------+          +------------------------+
  | FieldModel_Residual   +---fuses--+  Subcarrier_Reflection  |
  | ADR-030 perturbation  |          |  per-angle multipath   |
  | eigenmode projection  |          |  amplitude + phase     |
  +----------+------------+          +------------------------+
             |  correlates_with
             v
  +----------+------------+          +------------------------+
  |  Cardiac_HR_Profile   +--links---+  Cardiac_Waveform_     |
  |  baseline_bpm, HRV_LF |          |  Morphology (wavelet   |
  |  HRV_HF, rhythm_class |          |  coefficients)         |
  +----------+------------+          +------------------------+
             |  temporally_colocated
             v
  +----------+------------+
  | Respiratory_Pattern   |
  |  baseline_bpm, depth, |
  |  apnea_index, HRV_RSA |
  +----------+------------+
             |  temporally_colocated
             v
  +----------+------------+          +------------------------+
  |   Gait_Timing         +--links---+  Skeletal_Proportions  |
  |  cadence, stride_var, |          |  torso/limb ratios     |
  |  double_support_pct,  |          |  from ADR-079 keypoints |
  |  asymmetry_index      |          +------------------------+
  +----------+------------+
             |  attested_by
             v
  +----------+------------+
  |   WitnessChain        |  Ed25519 over (content_hash ||
  |   ADR-110 attestation |  timestamp || device_id) per ADR-110
  +-----------------------+
```

File naming convention: `signature-<sha256-of-rvf-content>.rvf`

---

## Implementation Status

This is a **research specification**. None of the soul-signature-specific graph
container logic is implemented yet. The constituent ADRs (AETHER, MERIDIAN,
RuvSense field model, ADR-039 vitals, ADR-110 witness chain) provide the substrate.
The soul signature is the composition layer above them.

A future implementation ADR should reference this document and assign acceptance
tests derived from the quality gates defined in `scanning-process.md`.
