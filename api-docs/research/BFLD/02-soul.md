# BFLD Soul — Architectural Intent and Ethical Stance

## 1. The Central Metaphor: Immune System, Not Surveillance Lens

An immune system does not catalog every pathogen it encounters. It classifies threats
by type, responds proportionally, and keeps its detailed records local to the organism.
When the immune system flags a cell as dangerous, it does not broadcast the cell's
identity to the outside world — it takes local action.

BFLD is built around this same principle. Its job is to detect when RF data is crossing
from the realm of "ambient sensing" into the realm of "identity record" — and to respond
locally: raise the risk score, restrict what leaves the node, rotate identifiers. It does
not produce identity; it guards against the accidental production of identity.

This distinction matters because the same physical signal that drives BFLD's presence
detection is also the signal that academic attackers (BFId, LeakyBeam) exploit for
re-identification. BFLD cannot suppress the underlying physics. What it can do is make
the node's *output* non-identifying, even when the node's *input* is capable of
supporting identification.

---

## 2. Distinguishing Identity from the Rest of WiFi Sensing

WiFi sensing produces a spectrum of information:

| Output | Privacy class | Reversibility |
|--------|--------------|---------------|
| Presence (yes/no) | 2 — anonymous | Not reversible to identity |
| Motion magnitude (0..1) | 1 — derived | Not reversible to identity |
| Person count (integer) | 1 — derived | Not reversible to identity |
| Zone activity | 1 — derived | Not reversible to identity |
| Identity risk score | 1 — derived | Risk score, not identity |
| RF signature hash | 1 — derived | Hash rotates daily; not reversible |
| Identity embedding | 0 — raw | Directly reversible to biometric |
| Raw BFI matrix | 0 — raw | Directly reversible to biometric |

BFLD's design follows this table structurally: the outputs in privacy class 0 never
leave the node. The outputs in class 1 leave the node only after explicit operator opt-in
for the sensitive ones (identity_risk_score). The outputs in class 2 flow freely.

This table is not a policy list — it is wired into the frame format. The `privacy_class`
byte in every `BfldFrame` is checked at the emitter boundary before any byte leaves the
node. Code that wants to send class-0 data must positively bypass a compile-time safety
check, not merely forget to set a flag.

---

## 3. Three Non-Negotiable Invariants

These are not configurable options. They are structural properties of BFLD that
hold regardless of operator configuration:

### Invariant 1: Raw BFI Never Leaves the Node

The BFI matrix, once ingested by the BFLD extractor, is consumed locally and never
serialized to any outbound channel. This is enforced in two ways:

1. The `BfldFrame` struct's `bfi_matrix` field is not part of the serializable payload
   — it exists only as a private field in `extractor.rs` and is dropped after
   feature extraction completes.
2. The MQTT emitter (`mqtt.rs`) has no code path that serializes a BFI matrix.
   The `ruview/<node_id>/bfld/raw/state` topic is disabled by default and, when
   enabled, publishes only a metadata summary (subcarrier count, timestamp, SNR range),
   not the angle matrices.

### Invariant 2: Identity Embedding Is Local-Only

The embedding computed by the RuVector pipeline (used to calculate `identity_risk_score`)
lives in an in-RAM ring buffer with a configurable retention window (default: 10 minutes).
It is never written to disk. It is never serialized to any MQTT topic. It is never
included in any `BfldFrame` payload even at `privacy_class = 0` — raw means raw angles,
not the derived embedding.

The mathematical property that enables this: `identity_risk_score` can be computed as a
scalar from the embedding (separability × temporal_stability × cross_perspective_
consistency × sample_confidence) without revealing the embedding itself. The score is a
projection onto a scalar; the full vector is not required by any downstream consumer.

### Invariant 3: Cross-Site Identity Matching Is Structurally Impossible

The `rf_signature_hash` is computed as:

    blake3(site_salt ‖ day_epoch ‖ ephemeral_features)

where `site_salt` is a secret generated at first boot, stored in NVS, and never
transmitted. Two BFLD nodes at two different sites will produce hashes in disjoint
hash spaces by construction. Even an adversary who obtains the hash stream from
both nodes cannot determine whether the same person visited both sites, because the
site_salt is unknown and different.

The daily rotation (`day_epoch` = floor(timestamp_ns / 86400e9)) means that even within
a single site, the hash of the same person changes each day. Hashes older than 24 hours
have zero correlation with hashes produced today.

This is structural impossibility, not policy. The invariant holds even if the operator
misconfigures the system, because it derives from the cryptographic property of blake3
with a secret key, not from access-control rules.

---

## 4. Relationship to RuView's Ambient Intelligence Positioning

The project memory records RuView's positioning as "ambient intelligence platform, not
sensor; packaging (HA, Docker, mDNS, blueprints) is the bottleneck." This framing is
load-bearing for BFLD's design.

A "sensor" in the Home Assistant model is a device that reports measurements. A "sensor"
is allowed to identify who is present — facial recognition cameras are sensors. BFLD
explicitly rejects this model: the node is an ambient intelligence node that knows
something about the environment (motion, occupancy, activity level) but structurally
cannot know *who* is in the environment.

This positioning enables deployment in spaces where identity-tracking would be
unacceptable: shared workspaces, guest accommodations, hotel rooms, care facilities.
The argument to an operator at a care facility is not "trust us, we won't log who your
patients are." It is: "the system is architecturally incapable of logging who your
patients are, because the identifier rotates daily with a site-specific secret we don't
hold."

---

## 5. Why This Layer Must Exist Before WiFi 7 Ships

802.11be (Wi-Fi 7) is entering mass market deployment in 2025–2026. It introduces
multi-link operation (MLO), which dramatically increases the frequency of beamforming
sounding exchanges. Where 802.11ax sonding might occur at 10–40 Hz, MLO sounding on
multiple links simultaneously could produce 3–5× more CBFR frames per second.

More frames means more training data for identity classifiers. The BFId result at 5
seconds of 802.11ac data will almost certainly improve with 5 seconds of 802.11be MLO
data. The attack surface is not static.

BFLD's frame format (magic 0xBF1D_0001, version byte for extension) is designed to
remain valid across protocol generations. The feature extraction modules are pluggable:
a WiFi 7 BFI extractor can be added without changing the privacy gate, the hash rotation,
or the MQTT emitter. The invariants remain invariant.

The window to establish safe defaults is now, before the installed base is hundreds of
millions of unprotected nodes. BFLD is the layer that carries those safe defaults into
every deployment from day one.
