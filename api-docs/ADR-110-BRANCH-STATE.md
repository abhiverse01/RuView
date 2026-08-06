# ADR-110 — Branch state (as of 2026-05-23, iter 22)

Reference card for anyone collaborating on or near the ADR-110 work. The /loop SOTA sprint that closed the firmware-side substrate ran into multiple cross-branch checkout incidents (see iter 17-19); this page exists so the next collaborator doesn't have to re-derive the layout from `git log`.

## Branch ownership

| Branch | Owner | What it carries | Don't merge from |
|---|---|---|---|
| `main` | shared | shipped release line | — |
| `adr-110-esp32c6` | ADR-110 / C6 firmware substrate | Everything described in `WITNESS-LOG-110 §A0.x` (4 firmware tags v0.6.7 → v0.7.0, Python + Rust decoders, sensing-server wire, mesh-aligned timestamp recovery, fps EMA, cross-language conformance gate) | Don't accidentally land `feat/adr-115-ha-mqtt-matter` work here uncommitted |
| `feat/adr-115-ha-mqtt-matter` | ADR-115 / HA-DISCO + HA-FABRIC + HA-MIND | MQTT publisher (`rumqttc`), Matter Bridge, semantic automation primitives, related Cargo features + CLI flags | Don't accidentally land ADR-110 `wifi-densepose-hardware` dep mods here |

## Files each branch touches

### `adr-110-esp32c6` — primary modifications

```
firmware/esp32-csi-node/version.txt                              # bumped 0.6.6 → 0.7.0
firmware/esp32-csi-node/main/c6_*.{c,h}                          # LP-core, TWT, timesync, soft-AP HE, ESP-NOW sync
firmware/esp32-csi-node/main/lp_core/main.c                      # real LP-core polling program
firmware/esp32-csi-node/main/csi_collector.c                     # byte 19 bit 4 OR-fix; sync packet emit
firmware/esp32-csi-node/main/Kconfig.projbuild                   # C6_* knobs
firmware/esp32-csi-node/main/CMakeLists.txt                      # ulp_embed_binary
firmware/esp32-csi-node/sdkconfig.defaults.esp32c6               # C6 overlay

archive/v1/src/hardware/csi_extractor.py                         # SyncPacketParser + SyncPacket dataclass
archive/v1/tests/unit/test_esp32_binary_parser.py                # TestSyncPacketParser (7 tests)

v2/crates/wifi-densepose-hardware/src/sync_packet.rs             # new module (15 tests)
v2/crates/wifi-densepose-hardware/src/lib.rs                     # re-exports
v2/crates/wifi-densepose-sensing-server/Cargo.toml               # ONLY adds wifi-densepose-hardware path dep
v2/crates/wifi-densepose-sensing-server/src/main.rs              # NodeState::{latest_sync, csi_fps_ema,
                                                                 #            mesh_aligned_us_for_csi_frame,
                                                                 #            observe_csi_frame_arrival}
                                                                 # udp_receiver_task magic dispatch
                                                                 # fps_ema_tests module (4 tests)

docs/adr/ADR-110-esp32-c6-firmware-extension.md                  # 670 → ~750 lines (P10 + sprint summary)
docs/WITNESS-LOG-110.md                                          # 13 §A0.x entries
docs/ADR-110-REVIEW-GUIDE.md                                     # reviewer one-pager
docs/ADR-110-BRANCH-STATE.md                                     # ← this file
```

### `feat/adr-115-ha-mqtt-matter` — primary modifications

```
docs/adr/ADR-115-home-assistant-integration.md                   # the design
v2/crates/wifi-densepose-sensing-server/Cargo.toml               # rumqttc dep + [features] block
v2/crates/wifi-densepose-sensing-server/src/cli.rs               # --mqtt / --matter / --semantic flags
```

## Known overlap points (handle with care)

Both branches touch `v2/crates/wifi-densepose-sensing-server/Cargo.toml` and `src/main.rs`. The conflict surface is **disjoint by section**:

| File | ADR-110 region | ADR-115 region |
|---|---|---|
| `Cargo.toml` | `[dependencies]` — `wifi-densepose-hardware = { path = "../wifi-densepose-hardware" }` near the existing `wifi-densepose-signal` line | `[dependencies]` — `rumqttc` block below + `[features]` block at end |
| `main.rs` | `NodeState` fields + `impl NodeState` helpers + `update_csi_fps_ema` free fn + `fps_ema_tests` module + `udp_receiver_task` magic dispatch | (TBD per ADR-115 P-plan) |

A merge between the two branches should be **clean line-merge** since the regions don't overlap. If git ever reports a real conflict in either of these files, that means one branch has drifted into the other's region — investigate before resolving blindly.

## Quick test commands (verify either branch is sane)

```bash
# Rust workspace (run from v2/)
cd v2
cargo test --workspace --no-default-features --lib       # 1437 tests at iter 22, 0 failures

# Python ADR-110 host decoder (from repo root)
python -m pytest archive/v1/tests/unit/test_esp32_binary_parser.py::TestSyncPacketParser -v

# Cross-language wire-format gate (the iter 21 pin)
cargo test -p wifi-densepose-hardware --no-default-features --lib sync_packet::tests::canonical_wire_bytes_match_python_decoder
python -m pytest archive/v1/tests/unit/test_esp32_binary_parser.py::TestSyncPacketParser::test_canonical_wire_bytes_match_rust_decoder -v
```

If either side of the canonical-wire-bytes pair fails alone, the OTHER decoder has drifted from the wire format — investigate that decoder first, not the failing test.

## Future-proofing

- When the ADR-115 agent ships `feat/adr-115-ha-mqtt-matter` to main and ADR-110 also ships, merge `main` into `adr-110-esp32c6` (or vice versa) and re-run both test suites. The disjoint-region structure above should make the merge a no-conflict fast-forward.
- When a third agent picks up either ADR, point them at this file before they start editing shared files.
- If a /loop drives autonomous iterations and hits a cross-branch checkout, the recovery procedure is in iter 18's commit message (`2997165bc`) — stash on the foreign branch, `git checkout` home, replay the iter locally.

## Lessons for `/loop` and `/loop-worker` future runs

Captured after the 38-iter ADR-110 SOTA sprint (`/loop 5m until sota. and ultra optmized`):

1. **Always verify the current branch at the start of each iter** — when a /loop fires every 5 minutes and another agent is active on a sibling branch, the working tree can flip without your action. Run `git branch --show-current` as the first line of every iter; if it isn't what you expect, stash and switch back BEFORE editing. We burned ~30 min in iter 17-19 recovering from two silent branch flips.
2. **Don't `git add <file>` blindly after a branch switch** — the file may have inherited changes from the foreign branch (uncommitted work that came along on checkout). Always `git diff --cached` before `git commit`. We accidentally absorbed ADR-115's Cargo.toml/cli.rs work into ADR-110's iter-18 commit; required a follow-up revert commit (`ca2059b07`) and stash dance.
3. **Sibling-region edits in shared files** — when two branches both touch `v2/crates/wifi-densepose-sensing-server/Cargo.toml` or `src/main.rs`, agree on which `[section]` or struct each owns. Document the regions in this file (see Known overlap points). Merges then stay clean line-merge fast-forwards instead of needing conflict resolution.
4. **Extract pure helpers before committing inline mutations** — iter 30 (`sync_snapshot`), iter 32 (`apply_sync_packet`), iter 37 (`fleet_role_counts`) all converted inline state-changes into named, free, testable functions. Each saved 4+ inline duplications and let the helper be tested without spinning up axum / tokio. Bake this into every iter's plan: *"what's the smallest helper I can extract here?"*
5. **Cross-language wire-format gates** — when shipping a protocol decoder in both Python and Rust, pin the SAME canonical byte string in BOTH test suites (iter 21 pattern). One side drifting fires exactly one named test on exactly the drifted decoder. Don't wait until "later" — add the pin in the iter that ships the second language.
6. **Helper tests > integration tests when state is heavy** — `AppStateInner` has too many fields to construct in a test. Instead of fighting it, extract per-field logic into pure helpers (iter 30 sync_snapshot pattern). Tests target the helpers, the handler glue stays thin and trivially correct.
7. **Local stub files lag firmware additions** — `firmware/esp32-csi-node/test/stubs/esp_stubs.c` doesn't get rebuilt with the firmware proper, so a new symbol added to a `*.h` won't surface as a fuzz-target link error until CI runs. Iter 38 caught `c6_sync_espnow_is_valid` this way. **Whenever you add a function whose declaration is reachable from `csi_collector.c`, also add a stub** in the same commit.
8. **Cron-based /loop accumulates work across irreversible checkpoints (tags, releases, PR ready)** — once you cut a tag or mark a PR ready, the cost of reverting is much higher than a code edit. Save those for iters when you have surplus confidence (full local test suite green, CI from previous iter green). Iter 12 (v0.7.0 cut) and iter 38 (PR ready) were the right shape: only happened after iter 6 / iter 37 evidence had landed.
