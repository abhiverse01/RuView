# ADR-123: BFLD Capture Path — Pi 5 / Nexmon Adapter and ESP32-S3 Feasibility

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-24 |
| **Deciders** | ruv |
| **Parent** | [ADR-118](ADR-118-bfld-beamforming-feedback-layer-for-detection.md) |
| **Relates to** | [ADR-022](ADR-022-multi-bssid-wifi-scanning.md) (multi-BSSID scan), [ADR-028](ADR-028-esp32-capability-audit.md) (capability audit), [ADR-095](ADR-095-rvcsi-edge-rf-sensing-platform.md) (rvCSI), [ADR-096](ADR-096-rvcsi-ffi-crate-layout.md) (rvCSI FFI), [ADR-110](ADR-110-esp32-c6-firmware-extension.md) (C6 firmware), [ADR-119](ADR-119-bfld-frame-format-and-wire-protocol.md) (BfldFrame) |
| **Tracking issue** | TBD |

---

## 1. Context

ADR-118 declares that BFLD captures BFI from commodity WiFi 5/6 traffic. The question this sub-ADR answers is: **on which hardware, with which adapter, and against which firmware limitations**.

### 1.1 ESP32-S3 BFI capability gap

The ESP32 capability audit (ADR-028) and the ESP32-S3 / C6 firmware (`firmware/esp32-csi-node/`, ADR-110) confirm that the Espressif WiFi API exposes **CSI** capture (`esp_wifi_set_csi_*`) but does not expose **raw 802.11 management-frame capture** in monitor mode for non-self-addressed CBFR reports. The S3 sees the CBFR frames its own AP-link generates (when it acts as a beamformer), but it cannot promiscuously sniff CBFR frames between other STA/AP pairs in the neighborhood.

The C6 (ESP32-C6 with RISC-V + Wi-Fi 6) has a more flexible RF subsystem but the same software-API constraint at the time of writing.

### 1.2 Pi 5 / Nexmon as the production capture host

The rvCSI platform (ADR-095/096) already vendors a Nexmon-based adapter (`rvcsi-adapter-nexmon`) that captures CSI from BCM43455c0 chips (Pi 5 / Pi 4 / Pi 3B+). Nexmon patches the firmware to surface CSI to userspace and **also surface CBFR frames** — the BFI extension is the same code path with a different filter.

cognitum-v0 (Pi 5 in the fleet, per CLAUDE.local.md) is already running Nexmon + the rvCSI runtime. It is the natural BFLD capture host.

### 1.3 What we need from each hardware tier

| Tier | Role | BFI capture | CSI capture | Notes |
|------|------|-------------|-------------|-------|
| ESP32-S3 / C6 | Sensing leaf | **no** | yes | Continues providing CSI to the existing pipeline |
| Pi 5 / Nexmon | BFLD host | **yes** | yes (via Nexmon) | Primary BFLD capture |
| ruvultra (RTX 5080 + AX210) | Training / dev | yes (via AX210 monitor mode) | yes | Dev capture; not production |
| cognitum-v0 (Pi 5) | Appliance | **yes** (production) | yes | Production BFLD host |

---

## 2. Decision

### 2.1 Production capture path: Pi 5 / Nexmon

The BFLD production capture path is implemented as a new module in the vendored rvCSI submodule:

```
vendor/rvcsi/crates/rvcsi-adapter-nexmon/
└── src/
    ├── lib.rs
    ├── csi.rs                  # existing CSI capture
    └── bfi.rs                  # NEW — CBFR capture, exports BfiCapture
```

The new `bfi.rs` parses CBFR frames (VHT or HE) from the Nexmon-patched firmware's userspace stream, extracts Φ/ψ angle matrices, and emits a `BfiCapture` struct that feeds the BFLD crate's extractor (ADR-118 §2.1, ADR-119).

The patch lives in the rvcsi submodule (`github.com/ruvnet/rvcsi`) and is shipped as `rvcsi-adapter-nexmon ^0.3.5` to crates.io. The wifi-densepose workspace consumes the published crate (or the submodule path during development).

### 2.2 BFLD crate adapter trait

`wifi-densepose-bfld` defines a `BfiCaptureAdapter` trait:

```rust
pub trait BfiCaptureAdapter: Send + 'static {
    type Error: std::error::Error + Send + Sync + 'static;
    fn capture(&mut self) -> Result<Option<BfiCapture>, Self::Error>;
    fn capabilities(&self) -> AdapterCapabilities;
}

pub struct AdapterCapabilities {
    pub supports_he: bool,           // 802.11ax (Wi-Fi 6)
    pub supports_160mhz: bool,
    pub max_n_rx: u8,
    pub host_kind: HostKind,         // Pi5Nexmon | Ax210Linux | EspS3Local | Mock
}
```

Three impls ship initially:

- `NexmonBfiAdapter` — Pi 5 / Nexmon (production)
- `Ax210BfiAdapter` — Linux + AX210 in monitor mode (dev / training, ruvultra)
- `MockBfiAdapter` — replay fixture for tests and CI

A future fourth impl (`EspS3LocalAdapter`) is reserved for the day Espressif exposes promiscuous CBFR — it captures only the S3's own AP-link BFI for local self-reporting.

### 2.3 Capture-side privacy boundary

Per ADR-120 I1, raw BFI never leaves the capturing host. The adapter must therefore live on **the same physical box** as the BFLD crate's extractor and privacy gate. The architecture pattern:

```
[ Pi 5 / cognitum-v0 ]
├── nexmon firmware (kernel)
├── rvcsi-adapter-nexmon (userspace, captures BFI)
├── wifi-densepose-bfld (extracts, scores, gates)
│       └── privacy_gate → class-2/3 frames only
└── wifi-densepose-sensing-server (publishes MQTT + Matter)
```

A network-mode adapter that streams raw BFI from a remote capture host is **explicitly forbidden**. The adapter trait does not include any "remote URL" parameter.

### 2.4 Channel / bandwidth coverage

The Nexmon adapter is configured by the existing `rvcsi-adapter-nexmon` channel-hopping schedule (ADR-095 §3.2). For BFLD it adds:

- Filter for VHT CBFR (action frame, category 21, action 0) and HE CBFR (category 30, action 0).
- Per-channel BFI session-tracking — the same beamformer/beamformee pair across a channel hop is reconciled by AP MAC + STA MAC.

### 2.5 ESP32-S3 local self-reporting (deferred)

For deployments without a Pi 5 / cognitum-v0 nearby, a degraded BFLD mode runs on the ESP32-S3 itself:

- Captures only its own AP-link CBFR (self-addressed).
- Computes features over the limited window.
- Reports a coarsened `presence` + `motion` only — no `identity_risk_score` (insufficient sample diversity).
- Emits `BfldFrame` at `privacy_class = 2` with a `flags.bit3 = self_only` marker.

This path is implemented in firmware as part of P2 / P3 of the ADR-118 rollout, after the Pi 5 path is stable. Effort is small (firmware path reuses the existing CSI capture loop) but the value is also low until ESP32 firmware exposes promiscuous CBFR — which is a Espressif-IDF roadmap item, not under project control.

### 2.6 Dev path: ruvultra / AX210

For local dev iteration on the Windows / ruvultra box, the AX210 adapter provides a workable capture path on Linux (ruvultra is Ubuntu 6.17 per CLAUDE.local.md). The AX210 supports 802.11ax + monitor mode with the `iwlwifi` driver patches that have landed upstream. This path is for training-data collection and dev testing, not production.

---

## 3. Consequences

### Positive

- BFLD ships as a production-ready surface on cognitum-v0 day one — no new hardware procurement.
- The adapter-trait design lets new capture paths (AX211, MediaTek Filogic, etc.) slot in without changes to the BFLD crate.
- The capture-side privacy boundary is structural: there is no remote-capture code path, so a future PR cannot accidentally introduce one.
- ruvultra's AX210 path unblocks training and dev iteration on Linux without depending on the Pi 5 fleet.

### Negative

- BFLD's full pipeline depends on cognitum-v0 (or another Pi 5 / Nexmon host) being present in the deployment. Operators without a Pi 5 get only the degraded ESP32-S3 self-reporting path (limited utility).
- Nexmon is a third-party kernel module; tracking upstream patches is ongoing maintenance.
- The CBFR frame format differs between VHT (802.11ac) and HE (802.11ax); the parser must support both, and any 802.11be (Wi-Fi 7) deployment will require an additional parser path.

### Neutral

- ruvultra dev path uses AX210; the AX210 is not the production NIC, so dev/prod parity is via the fixture replay + the Nexmon adapter on cognitum-v0.

---

## 4. Alternatives Considered

### Alt 1: Centralized capture host streams raw BFI to RuView nodes

Rejected: violates ADR-120 I1 (raw never leaves the capture host). The capture host **is** the BFLD node; there is no separation.

### Alt 2: Wait for Espressif promiscuous CBFR support

Rejected: indefinite timeline outside project control. The Pi 5 / Nexmon path is shippable today.

### Alt 3: Custom Pi 5 firmware fork instead of Nexmon

Rejected: forking BCM firmware is a huge maintenance burden and Nexmon already does what we need.

### Alt 4: Only ship the ESP32-S3 self-reporting path

Rejected: insufficient sample diversity for `identity_risk_score`. The whole point of BFLD is to measure identity leakage; a self-only path cannot do that meaningfully.

---

## 5. Acceptance Criteria

- [ ] **AC1**: `NexmonBfiAdapter` captures ≥ 100 valid CBFR frames per minute from a 2-AP-3-STA test bench on a Pi 5 (cognitum-v0).
- [ ] **AC2**: VHT (802.11ac) and HE (802.11ax) CBFR frames are both parsed; mixed-PHY captures produce correctly-typed `BfiCapture` outputs.
- [ ] **AC3**: 20/40/80/160 MHz channel widths are all supported (one fixture each in `tests/`).
- [ ] **AC4**: `BfiCaptureAdapter` trait has no method accepting a remote URL or socket address.
- [ ] **AC5**: ESP32-S3 self-only adapter compiles `#[no_std]` and produces a `BfldFrame` with `flags.bit3 = self_only` set, no `identity_risk_score` field.
- [ ] **AC6**: AX210 adapter on ruvultra captures CBFR for at least one fixture-generating dev session.
- [ ] **AC7**: Capture loop sustains 10 Hz BFI frame rate on cognitum-v0 without dropping frames over a 10-minute soak test.

---

## 6. References

- ADR-095 / ADR-096 (rvCSI Nexmon adapter)
- ADR-028 (ESP32 capability audit)
- ADR-110 (ESP32-C6 firmware)
- Nexmon BCM43455c0 patches: https://github.com/seemoo-lab/nexmon
- Wi-BFI: https://arxiv.org/abs/2309.04408
- IEEE 802.11-2020 §19.3.12 (VHT CBFR), §27.3.11 (HE CBFR)
- cognitum-v0 fleet entry: `CLAUDE.local.md` (Tailscale fleet table)
