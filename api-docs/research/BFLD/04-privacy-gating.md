# BFLD Privacy Gating — Mechanisms in Depth

## 1. The privacy_class Byte: Concrete Data Exposure Tables

The `privacy_class` byte is the single authoritative classifier for what a BFLD node
is permitted to emit. It is set by the privacy gate module (`privacy_gate.rs`) on every
outbound `BfldFrame` based on the computed `identity_risk_score` and operator configuration.

### Class 0 — raw

Intended exclusively for local research captures and red-team validation. Not a
deployable configuration.

| Field | Published | Notes |
|-------|-----------|-------|
| presence | Yes | Boolean |
| motion | Yes | 0..1 float |
| person_count | Yes | u8 |
| identity_risk_score | Yes | f32 |
| rf_signature_hash | Yes | Rotated blake3, 32 bytes hex |
| zone_activity | Yes | |
| confidence | Yes | |
| compressed_angle_matrix | Yes | Phi/Psi per subcarrier — the sensitive surface |
| amplitude_proxy | Yes | |
| phase_proxy | Yes | |
| snr_vector | Yes | |
| bfi_matrix (raw) | NEVER | Dropped before serialization; not in wire format |
| identity_embedding | NEVER | Local RAM only; not in wire format |

### Class 1 — derived

Default for operator-opted-in diagnostics. Includes identity_risk_score and hash but
no angle matrices.

| Field | Published | Notes |
|-------|-----------|-------|
| presence | Yes | |
| motion | Yes | |
| person_count | Yes | |
| identity_risk_score | Yes | Diagnostic; not in HA default entities |
| rf_signature_hash | Yes | Rotated hash only |
| zone_activity | Yes | |
| confidence | Yes | |
| compressed_angle_matrix | No | Zeroed |
| amplitude_proxy | No | |
| phase_proxy | No | |
| snr_vector | Yes | Per-stream aggregate only |
| bfi_matrix (raw) | NEVER | |
| identity_embedding | NEVER | |

### Class 2 — anonymous

Default for all standard deployments. No identity-correlated fields.

| Field | Published | Notes |
|-------|-----------|-------|
| presence | Yes | |
| motion | Yes | |
| person_count | Yes | |
| identity_risk_score | No | Suppressed |
| rf_signature_hash | No | Suppressed |
| zone_activity | Yes | |
| confidence | Yes | |
| All angle/amplitude/phase fields | No | Zeroed |
| bfi_matrix (raw) | NEVER | |
| identity_embedding | NEVER | |

### Class 3 — restricted

Maximum privacy. Suitable for care facilities, medical deployments, guest spaces.

| Field | Published | Notes |
|-------|-----------|-------|
| presence | Yes | |
| motion | No | Suppressed |
| person_count | No | Suppressed |
| All other fields | No | |
| bfi_matrix (raw) | NEVER | |
| identity_embedding | NEVER | |

---

## 2. rf_signature_hash Rotation Algorithm

### Construction

```
site_salt   := blake3_keyed_hash(secret="bfld-site-seed", data=node_mac_address)
               # Generated once at first boot, stored in NVS, never transmitted
               # 32 bytes

day_epoch   := floor(timestamp_ns / 86_400_000_000_000)
               # One new epoch per UTC day

ephemeral   := mean_angle_delta ‖ subcarrier_variance ‖ burst_motion_score
               # A small fixed-length summary of the current window's features
               # Not identity-specific — any of several persons could produce
               # similar values

rf_signature_hash := BLAKE3(
    key   = site_salt,            // 32 bytes; site-specific secret key
    input = day_epoch_bytes(8) ‖ ephemeral_features(24)
)
```

### Why cross-site re-identification is structurally impossible

Two BFLD nodes at sites A and B produce:

```
hash_A = BLAKE3(key=salt_A, input=day ‖ features)
hash_B = BLAKE3(key=salt_B, input=day ‖ features)
```

BLAKE3 is a PRF (pseudorandom function family) keyed on site_salt. Given identical
`day ‖ features` inputs, hash_A and hash_B are pseudorandom and independent because
salt_A != salt_B. An adversary who observes hash_A and hash_B cannot determine whether
they correspond to the same person without knowing both salts.

This is not a security proof; it is a consequence of BLAKE3's PRF security assumption,
which holds as long as the site_salt remains secret.

### Why within-site, within-day tracking is safe

Within a single day at a single site, two frames from the same person will produce
similar ephemeral features, leading to similar (though not identical — ephemeral features
have some frame-to-frame variation) hash values. This is intentional: it allows
clustering of same-person events within a session without enabling identity recovery.

The hash is NOT the identity. It is a pseudonym within the scope of (site, day). A
person who visits the same site on two different days gets different pseudonyms on each
day.

### Daily rotation schedule

```
epoch_0 = 0                        # day 0 (unix epoch: 1970-01-01)
epoch_k = k * 86_400_000_000_000   # day k in nanoseconds
rotation_time = epoch_{k+1}        # midnight UTC
```

At rotation time, all existing rf_signature_hash values become cryptographically
disconnected from future values. Logs from before rotation cannot be correlated with
logs after rotation even by the node operator.

---

## 3. Identity Embedding Lifecycle

```
BFI frame arrives
      |
      v
Feature extraction (identity_risk.rs)
      |
      v
RuVector embedding computed: Vec<f32, 128>
      |
      +-------> identity_risk_score (scalar projection)
      |         Published (class 1) or suppressed (class 2/3)
      |
      v
In-RAM ring buffer (EmbeddingRingBuf)
      - capacity: 600 frames (default 10 minutes at 1 Hz)
      - implemented as VecDeque<Embedding> in heap memory
      - NEVER written to disk (no serde, no file I/O in the type)
      - NEVER serialized to any MQTT or HTTP path
      - Cleared on node restart (RAM is volatile)
      |
      v [after retention window]
Dropped from ring buffer
```

The ring buffer serves two purposes: (1) temporal_stability calculation requires
comparing the current embedding to recent embeddings; (2) the coherence gate
(`coherence_gate.rs`, from `v2/crates/wifi-densepose-signal/src/ruvsense/`) uses
recent frames to determine whether a new frame is a continuation of an existing
trajectory or a new event.

Both purposes require only that the embeddings exist in RAM during the computation.
Neither purpose requires persistence.

---

## 4. Privacy-Mode Wire-Format Diff

The following shows what changes in the serialized `BfldFrame` payload when the node
transitions from class 1 (derived) to class 2 (anonymous), which is the transition
that happens when `privacy_mode` is enabled by the operator.

```
BfldFrame {
    magic: 0xBF1D_0001,               // unchanged
    version: 1,                        // unchanged
    ap_id: blake3(node_mac ‖ "ap"),   // unchanged (already hashed at ingress)
    sta_id: ephemeral_u64,             // unchanged (already ephemeral)
    session_id: u64,                   // unchanged
    quantization: 0x02,                // unchanged (i8 in class 1)
    privacy_class: 0x01 -> 0x02,       // CHANGED

    // Payload (compressed):
    compressed_angle_matrix: [...],    // class 1: present; class 2: zeroed + omitted
    amplitude_proxy: [...],            // class 1: present; class 2: omitted
    phase_proxy: [...],                // class 1: present; class 2: omitted
    snr_vector: [...],                 // class 1: present; class 2: present (aggregate)

    // Event (JSON within payload or outer envelope):
    presence: true,                    // unchanged
    motion: 0.42,                      // unchanged
    person_count: 1,                   // unchanged
    identity_risk_score: 0.71,         // class 1: present; class 2: OMITTED
    rf_signature_hash: "a3f2...",      // class 1: present; class 2: OMITTED
    zone_activity: "living_room",      // unchanged
    confidence: 0.88,                  // unchanged
    payload_crc32: <recomputed>        // recomputed after changes
}
```

The wire-format diff is verified by the acceptance test suite: the same input must
produce a deterministic output for each privacy_class value.

---

## 5. Default-Deny Posture for Future Fields

Every new field added to `BfldFrame` or the BFLD event JSON in the future MUST be
classified before it ships. The process:

1. New field is added to `BfldFrame` struct.
2. A `#[privacy_class(minimum = N)]` attribute annotation (or equivalent runtime
   check in `privacy_gate.rs`) declares the minimum privacy class at which this
   field is suppressed.
3. Unit test asserts that serializing at class < N includes the field and at class ≥ N
   omits it.
4. The PR that adds the field cannot pass CI without the classification annotation.

This is enforced by a custom `#[must_classify]` lint in the crate — any public field
on `BfldFrame` without a classification attribute produces a compile warning that
becomes a CI error.

---

## 6. Auditability: Verifying That Raw BFI Never Left the Network

An operator who wants to verify that no raw BFI or identity data has been transmitted
from their BFLD node can use the following procedure:

### 6.1 Network-level audit (tcpdump)

```bash
# On the node or a port-mirrored switch:
tcpdump -i eth0 -w bfld_audit.pcap port 1883 or port 8883

# After capture, search for the BFI frame magic bytes in the PCAP:
# Magic 0xBF1D_0001 in big-endian is bytes BF 1D 00 01
# If these bytes appear in the MQTT payload, raw BFI may be present.
# They should NOT appear — BFLD strips the angle matrix at privacy_class >= 2.
strings bfld_audit.pcap | grep -v "presence\|motion\|person_count" | wc -l
# Expected: only presence/motion/person_count keys in the MQTT payloads.
```

### 6.2 Node self-check command

```bash
# RuView CLI (planned for P3):
wifi-densepose bfld audit --duration 60s
# Output: "60 frames processed. 0 frames with raw_bfi in payload.
#          0 frames with identity_embedding in payload.
#          privacy_class distribution: {2: 57, 3: 3}"
```

### 6.3 CI deterministic hash check

```bash
python python/wifi_densepose/verify_bfld.py
# Must print: VERDICT: PASS
# If a modified binary is exfiltrating raw BFI as part of the payload,
# the output hash will differ from the committed expected hash.
```
