# BFLD Research Bundle — Beamforming Feedback Layer for Detection

BFLD is the safety layer that detects when RF data becomes identifying. It sits between
raw 802.11 beamforming feedback (BFI) and every downstream consumer — home automation,
MQTT, Matter, cloud — measuring the identity-leakage potential of each frame and gating
what leaves the node. It does not produce identity; it guards against accidental or
adversarial exposure of identity.

---

## Table of Contents

| File | Purpose |
|------|---------|
| [01-sota-survey.md](01-sota-survey.md) | State-of-the-art literature: BFI vs CSI, attack tooling, identity-inference research, privacy-preserving techniques |
| [02-soul.md](02-soul.md) | Architectural intent, ethical stance, three non-negotiable invariants |
| [03-security-threat-model.md](03-security-threat-model.md) | Adversary classes, attack trees, mitigations, trust-boundary diagram, per-privacy-class analysis |
| [04-privacy-gating.md](04-privacy-gating.md) | privacy_class byte semantics, hash rotation algorithm, embedding lifecycle, wire-format diffs |
| [05-automation-integration.md](05-automation-integration.md) | Home Assistant entities, Matter clusters, MQTT ACLs, cognitum federation |
| [06-implementation-plan.md](06-implementation-plan.md) | New crate layout, reuse map, ESP32 additions, test plan, phased rollout |
| [07-benchmarks-and-evaluation.md](07-benchmarks-and-evaluation.md) | Datasets, metrics, red-team protocol, comparison baselines |
| [08-adr-draft.md](08-adr-draft.md) | Draft ADR-118 for formal project adoption |
| [09-github-issue.md](09-github-issue.md) | GitHub issue draft for tracking implementation |
| [10-gist.md](10-gist.md) | Public-facing one-pager / blog summary |

---

## Executive Summary

1. **Problem.** IEEE 802.11ac/ax beamforming feedback (BFI) — the compressed angle matrices
   (Phi/Psi, Givens rotation) exchanged between client and AP — is transmitted unencrypted
   on the management plane. Academic work (BFId at ACM CCS 2025, LeakyBeam at NDSS 2025)
   demonstrates that a passive sniffer with commodity hardware can re-identify individuals
   and infer occupancy through walls using only these frames. Existing CSI-based sensing
   pipelines have no explicit layer to detect when their output crosses from "motion event"
   into "identity record."

2. **Approach.** BFLD is a new crate (`wifi-densepose-bfld`) that wraps the BFI extraction
   and normalization path in an identity-leakage estimator. Every output frame carries a
   computed `identity_risk_score` and a `privacy_class` byte; downstream consumers decide
   whether to act based on those tags rather than on raw measurements.

3. **Novel contribution.** BFLD does not try to suppress identity inference — it tries to
   *measure* it continuously and make the measurement explicit in every event. This
   transforms a latent, silent risk into an observable, auditable signal. The combination
   of per-day per-site hash rotation and a local-only identity embedding creates structural
   impossibility of cross-site re-identification — not merely a policy promise.

4. **Security posture.** Raw BFI never leaves the node. Identity embeddings live only in
   an in-RAM ring buffer. The rf_signature_hash rotates daily using a per-site blake3
   keyed-hash that is never transmitted. Matter and HA expose only presence, motion, and
   person_count — never risk scores or embeddings.

5. **Integration plan.** Six phases: P1 frame format + extractor stub, P2 feature
   extraction + identity_risk, P3 privacy gate + MQTT, P4 HA integration, P5 Matter
   exposure, P6 cognitum federation. Each phase maps to a numbered acceptance criterion.
   The crate slots into the existing workspace between `wifi-densepose-signal` and
   `wifi-densepose-sensing-server`.
