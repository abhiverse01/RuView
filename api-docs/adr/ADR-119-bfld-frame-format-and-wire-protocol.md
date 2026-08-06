# ADR-119: BFLD Frame Format and Wire Protocol

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-24 |
| **Deciders** | ruv |
| **Parent** | [ADR-118](ADR-118-bfld-beamforming-feedback-layer-for-detection.md) |
| **Relates to** | [ADR-028](ADR-028-esp32-capability-audit.md) (witness/deterministic proof), [ADR-095](ADR-095-rvcsi-edge-rf-sensing-platform.md) (rvCSI `CsiFrame` schema) |
| **Tracking issue** | TBD |

---

## 1. Context

The BFLD pipeline (ADR-118) emits an over-the-wire `BfldFrame` consumed by the RuView aggregator, HA bridge, and witness bundle. The frame must be:

1. **Deterministic** — identical input ⇒ bit-identical output, so witness hashes survive verification (ADR-028 pattern).
2. **Self-describing** — magic + version so future BFLD revisions don't silently corrupt aggregator state.
3. **Privacy-classified at the byte level** — the receiver must know the data class before it even parses the payload, so it can drop frames it isn't authorized to handle.
4. **Compact** — BFLD nodes may emit at up to 10 Hz; the frame must be small enough for unsharded MQTT and ESP-NOW transport.
5. **Endianness-stable** — captures from x86_64 (ruvultra), aarch64 (cognitum-v0, Pi 5 cluster), and Xtensa (ESP32-S3) must produce identical bytes.

The existing rvCSI `CsiFrame` (ADR-095) is the closest precedent. BFLD reuses the same little-endian convention and the same "validate-before-FFI" posture.

---

## 2. Decision

### 2.1 `BfldFrame` header (40 bytes, little-endian, packed)

```rust
#[repr(C, packed)]
pub struct BfldFrameHeader {
    pub magic: u32,              // 0xBF1D_0001
    pub version: u16,            // 1
    pub flags: u16,              // bit0=has_csi_delta, bit1=privacy_mode, bit2-15 reserved
    pub timestamp_ns: u64,       // monotonic capture clock

    pub ap_hash: [u8; 16],       // BLAKE3-keyed(site_salt, ap_mac)[0..16]
    pub sta_hash: [u8; 16],      // BLAKE3-keyed(site_salt ‖ day_epoch, sta_mac)[0..16]
    pub session_id: [u8; 16],    // ephemeral, rotated on capture-session boundary

    pub channel: u16,            // 802.11 channel number
    pub bandwidth_mhz: u16,      // 20 | 40 | 80 | 160
    pub rssi_dbm: i16,
    pub noise_floor_dbm: i16,

    pub n_subcarriers: u16,
    pub n_tx: u8,
    pub n_rx: u8,
    pub quantization: u8,        // 0=f32, 1=i16, 2=i8, 3=packed (4-bit nibbles)
    pub privacy_class: u8,       // 0=raw, 1=derived, 2=anonymous, 3=restricted (default 2)

    pub payload_len: u32,
    pub payload_crc32: u32,      // CRC-32/ISO-HDLC over payload bytes only
}
```

Total header size: **86 bytes packed** (validated by `static_assertions::const_assert_eq!` in `wifi-densepose-bfld/src/frame.rs`). Earlier drafts stated 40 bytes — that was a counting error caught during P1 scaffold; see AC1 below.

### 2.2 Payload structure

Payload is a length-prefixed sequence of typed sections in this exact order:

```
payload = compressed_angle_matrix
        ‖ amplitude_proxy
        ‖ phase_proxy
        ‖ snr_vector
        ‖ optional_csi_delta            (present iff flags.bit0 set)
        ‖ optional_vendor_extension     (length 0 allowed)
```

Each section is `[u32 len_le][bytes...]`. The CRC32 covers all section bytes including length prefixes, but **not** the header.

### 2.3 Privacy-class gating at serialization

The serializer enforces these rules **before** writing any payload bytes:

| `privacy_class` | `compressed_angle_matrix` | Identity-derived fields | Notes |
|-----------------|---------------------------|-------------------------|-------|
| 0 (`raw`) | full | full | **Local-only**, never serialized to a network sink |
| 1 (`derived`) | downsampled to 8-bit, top-k subcarriers | full | Operator-acknowledged research mode |
| 2 (`anonymous`, **default**) | absent (zero-length section) | absent | Production default |
| 3 (`restricted`) | absent | absent + diagnostic-only | Equivalent to class 2 + suppresses `identity_risk_score` on the bus |

The serializer returns `Err(BfldError::PrivacyViolation)` if the caller attempts to publish a class-0 frame through a network sink. This is enforced by a sink-type marker trait (`LocalSink` vs `NetworkSink`).

### 2.4 Deterministic serialization

Three guarantees:

1. **Field order is fixed** by `#[repr(C, packed)]`.
2. **Float quantization is canonical** — `quantization` byte values 1/2/3 use specified round-half-to-even with documented saturation; f32 (value 0) is forbidden over the wire (local-only).
3. **CRC32 is computed last**, after all section bytes are placed.

The witness test in `tests/determinism.rs` captures a 200-frame BFI fixture, serializes it 1,000 times across two threads, and verifies the BLAKE3 of the resulting byte stream is bit-identical.

### 2.5 Magic value rationale

`0xBF1D_0001` is chosen so that `bf1d` reads as "BFLD" in hex-dump output, easing wireshark / xxd debugging. The final `0001` is the major version; minor revisions bump `version` field.

---

## 3. Consequences

### Positive

- 40-byte header + compact payload fits comfortably in a 1500-byte MTU even at 4×4 MIMO with 256 subcarriers.
- Serialization is `#[no_std]` compatible — same code can run on ESP32-S3 (when ESP-NOW transport is added under ADR-123 P2).
- Witness-bundle integration is direct: the existing `archive/v1/data/proof/verify.py` pattern extends to a `bfld_verify.py` that consumes the same SHA-256 expected-hash file format.

### Negative

- `#[repr(C, packed)]` on the header means consumers must use `read_unaligned` — small ergonomic cost, mitigated by a `#[derive(BfldFrameAccess)]` proc-macro.
- Reserved flag bits 2-15 lock in future-extension order; any new bit assignment is a version bump.

### Neutral

- The vendor-extension section allows downstream RuView cogs (e.g., `cog-pose-estimation`) to attach metadata without a header change, at the cost of CRC scope creep. Vendor sections are explicitly outside the witness hash.

---

## 4. Alternatives Considered

### Alt 1: Protobuf / FlatBuffers

Rejected: schema evolution overhead, witness-hash instability across protoc versions, ~3× wire bloat for the small fixed-shape fields.

### Alt 2: CBOR

Rejected: deterministic CBOR (RFC 8949 §4.2) is achievable but the parser surface is large and tag handling is a footgun for the `no_std` ESP32 path.

### Alt 3: Variable-width magic / no magic

Rejected: receivers must distinguish BFLD frames from rvCSI `CsiFrame` and other RuView payloads on shared transports.

### Alt 4: Move CRC32 to header

Rejected: CRC must be computed after the payload, so its value would otherwise force a header rewrite; placing it last avoids a buffer-pass-back.

---

## 5. Acceptance Criteria

- [ ] **AC1**: `BfldFrameHeader` size is exactly **86 bytes** (packed) on x86_64, aarch64, and xtensa-esp32s3. The size was initially documented as 40 bytes during ADR drafting — that was a counting error; the implementation in `wifi-densepose-bfld/src/frame.rs` enforces the correct value via `const_assert_eq!`.
- [ ] **AC2**: 1,000 serializations of a fixed `BfiCapture` fixture produce a bit-identical BLAKE3 hash.
- [ ] **AC3**: `privacy_class = 0` frame returned through `NetworkSink::publish()` returns `Err(BfldError::PrivacyViolation)`.
- [ ] **AC4**: Payload CRC32 mismatch causes `BfldFrame::parse()` to return `Err(BfldError::Crc)` without exposing partial payload state.
- [ ] **AC5**: Round-trip serialize/parse preserves all header fields exactly.
- [ ] **AC6**: A frame with `flags.bit0 = 0` (no CSI delta) and an unexpected CSI-delta section is rejected.
- [ ] **AC7**: Bench: serialization throughput ≥ 50k frames/sec on a 2025-era M1/M2 / Pi 5 core.

---

## 6. References

- ADR-118 §2 (umbrella decision)
- ADR-095 `CsiFrame` (`vendor/rvcsi/crates/rvcsi-core/src/frame.rs`)
- CRC-32/ISO-HDLC: `crc = "3"` crate
- BLAKE3 keyed mode: `blake3 = "1.5"`
- IEEE 802.11-2020 §19.3.12 (Compressed Beamforming Report)
