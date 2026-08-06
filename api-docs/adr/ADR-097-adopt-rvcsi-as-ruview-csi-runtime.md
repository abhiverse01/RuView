# ADR-097: Adopt rvCSI as RuView's primary CSI runtime

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-13 |
| **Deciders** | ruv |
| **Codename** | **rvCSI-in-RuView** |
| **Relates to** | ADR-095 (rvCSI platform), ADR-096 (rvCSI crate topology / FFI), ADR-014 (SOTA signal processing in `wifi-densepose-signal`), ADR-016 (RuVector training pipeline integration), ADR-024 (AETHER contrastive embeddings), ADR-031 (RuView sensing-first RF mode), ADR-049 (cross-platform WiFi interface detection) |
| **rvCSI repo** | [github.com/ruvnet/rvcsi](https://github.com/ruvnet/rvcsi) (vendored at `vendor/rvcsi`) |

---

## 1. Context

rvCSI — the **edge RF sensing runtime** — was incubated inside RuView under ADR-095 and ADR-096 (PR #542), extracted into its own repo (`ruvnet/rvcsi`, PR #543), and the inline `v2/crates/rvcsi-*` copies were removed in favour of the `vendor/rvcsi` submodule (PR #544). All nine crates are published on crates.io at `0.3.1`; `@ruv/rvcsi 0.3.1` is on npm; a Claude Code plugin marketplace ships with the repo.

> rvCSI normalizes WiFi CSI from many sources (Nexmon, ESP32, Intel, Atheros, file, replay) into one validated `CsiFrame` / `CsiWindow` / `CsiEvent` schema, runs reusable DSP, emits typed confidence-scored events, and bridges to RuVector RF memory. The crate topology — `rvcsi-core` (kernel) → `rvcsi-dsp` / `rvcsi-events` / `rvcsi-adapter-{file,nexmon}` / `rvcsi-ruvector` (leaves) → `rvcsi-runtime` (composition) → `rvcsi-node` (napi-rs) + `rvcsi-cli` — is fixed by ADR-096.

**Today, RuView vendors rvCSI but does not consume it.** No Cargo `Cargo.toml` in `v2/crates/*` depends on any `rvcsi-*` crate; no Rust source `use rvcsi_…`; no `@ruv/rvcsi` import in `ui/`, `dashboard/`, or anywhere else. The submodule (`vendor/rvcsi`) is a pinned reference-only — currently at the initial `0.3.0` commit (not even tracking the latest `0.3.1`).

Meanwhile, RuView's `v2/` workspace carries its own substantial CSI infrastructure that overlaps directly with rvCSI:

| RuView crate (today) | Overlapping rvCSI crate |
|---|---|
| `wifi-densepose-signal` (DSP stages, RuvSense modules) — ADR-014 | `rvcsi-dsp` (DC removal, phase unwrap, Hampel/MAD, smoothing, baseline subtraction, motion-energy/presence) |
| `wifi-densepose-signal::ruvsense::pose_tracker` etc. (per-window aggregates, presence/motion) | `rvcsi-events` (`WindowBuffer`, presence / motion / quality / baseline-drift detectors) |
| `wifi-densepose-hardware` (ESP32 aggregator, TDM, channel hopping) | `rvcsi-adapter-esp32` *(not yet shipped — ADR-095 §1.2 / D15 follow-up)* |
| `wifi-densepose-ruvector` (cross-viewpoint fusion + RuVector v2.0.4 integration) — ADR-016 | `rvcsi-ruvector` (deterministic window/event embeddings, `RfMemoryStore`) |
| `wifi-densepose-sensing-server` (Axum REST + WS) | `rvcsi-node` (napi-rs SDK) + `rvcsi-cli` |

Carrying both indefinitely is a maintenance liability: two diverging code paths for the same concepts, two test surfaces, two bug-fix queues, two API contracts. The extraction of rvCSI was explicitly motivated by giving these primitives a stable, hardware-abstracted home; the natural next step is for RuView to *consume* that home rather than carry parallel implementations.

This ADR decides **how RuView starts depending on rvCSI, where the seams are, and what survives in `v2/crates/wifi-densepose-*`.**

### 1.1 What this ADR is *not*

- Not a rewrite of `wifi-densepose-signal`'s SOTA / RuvSense modules. Those modules go beyond rvCSI's scope (cross-viewpoint fusion, AETHER re-ID, RF tomography, longitudinal biomechanics, adversarial detection) and *stay* in RuView — they consume rvCSI's normalized `CsiFrame` rather than reimplementing the parsing/validation/DSP plumbing below them.
- Not a forced migration of every consumer simultaneously. Adoption is phased.
- Not a decision on whether to delete `archive/v1/` (the Python reference) — that's its own discussion.

---

## 2. Decision

**Adopt rvCSI as the primary CSI ingestion / validation / DSP / event-extraction runtime for RuView, consumed via the published crates.** The decisions below are the architectural contract for that adoption.

### D1 — Depend on the published `rvcsi-*` crates, not the submodule path

Each consuming RuView crate adds `rvcsi-runtime = "0.3"` (or whichever rvCSI crate(s) it needs) to its `Cargo.toml`. Cargo resolves these from crates.io. `vendor/rvcsi` remains a **pinned source-of-truth for local dev / patches / offline builds**, not the build path.
*Consequences:* normal `cargo build` works without `git submodule update --init`; version pinning is explicit in `Cargo.toml`; coordinated upgrades are a single SemVer bump per crate; the submodule pin can lag and that's fine.

### D2 — `wifi-densepose-sensing-server` is the pilot consumer

The sensing-server (Axum REST + WebSocket) is the smallest, best-bounded touchpoint: its UDP CSI receiver and `latest`/`vital-signs`/`edge-vitals` endpoints map cleanly onto `rvcsi-runtime::CaptureRuntime` + the `rvcsi_events` pipeline. The pilot replaces only the **ingestion / validation / DSP / event** path; the existing handlers, the WebSocket fan-out, the RVF model loader, the adaptive classifier and the vital-sign extractor stay.
*Consequences:* one PR-sized adoption to learn from before touching the heavier crates; integration tests in `wifi-densepose-sensing-server` exercise the rvCSI surface against synthetic + real ESP32 captures (the `scripts/esp32_jsonl_to_rvcsi.py` bridge in the standalone repo is the de-facto fixture path).

### D3 — `wifi-densepose-signal` is *layered on top of* rvCSI, not replaced

The RuvSense modules (`multistatic`, `phase_align`, `tomography`, `pose_tracker`, `field_model`, `longitudinal`, `intention`, `cross_room`, `gesture`, `adversarial`, `coherence_gate`) go strictly beyond `rvcsi-dsp` and stay in RuView. They consume `rvcsi_core::CsiFrame` / `CsiWindow` instead of the current `wifi_densepose_core::CsiFrame`-like types.
The genuinely-overlapping primitives in `wifi-densepose-signal` (basic DSP — DC removal, phase unwrap, Hampel, smoothing, baseline subtraction, motion-energy / presence) are either replaced with `rvcsi-dsp::stages::*` calls or kept as thin shims that delegate. A single `From<wifi_densepose_core::CsiFrame> for rvcsi_core::CsiFrame` (and the reverse) lives in `wifi-densepose-signal` during the transition.
*Consequences:* the SOTA work stays in RuView (where it belongs); the parsing/validation/baseline plumbing centralizes in rvCSI; the public API of `wifi-densepose-signal` shifts gradually toward "modules built on top of `rvcsi-*`".

### D4 — `wifi-densepose-hardware` stops carrying ESP32 wire-format parsing

The ESP32 ADR-018 binary frame parsing (magic 0xC5110001, 20-byte header, int8 I/Q — see the `scripts/esp32_jsonl_to_rvcsi.py` bridge in the rvCSI repo) becomes part of a new `rvcsi-adapter-esp32` crate (ADR-095 §1.2 / D15 follow-up, owned in the rvCSI repo). `wifi-densepose-hardware` keeps the firmware/aggregator side (UDP listener, mesh, TDM, channel hopping, NVS provisioning) — i.e. the parts above the wire — and emits parsed `CsiFrame`s via the new adapter trait.
*Consequences:* the firmware-side and host-side concerns split cleanly; the parser lives once (in rvCSI) and is testable in isolation; the wire format is documented once.

### D5 — Embeddings & RF memory: the two `ruvector` paths stay separate (for now)

`wifi-densepose-ruvector` (ADR-016) is the **training** pipeline integration — feeding RuvSense outputs into RuVector for cross-viewpoint fusion, AETHER contrastive embeddings, domain generalization (MERIDIAN). `rvcsi-ruvector` is the **runtime RF-memory** bridge — deterministic per-window/per-event embeddings + `RfMemoryStore`. They serve different jobs; both stay. A follow-up ADR can unify them once `rvcsi-ruvector`'s production backend (currently the `JsonlRfMemory` standin) lands the real RuVector binding.
*Consequences:* no churn in the training pipeline today; the runtime memory and the training-time fusion remain distinct contexts in the DDD sense.

### D6 — Schema: `rvcsi_core::CsiFrame` becomes the boundary type at the runtime edge

At the *runtime* edge (sensing-server, future daemon, any new adapter), `rvcsi_core::CsiFrame` is the validated normalized object. RuView's internal types (`wifi_densepose_core::CsiFrame` and friends) continue to exist for training and SOTA pipelines, but a single explicit conversion happens at the boundary and is the only allowed translation point.
*Consequences:* one validation gate at one edge; downstream code stops re-deriving amplitude/phase / re-checking finiteness; the `validate_frame` quality scoring is the only source of truth for "is this frame usable".

### D7 — Versioning: track rvCSI via SemVer-compatible ranges + pin the submodule

`Cargo.toml` deps use `rvcsi-runtime = "0.3"` etc. (`^0.3`, so 0.3.x picks up automatically). The `vendor/rvcsi` submodule pin is **bumped per RuView release** to whatever rvCSI commit RuView was tested against — providing reproducible offline builds and a source-level reference, even though the actual build resolves from crates.io.
*Consequences:* RuView keeps moving; rvCSI patch releases roll in automatically; minor-version bumps require a deliberate `^0.3` → `^0.4` change (and a re-test of the consumers); the submodule pin advances with each release tag so it never silently drifts.

### D8 — Replace `vendor/rvcsi` with crates.io once D1–D7 are merged

If, after the pilot, every consumer depends on crates.io (no consumer touches `vendor/rvcsi/crates/*`), `vendor/rvcsi` is *redundant*. A future ADR can decide to drop the submodule entirely. Until then it stays.
*Consequences:* the migration path has a clear terminal state; no decision on submodule removal made today.

---

## 3. Adoption phases

| Phase | Scope | Closes |
|---|---|---|
| **P1 (pilot)** — `wifi-densepose-sensing-server` ingestion | UDP receiver + simulated source go through `rvcsi-runtime::CaptureRuntime` + `rvcsi_events::EventPipeline`; sensing-server emits rvCSI events on `/api/v1/events` and the WebSocket. | D1, D2, D6 partly |
| **P2 (signal shim)** — `wifi-densepose-signal` thin-shim adoption | Overlapping DSP primitives delegate to `rvcsi-dsp`; SOTA modules stay; `From`/`Into` bridge added. | D3, D6 |
| **P3 (ESP32 adapter)** — `rvcsi-adapter-esp32` lands in the rvCSI repo; `wifi-densepose-hardware` switches over | New crate in `ruvnet/rvcsi`; RuView consumes it as `rvcsi-adapter-esp32 = "0.3"`. | D4 |
| **P4 (clean-up)** — duplicates removed | Inline DSP primitives in `wifi-densepose-signal` deleted (only shims left for back-compat or fully removed). | D3 fully |
| **P5 (post-pilot)** — `vendor/rvcsi` review | Decide whether to keep the submodule. | D8 |

Each phase is one PR, each PR has unit + integration tests against the rvCSI surface, the workspace test stays green (1,031+ tests).

---

## 4. Consequences

**Positive**

- Single normalized schema (`CsiFrame` / `CsiWindow` / `CsiEvent`) across RuView's runtime surface — fewer bespoke types, less duplication.
- Bad packets quarantined at one place (rvCSI's `validate_frame`), not at every consumer.
- New CSI sources (Intel `iwlwifi`, Atheros, SDR) plug in once at the rvCSI layer, work for every RuView consumer immediately.
- rvCSI's structured `RvcsiError` + the C shim's panic-free contract replace ad-hoc parser error handling in RuView's hardware-side code.
- The sensing-server inherits the FFI-boundary hardening from rvCSI (e.g. the NaN-safe `napi-c` encode fix in `rvcsi-adapter-nexmon 0.3.1` flows in automatically).

**Negative / costs**

- Two repos to keep in lockstep during the adoption (`ruvnet/RuView` + `ruvnet/rvcsi`). Mitigated by SemVer + the per-release submodule bump.
- Per-frame conversion at the boundary in P1/P2 (one `From<rvcsi_core::CsiFrame> for wifi_densepose_core::CsiFrame`-style hop). Cost is a single `Vec` clone of the I/Q + amplitude/phase arrays per frame; at the project's target rates this is well under the 50 ms latency budget.
- The training pipeline (`wifi-densepose-ruvector`) and the runtime RF memory (`rvcsi-ruvector`) coexist until D5's follow-up.
- The Nexmon ESP32 adapter (D4 / P3) is real work in the rvCSI repo before P3 can land.

**Risks**

- API drift between `wifi_densepose_core::CsiFrame` and `rvcsi_core::CsiFrame` if both keep evolving; mitigated by D6 (one explicit conversion point, every other consumer reads only `rvcsi_core::CsiFrame`).
- crates.io as a hard dependency — if crates.io is unreachable in an air-gapped build, `vendor/rvcsi` + `[patch.crates-io]` is the documented escape hatch.

---

## 5. Alternatives considered

| Alternative | Why not |
|---|---|
| Keep both in parallel indefinitely | Two diverging implementations of the same concepts → twice the bug-fix surface, twice the docs, twice the tests; defeats the reason rvCSI was extracted in the first place. |
| Big-bang adoption — replace `wifi-densepose-signal` end-to-end in one PR | Too much surface to land safely; the SOTA modules go *beyond* rvCSI's scope and don't lift cleanly. D3's "layered on top" preserves what matters. |
| Consume `vendor/rvcsi/crates/*` via path deps instead of crates.io | Couples RuView to the submodule's HEAD; loses the SemVer ratchet; makes `cargo build` fail when the submodule isn't initialized. D1 (published crates) is the standard pattern. |
| Move RuView itself into `ruvnet/rvcsi` (monorepo) | Defeats the reason rvCSI was extracted — rvCSI is a runtime usable beyond RuView (other agents, other apps, the standalone CLI + npm SDK). The repo split is intentional. |
| Stay on `wifi-densepose-signal` and treat rvCSI as a sibling library only | Means RuView reimplements every adapter, every validation rule, every event detector forever. D2's pilot validates whether the seams are right before committing to D3. |

---

## 6. Open questions

- **Per-subcarrier calibration baseline.** rvCSI's `events` pipeline benefits from a learned baseline (`SignalPipeline::baseline_amplitude`) — RuView's existing per-node calibration logic (in `wifi-densepose-sensing-server`'s field-model endpoints) should feed that baseline in. The plumbing is straightforward; documenting the format is a P1 sub-task.
- **Single-frame schema overhead.** `rvcsi_core::CsiFrame` carries `i_values + q_values + amplitude + phase + quality_reasons` (four `Vec<f32>` plus a `Vec<String>`). RuView's training pipeline (which sometimes processes 100k+ frames in batch) may want a "lean frame" view to avoid the extra allocations. Track as a separate optimization once P1 is in.
- **Cross-viewpoint fusion outputs as `CsiEvent` metadata.** The `metadata_json: String` field on `CsiEvent` is the natural carrier for RuvSense-derived multistatic fusion outputs; a small `serde` helper in `wifi-densepose-signal` standardizes the JSON shape.

---

## 7. References

- [ADR-095 — rvCSI Edge RF Sensing Platform](ADR-095-rvcsi-edge-rf-sensing-platform.md)
- [ADR-096 — rvCSI Crate Topology, the napi-c Shim, the napi-rs Surface](ADR-096-rvcsi-ffi-crate-layout.md)
- [ADR-014 — SOTA Signal Processing in `wifi-densepose-signal`](ADR-014-sota-signal-processing.md)
- [ADR-016 — RuVector Training Pipeline Integration](ADR-016-ruvector-training-pipeline.md)
- [ADR-031 — RuView Sensing-First RF Mode](ADR-031-ruview-sensing-first-rf-mode.md)
- [`github.com/ruvnet/rvcsi`](https://github.com/ruvnet/rvcsi) — 9 crates on crates.io @ 0.3.1, `@ruv/rvcsi 0.3.1` on npm, Claude Code plugin marketplace
- `vendor/rvcsi` (submodule) — currently pinned at `acd5689d` (0.3.0 commit); bumps to `0.3.1` HEAD as part of P1
