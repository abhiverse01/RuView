# BFLD: The Privacy Layer Your WiFi Sensing Stack Has Been Missing

Your WiFi router is broadcasting your identity in plaintext. Here is the layer that
catches it.

---

## The Problem

Every time your phone or laptop connects to a WiFi 5 or WiFi 6 router, it periodically
transmits a Beamforming Feedback Report (CBFR frame). This frame contains the compressed
channel matrix the router needs to aim its antennas at your device. The compression uses
Givens rotations — a pair of angles (Phi and Psi) per active subcarrier — that encode
the spatial geometry of the wireless channel around your body.

Here is the catch: these frames are transmitted before WPA2/WPA3 encryption is applied.
They are plaintext management frames, passively readable by any WiFi adapter in monitor
mode within roughly 20 meters.

Two papers published in 2024–2025 confirm the threat is real:

- **BFId** (KIT, ACM CCS 2025): re-identifies 197 people from beamforming feedback alone,
  >90% accuracy from just 5 seconds of capture. Tools needed: a WiFi adapter, a pip
  install, and no access to the target network.
  (https://dl.acm.org/doi/10.1145/3719027.3765062)

- **LeakyBeam** (Zhejiang U. / NTU / KAIST, NDSS 2025): detects occupancy through walls
  at 20 m range using beamforming feedback with 82.7% accuracy.
  (https://www.ndss-symposium.org/ndss-paper/lend-me-your-beam-privacy-implications-of-plaintext-beamforming-feedback-in-wifi/)

WiFi sensing systems — including this project — process these same signals to detect
presence, count people, and track motion. Without a privacy layer, there is no way to
know whether the sensing output is derived from anonymizable motion data or from
identity-discriminative data.

---

## What BFLD Does

BFLD (Beamforming Feedback Layer for Detection) is a new Rust crate in the
wifi-densepose workspace that adds one thing: an explicit, continuous measurement of
whether the beamforming data currently being processed is capable of identifying
individuals.

It outputs a small, structured event on every sensing cycle:

```json
{
  "timestamp_ns": 1748092800000000000,
  "presence": true,
  "motion": 0.42,
  "person_count": 1,
  "identity_risk_score": 0.71,
  "rf_signature_hash": "a3f2c1...e9b4",
  "zone_id": "living_room",
  "confidence": 0.88,
  "privacy_class": 1
}
```

High `identity_risk_score` (approaching 1.0) means the current sensing environment is
producing data from which an attacker could re-identify individuals. Low score means
the data is effectively anonymous.

The score is computed from four components: how separable the current RF embedding is
from a population distribution, how stable that separability is over time, how
consistent it is across multiple sensor viewpoints, and how confident the current sample
is. Multiply them together, clamp to [0, 1].

---

## Three Invariants That Cannot Be Turned Off

BFLD enforces three properties structurally — not as settings, not as policies:

**1. Raw BFI never leaves the node.** The Phi/Psi angle matrices are consumed locally
and dropped after feature extraction. They are not in the wire format. They are not in
the MQTT payload. There is no code path to serialize them outbound.

**2. Identity embeddings are RAM-only.** The vector embedding used to compute the risk
score lives in a fixed-size ring buffer (default: 10 minutes). It is never written to
disk. When the node restarts, the buffer is gone.

**3. Cross-site re-identification is cryptographically impossible.** The
`rf_signature_hash` is computed with a per-site secret key (generated at first boot,
stored in local NVS, never transmitted) and a per-day epoch. Two nodes at two
different sites, even receiving signals from the same person on the same day, produce
hash values in completely disjoint hash spaces. No amount of hash-list comparison can
reveal a cross-site visit.

---

## What Reaches Home Assistant and Matter

BFLD publishes to MQTT and HA. The following entities reach HA:

- `binary_sensor.bfld_presence`
- `sensor.bfld_motion`
- `sensor.bfld_person_count`
- `sensor.bfld_confidence`

The Matter bridge exposes only OccupancySensing (presence) and motion. Identity risk
score, rf_signature_hash, and all raw fields are rejected at both the HA and Matter
boundaries.

---

## Seven Acceptance Criteria

The implementation is done when these seven tests pass:

1. Parse 802.11ac and 802.11ax BFI at 20–160 MHz bandwidth, 2×2 to 4×4 MIMO.
2. Presence latency ≤ 1 second p95.
3. Motion published at ≥ 1 Hz.
4. Raw BFI bytes absent from all output (verified by fuzz test).
5. Privacy mode suppresses all identity fields.
6. Identical input → identical output hash (cross-platform determinism).
7. Pipeline runs without CSI input (BFI-only mode).

---

## BFLD Is an Immune System, Not a Surveillance Lens

The framing matters. BFLD does not produce identity — it measures identity risk and
uses that measurement to gate what leaves the node. An immune system does not broadcast
the identity of pathogens it encounters; it classifies, responds locally, and keeps
detailed records inside the organism.

WiFi 7 / 802.11be is deploying now. Multi-link operation will increase beamforming
sounding frequency 3–5x. The passive attack surface will grow. The time to establish
safe defaults in WiFi sensing stacks is before that installed base is in place.

BFLD is that default.

Full research bundle: `docs/research/BFLD/` in the wifi-densepose repository.
Draft ADR: `docs/research/BFLD/08-adr-draft.md` (ADR-118).
