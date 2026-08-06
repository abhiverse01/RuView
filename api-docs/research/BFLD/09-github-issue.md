# GitHub Issue Draft

**Title**: feat: BFLD — Beamforming Feedback Layer for Detection (privacy-gated WiFi sensing)

**Labels**: `enhancement`, `privacy`, `security`, `area/signal`, `area/firmware`

**Milestone**: (TBD — suggest: v0.8.0)

---

## Summary

Add a new crate `wifi-densepose-bfld` that turns raw 802.11 Beamforming Feedback
Information (BFI) into bounded, privacy-gated sensing outputs. BFLD detects when RF
data crosses from "ambient sensing" into "identity record" and structurally prevents
identity-correlated data from leaving the node.

This is the safety layer that was missing from the CSI pipeline. As passive BFI sniffing
tools (Wi-BFI, PicoScenes) become widely available and academic attacks (BFId at ACM CCS
2025, LeakyBeam at NDSS 2025) demonstrate >90% re-identification from commodity WiFi,
the wifi-densepose ecosystem needs an explicit privacy layer before scaling deployment.

## Motivation

1. **BFI is plaintext and passively sniffable.** IEEE 802.11ac/ax CBFR frames are
   transmitted before WPA2/WPA3 encryption is applied. Any nearby device in monitor mode
   can capture them (NDSS 2025: https://www.ndss-symposium.org/ndss-paper/lend-me-your-beam-privacy-implications-of-plaintext-beamforming-feedback-in-wifi/).

2. **BFI enables re-identification.** The KIT BFId paper (ACM CCS 2025:
   https://dl.acm.org/doi/10.1145/3719027.3765062) demonstrates >90% identity
   recognition from 5 seconds of BFI, from a dataset of 197 individuals, using only
   the Phi/Psi Givens rotation angles.

3. **The existing pipeline has no identity-leakage measurement.** The rvCSI pipeline
   produces presence/motion/pose events without any indication of whether those outputs
   were derived from identity-discriminative data. An operator deploying in a care
   facility or shared office has no way to verify the system is behaving anonymously.

4. **WiFi 7 will make this worse.** 802.11be (Wi-Fi 7) multi-link operation increases
   sounding frequency 3–5×. The attack surface is not static.

## Proposed Solution

New crate at `v2/crates/wifi-densepose-bfld/` with the following pipeline:

```
BFI capture (CBFR frames, Pi 5 / Nexmon monitor mode)
    → BFI extractor (Phi/Psi parser, 802.11ac/ax)
    → Normalization + temporal windowing
    → Feature extraction (9 named features)
    → Identity risk engine (in-RAM embeddings, coherence gate)
    → Privacy gate (privacy_class byte, field masking)
    → MQTT emitter (per-class topic routing)
```

Three structural invariants (not configurable, not policy):
1. Raw BFI never leaves the node.
2. Identity embedding is in-RAM-only (VecDeque, never persisted).
3. Cross-site identity matching is cryptographically impossible via per-site BLAKE3
   keyed hash with daily rotation.

Output events published on `ruview/<node_id>/bfld/{presence,motion,person_count,...}/state`.

Matter and HA expose only: presence, motion, person_count. Identity fields are rejected
at both boundaries.

## Acceptance Criteria

- [ ] **AC1**: Parser handles 802.11ac VHT and 802.11ax HE CBFR frames at 20/40/80/160 MHz,
  2×2 through 4×4 MIMO.
- [ ] **AC2**: Presence detection latency ≤ 1s p95 from first non-empty BFI frame in
  a new occupancy event.
- [ ] **AC3**: Motion score published at ≥ 1 Hz on `ruview/<node_id>/bfld/motion/state`
  during sustained occupancy.
- [ ] **AC4**: Raw BFI bytes (Phi/Psi angle matrices) are never present in any
  serialized output at any `privacy_class` value.
- [ ] **AC5**: Privacy mode suppresses all identity-derived fields (`identity_risk_score`,
  `rf_signature_hash`, `identity_embedding`) from all outbound events.
- [ ] **AC6**: Identical `BfiCapture` input → bit-identical `BfldFrame` output
  (deterministic, cross-platform).
- [ ] **AC7**: Pipeline produces valid `BfldEvent` with `csi_matrix = None` (BFI-only
  mode), without panic or significant accuracy degradation.

## References

- BFId paper: https://dl.acm.org/doi/10.1145/3719027.3765062
- KIT BFId dataset: https://ps.tm.kit.edu/english/bfid-dataset/index.php
- LeakyBeam (NDSS 2025): https://www.ndss-symposium.org/ndss-paper/lend-me-your-beam-privacy-implications-of-plaintext-beamforming-feedback-in-wifi/
- Wi-BFI tool: https://arxiv.org/abs/2309.04408
- Protecting activity signatures in CSI feedback: https://arxiv.org/pdf/2512.18529
- Research bundle: `docs/research/BFLD/` (this repo)
- Draft ADR: `docs/research/BFLD/08-adr-draft.md` → ADR-118

## Out of Scope

- Preventing passive BFI capture by external attackers (hardware-level problem, not
  software).
- Differential privacy noise injection (noted as future extension in ADR-118).
- Federated identity learning (local-only is sufficient for the current use case).
- BFI capture directly from ESP32-S3 firmware (Espressif API does not expose CBFR;
  host-side Pi 5 / Nexmon capture is the implementation path).
- WiFi 7 / 802.11be multi-link BFI (frame format versioning accommodates it; not
  in scope for v1 implementation).

## Related Issues / PRs

- ADR-028 witness bundle (ref: this repo's `docs/WITNESS-LOG-028.md`)
- ADR-115 HA integration (21 entities — BFLD adds 6 more)
- ADR-116 Matter seed packaging (`cog-ha-matter` crate needs Matter boundary update)
- ADR-117 pip modernization (PyO3 pattern reused for BFLD Python bindings)
- rvCSI platform (ADR-095/096) — Nexmon adapter shared with BFLD BFI capture path
