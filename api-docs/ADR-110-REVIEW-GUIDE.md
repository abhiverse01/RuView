# ADR-110 review guide

This is the **one-pager** for reviewers of the `adr-110-esp32c6` branch / draft PR. The canonical record is [`docs/WITNESS-LOG-110.md`](WITNESS-LOG-110.md); this guide is just a faster on-ramp.

## What this branch ships

A dual-target build for `firmware/esp32-csi-node`: same source tree compiles for `esp32s3` (existing production) and `esp32c6` (new research target with Wi-Fi 6 / 802.15.4 / TWT / LP-core). Every C6-only module is `#ifdef CONFIG_IDF_TARGET_ESP32C6` gated, so the S3 build path is byte-identical to before.

## Five-minute reviewer tour

1. **Read the ADR**: [`docs/adr/ADR-110-esp32-c6-firmware-extension.md`](adr/ADR-110-esp32-c6-firmware-extension.md) — design, phases, trade-offs.
2. **Read the witness**: [`docs/WITNESS-LOG-110.md`](WITNESS-LOG-110.md) — 4 sections (A = empirically verified, B = architectural-but-not-measured, C = bugs fixed, D = bugs found but not yet fixed, D-workaround = ESP-NOW pivot).
3. **Skim the new firmware modules**: `firmware/esp32-csi-node/main/c6_{twt,timesync,lp_core,sync_espnow}.{h,c}`.
4. **Skim the new host decoders + tests**:
   - Rust: `v2/crates/wifi-densepose-hardware/src/{csi_frame,esp32_parser}.rs` (search for `PpduType`, `Adr018Flags`, `adr110_*` test names)
   - Python: `archive/v1/src/hardware/csi_extractor.py` + `archive/v1/tests/unit/test_esp32_binary_parser.py` (search for `TestAdr110ByteEncoding`)
5. **Glance at CI**: `firmware-ci.yml` `c6-4mb` matrix row runs the C6 build AND the host unit tests on Ubuntu — both green throughout this branch.

## Empirical scorecard (what's actually measured)

| Dimension | Status |
|---|---|
| C6 build + boot + dual-target | ✅ verified on 3 boards (COM6/COM9/COM12), CI matrix green, S3 regression green |
| HE-LTF wire format (ADR-018 byte 18-19) | ✅ verified end-to-end across firmware / Rust / Python (17 unit tests) |
| HE-LTF live capture | ⏸ blocked — need 11ax AP (only 11n AP on bench) |
| TWT graceful NACK | ✅ verified live — `c6_twt: iTWT setup failed: ESP_ERR_INVALID_ARG` captured + handled |
| TWT cadence determinism | ⏸ blocked — same 11ax AP gap |
| ESP-NOW transport TX + stability | ✅ verified — 120 s + 300 s soaks, 4102 cumulative transmits, 0 failures |
| ESP-NOW cross-board RX | ⏸ blocked — 3 of 4 boards dropped USB enumeration mid-experiment |
| Raw 802.15.4 cross-node sync | ❌ broken — IDF v5.4 driver bug, 5 hypotheses tested + rejected; ESP-NOW workaround in place |
| 5 µA hibernation | ⏸ blocked — datasheet number, need INA / Joulescope to measure |
| Witness bundle regenerable + clean | ✅ 6/7 PASS (1 fail is pre-existing Python proof env issue unrelated to ADR-110), all hashes recorded, secret-redacted |

## Honest verdict

Protocol layer + transport substrate are bullet-proofed. **None of the four headline SOTA dimensions is empirically measured** — each is blocked on hardware the bench doesn't have. Each blocker is documented in `WITNESS-LOG-110.md` §B with the exact instrument needed to unblock it. **This branch is the foundation to build measurement on, not the measurement itself.**

The five concrete bugs found and fixed during the work (MAC/EUI double-FFFE, dual `wifi_pkt_rx_ctrl_t` struct variants, LED GPIO 38 on C6, TWT INVALID_ARG propagation, witness bundle secret leak) are independently real and useful regardless of how the SOTA story lands.

## Security note for the operator (not the reviewer)

The witness bundle's Python proof step was leaking `.env` contents into the bundled log via Pydantic validation error dumps. Bundle was nuked before push, and `scripts/redact-secrets.py` filter was added (commit `f8a2e3695`). **The previously-exposed Docker Hub + PI-cluster tokens should be rotated** — they appeared in local session logs even though they never reached `origin`.

## Commits on this branch (chronological)

| # | SHA prefix | What |
|---|---|---|
| 1 | `f23e34e` | Initial ADR-110 firmware + ADR + tests + docs + witness scaffolding |
| 2 | `6652384` | TWT INVALID_ARG graceful + diagnostic counters |
| 3 | `4c39e28` | PAN-match + 4-experiment D1 record |
| 4 | `f8a2e36` | **SECURITY**: witness bundle secret redaction |
| 5 | `88be283` | ESP-NOW transport (D1 workaround) |
| 6 | `3959fab` | Rust host decoder + 6 unit tests |
| 7 | `8eaa92c` | Python host decoder + 5 unit tests |
| 8 | `b808a63` | 120 s ESP-NOW soak witness |
| 9 | `89972c0` | CHANGELOG expanded |
| 10 | `fc75a8a` | Fuzz harness extended for byte 18-19 |
| 11 | `9de34ba` | ADR-110 indexed in docs/adr/README.md |
| 12 | `553b07d` | README C6 row tightened (claim → wire-format-ready) |
| 13 | `e255b7d` | firmware/README acknowledges S3+C6 |
| 14 | `9a46fc8` | 300 s ESP-NOW soak witness (2.5× sample) |
| 15 | _(this commit)_ | This review guide |
