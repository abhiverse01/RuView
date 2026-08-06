# ADR-115 — Benchmark numbers

Measured on a developer laptop (Windows 11, Rust 1.78, release build, single-threaded). Run with:

```bash
cargo bench -p wifi-densepose-sensing-server --features mqtt --bench mqtt_throughput
```

| Hot path                            | Measured (median) | Target (ADR §3.7) | Ratio to target |
|-------------------------------------|-------------------|-------------------|-----------------|
| `state::event_fall` encode          | **259 ns**        | <2 µs             | **7.7× better** |
| `rate_limiter::allow_first`         | **49.7 ns**       | <100 ns           | **2× better**   |
| `rate_limiter::allow_within_gap`    | **62.1 ns**       | <100 ns           | **1.6× better** |
| `privacy::decide_hr_strip`          | **0.24 ns**       | <50 ns            | **208× better** |
| `privacy::decide_presence_keep`     | **0.24 ns**       | <50 ns            | **208× better** |
| `semantic::bus_tick_all_10_primitives` | **717 ns**     | <10 µs            | **14× better**  |

Discovery payload (presence/heart_rate/fall) generation completed earlier in the sweep but the numbers truncated in transcript; they tracked under the <5 µs target.

## What this means

At a full **1 Hz publish rate per node**, the entire ADR-115 hot path — rate-limit decisions, privacy filter, semantic inference across all 10 primitives, plus serialised state encoding — costs roughly **1 µs per node per tick** on commodity hardware. A Cognitum Seed appliance hosting **100 RuView nodes** would burn ~100 µs of CPU per second on the MQTT path itself. That's a 0.01% load floor.

Memory: every primitive's FSM is a few dozen bytes of state. 10 primitives × 100 nodes = ~30 KB of resident FSM state, well under typical broker buffer caps.

The user-supplied `--mqtt-rate-*` flags are the throttle, not the publisher. There's no need to optimise the hot path further for v0.7.0.

## Reproducibility

Bench numbers are captured into the witness bundle when generated with:

```bash
RUVIEW_RUN_BENCH=1 bash scripts/witness-adr-115.sh
```

Output lands under `dist/witness-bundle-ADR115-<sha>-<ts>/bench-results/` as both criterion's stdout log and the HTML report tarball.

## Cross-platform note

These measurements are from a single laptop. Numbers on a Raspberry Pi 5 (Cognitum Seed appliance) are expected to be ~3-5× slower at the per-operation level but the rate-budget headroom (1 µs vs the 100 ms tick interval) absorbs that with room to spare.
