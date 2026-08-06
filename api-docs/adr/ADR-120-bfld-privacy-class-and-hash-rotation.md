# ADR-120: BFLD Privacy Class and Hash Rotation

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-24 |
| **Deciders** | ruv |
| **Parent** | [ADR-118](ADR-118-bfld-beamforming-feedback-layer-for-detection.md) |
| **Relates to** | [ADR-027](ADR-027-cross-environment-domain-generalization.md) (MERIDIAN no-cross-site), [ADR-032](ADR-032-multistatic-mesh-security-hardening.md) (mesh security), [ADR-106](ADR-106-dp-sgd-and-primitive-isolation.md) (primitive isolation), [ADR-115](ADR-115-home-assistant-integration.md) (privacy mode) |
| **Companion research** | [`docs/research/soul/`](../research/soul/) — Soul Signature operates at `privacy_class = 1` (derived). §2.7 defines the dual-ID-space contract. |
| **Tracking issue** | TBD |

---

## 1. Context

ADR-118 declares three structural invariants for BFLD:

- **I1**: Raw BFI never exits the node.
- **I2**: Identity embedding is in-RAM-only.
- **I3**: Cross-site identity correlation is cryptographically impossible.

I1/I2 are enforced by sink typing and module visibility (ADR-119 §2.3). I3 requires a hash-rotation scheme that makes the same physical person produce **different** `rf_signature_hash` values across sites and across day boundaries, without any out-of-band coordination between sites.

The existing `HA-PRIVACY` mode in ADR-115 already toggles between "full" and "anonymous" surfaces, but at a per-event granularity — not at a per-byte-field granularity. BFLD requires the latter because the `BfldFrame` payload mixes sensing data (publishable) and identity-derived data (non-publishable) in the same struct.

The BFId paper (KIT, ACM CCS 2025) demonstrates that even a few minutes of BFI capture across the same site is sufficient to build a persistent biometric. The mitigation must be **structural**, not policy-dependent.

---

## 2. Decision

### 2.1 The four privacy classes

A single `privacy_class: u8` byte in the `BfldFrame` header (ADR-119 §2.1) selects one of four classes. The crate enforces field availability statically through marker types.

| Class | Name | Use case | Available fields |
|-------|------|----------|------------------|
| **0** | `raw` | Local-only research, never networked | All fields, full-precision BFI matrix, identity embedding |
| **1** | `derived` | Operator-acknowledged research over LAN | Downsampled angle matrix, full features, identity_risk_score, identity_embedding |
| **2** | `anonymous` (**default**) | Production deployment | Aggregate sensing only: presence, motion, person_count, zone_id, confidence |
| **3** | `restricted` | Care-home / regulated deployment | Class 2 minus `identity_risk_score` and `rf_signature_hash` |

Default for new RuView nodes is class **2**. Operators must explicitly opt-down to class 1 via the existing `--research-mode` flag (ADR-115 §7); class 0 is reserved for `cargo test` and is unreachable from `wifi-densepose-sensing-server`.

### 2.2 Enforcement via marker types

```rust
pub trait Sink {}

pub trait LocalSink: Sink {}     // Allowed: classes 0,1,2,3
pub trait NetworkSink: Sink {}   // Allowed: classes 1,2,3 (NOT class 0)
pub trait MatterSink: NetworkSink {}  // Allowed: class 2,3 + cluster-filter (ADR-122)

impl Emitter {
    pub fn publish<S: NetworkSink>(&self, sink: &S, frame: BfldFrame)
        -> Result<(), BfldError>
    {
        if frame.header.privacy_class == 0 {
            return Err(BfldError::PrivacyViolation {
                reason: "class 0 to NetworkSink",
            });
        }
        // ... serialize and write
    }
}
```

The compiler refuses to call `publish` on a sink that doesn't impl `NetworkSink` with a class-0 frame because the runtime check is paired with a sink-marker check. Cross-sink frame routing requires an explicit class transition (see §2.4).

### 2.3 BLAKE3 keyed hash rotation for `rf_signature_hash`

The signature hash is computed as:

```rust
pub fn rf_signature_hash(
    site_salt: &[u8; 32],       // generated on first boot, persisted in TPM/KMS
    day_epoch: u32,             // floor(unix_time_utc / 86400)
    features: &IdentityFeatures,
) -> Hash {
    let mut hasher = blake3::Hasher::new_keyed(site_salt);
    hasher.update(&day_epoch.to_le_bytes());
    hasher.update(&features.canonical_bytes());
    hasher.finalize()
}
```

**Structural cross-site isolation**: because `site_salt` is a 256-bit random secret unique to each node and never transmitted, two sites observing the same physical person produce uncorrelated hashes. There is no key the operator (or an attacker who compromises one node) can use to bridge sites. This is stronger than a policy-based "do not share" rule because the bridge **cannot be computed**.

**Daily rotation**: `day_epoch` flipping at UTC midnight forces the hash of the same person to change once per day. Multi-day correlation requires re-acquiring the biometric, which the rotation actively breaks.

### 2.4 Class-transition transformer

The only way a high-class frame becomes a lower-class frame is through `PrivacyGate::demote(frame, target_class)`. This function:

1. Asserts the target class is strictly higher number than (or equal to) the input class.
2. Zeroes the disallowed fields with `subtle::Zeroize`.
3. Re-computes `payload_crc32`.
4. Returns the new frame.

There is no `promote` operation — a class-2 frame cannot be turned back into a class-1 frame, because the dropped fields were not retained anywhere reachable from the gate.

### 2.5 `identity_embedding` lifecycle

The embedding (output of the AETHER encoder, ADR-024) is held in a `subtle::Zeroizing<[f32; 128]>` ring buffer of 64 entries (≈30 KB). Entries are:

1. Written by the encoder on each capture window.
2. Consumed by `identity_risk_score` computation (ADR-121).
3. **Never** written to disk, MQTT, or any other I/O sink — there is no `Serialize` impl on the type.
4. Overwritten by the ring (FIFO).

A compile-time `#[forbid(serde::Serialize)]` lint on `IdentityEmbedding` ensures a future PR cannot accidentally add a `Serialize` derive.

### 2.6 Default-deny field classification

Every new field added to `BfldFrame` or `BfldEvent` must be tagged with `#[must_classify]` (a custom attribute macro). The macro fails compilation if the field is not listed in the per-class allow-list table. This forces future contributors to make an explicit privacy decision on every new field.

### 2.7 Dual-ID-space contract for Soul Signature deployments

Soul Signature (`docs/research/soul/`) is a consent-based biometric system that *intentionally* produces long-lived per-person identity. It cannot operate at the default class 2 — the identity_embedding it needs is structurally absent there. The contract:

| Deployment mode | `privacy_class` | ID space for unenrolled bystanders | ID space for enrolled persons |
|---|---|---|---|
| Default BFLD-only | 2 (anonymous) | Daily-rotated `rf_signature_hash` | n/a — no enrollment |
| Soul Signature opt-in | **1 (derived)** | Daily-rotated `rf_signature_hash` (unchanged) | Long-lived opaque `person_id` from Soul Signature graph |
| Restricted / care-home | 3 (restricted) | Suppressed | n/a — Soul Signature **disabled** at class 3 |

Two ID spaces coexist with **no collision**: the rotating hash is the privacy-preserving identifier for everyone *not* on the consent roster; the stable `person_id` is reserved for enrolled subjects under their own GDPR/HIPAA mode. Soul Signature's `match_against_enrolled()` function consumes only the in-RAM `identity_embedding` (I2 still holds) and emits a `person_id` plus a calibrated similarity score; it never writes the embedding to disk or the wire. The class-1 requirement is enforced statically: the Soul Signature match API takes a `&IdentityEmbedding` parameter, which is only constructible when the BFLD crate is compiled with `--features soul-signature` against a class-1 frame.

---

## 3. Consequences

### Positive

- Cross-site identity correlation is **computationally impossible**, not merely "prohibited by policy". This is the strongest form of privacy guarantee available without a TEE.
- Default-deny via `#[must_classify]` prevents the common pattern of "a new field shipped, then six months later we noticed it was identity-leaky".
- `identity_embedding` cannot be serialized by accident — the type system carries the constraint.
- The class transition transformer makes the data lifecycle explicit and auditable.

### Negative

- `site_salt` storage requires either a TPM (ADR-095/096 rvCSI platform feature gap) or a secrets file with strict mode. Loss of `site_salt` makes historical witness comparisons impossible — by design, but a documentation hazard.
- `#[must_classify]` is a custom proc-macro; another moving part in the build.
- Operators wanting multi-day analytics must work in aggregates only, not on per-individual signatures.

### Neutral

- Class 0 is `cargo test`-only. Some CI runners may need an explicit feature flag to compile class-0 paths.

---

## 4. Alternatives Considered

### Alt 1: Single boolean `privacy_mode` flag (status quo from ADR-115)

Rejected: insufficient granularity. The frame mixes publishable sensing with non-publishable identity, so the gate must operate at field-level, not event-level.

### Alt 2: SHA-256 instead of BLAKE3

Rejected: BLAKE3 keyed-hash mode is ~5× faster on the ESP32-S3 / Cortex-M cores and the security margin is equivalent for this use case. SHA-256 has no keyed-hash mode (HMAC-SHA256 is the alternative; works but is slower).

### Alt 3: Hash rotation on the hour, not the day

Rejected: hourly rotation breaks legitimate "person was here in the morning, came back in the afternoon" use-cases that operators may want. Day boundary is the compromise.

### Alt 4: Per-event nonces instead of daily epoch

Rejected: per-event nonces would force the consumer to track which events came from the same person within a session, which leaks identity information by structure. The day epoch preserves a coarse temporal grouping without leaking finer-grained identity.

---

## 5. Acceptance Criteria

- [ ] **AC1**: Calling `Emitter::publish` with a `privacy_class = 0` frame on a `NetworkSink` returns `BfldError::PrivacyViolation`.
- [ ] **AC2**: Two BFLD nodes with different `site_salt` values observing the same simulated person produce `rf_signature_hash` values whose Hamming distance is ≥ 120 bits over 100 trials (statistical isolation test).
- [ ] **AC3**: A frame with `privacy_class = 3` has both `identity_risk_score` and `rf_signature_hash` absent from the serialized payload.
- [ ] **AC4**: `PrivacyGate::demote(class_1_frame, target=0)` fails to compile (compile-fail test).
- [ ] **AC5**: A PR adding a new field to `BfldEvent` without `#[must_classify]` fails the build.
- [ ] **AC6**: `IdentityEmbedding` has no `Serialize` impl reachable from any public function.
- [ ] **AC7**: Dropping an `IdentityEmbedding` value zeroizes its memory (verified by a debugger-readable test under `cargo test --features zeroize-validation`).

---

## 6. References

- ADR-118 (umbrella)
- ADR-119 (frame format; `privacy_class` byte location)
- KIT BFId (ACM CCS 2025): https://publikationen.bibliothek.kit.edu/1000185756
- NDSS LeakyBeam (2025): https://www.ndss-symposium.org/wp-content/uploads/2025-5-paper.pdf
- BLAKE3 keyed-hash: https://github.com/BLAKE3-team/BLAKE3
- `subtle::Zeroize` for memory hygiene
