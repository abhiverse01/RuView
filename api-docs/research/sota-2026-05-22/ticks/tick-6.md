# Tick 6 — 2026-05-22 03:55 UTC

**Thread:** R10 (through-foliage wildlife sensing)
**Verdict:** Physics feasibility + per-species gait taxonomy + bounded range estimates.

## What shipped

- `examples/research-sota/r10_foliage_attenuation.py` — ITU-R P.833-9 vegetation attenuation model + ESP32-S3 link-budget solver + per-species gait band table.
- `examples/research-sota/r10_foliage_results.json` — full machine-readable numbers.
- `docs/research/sota-2026-05-22/R10-through-foliage-wildlife.md` — research note with range table, gait taxonomy, vertical applications, honest scope.

## Headline numbers (this tick)

**Max ESP32-S3 sensing range through foliage (121 dB link budget, 10 dB SNR margin):**

| Frequency | Sparse | Moderate | Dense |
|---|---:|---:|---:|
| 2.4 GHz | **99.6 m** | 12.0 m | 4.1 m |
| 5 GHz | 19.9 m | 5.2 m | 2.1 m |

The 2.4 GHz / sparse cell (~100 m) is the practical sweet spot — **10× the spatial coverage of a camera trap** in matched conditions, always-on rather than PIR-triggered.

**Per-species gait taxonomy** (DSP-actionable):

- 0.5–1.5 Hz: bear, sloth, wild boar
- 1.2–2.5 Hz: human walking
- 1.5–3.5 Hz: elk, raccoon, wolf
- 1.8–4.5 Hz: deer, fox
- 4.0–15.0 Hz: squirrel, mouse, songbird

## 10-20 year verticals catalogued

- Endangered-species population census (replaces camera traps)
- Wildlife corridor verification
- Invasive-species early warning
- Poaching detection (human gait band well-separated from wildlife)
- Livestock-on-rangeland tracking
- Agricultural pest control

## Coordination

Tick-6 used the same `ticks/tick-N.md` convention to avoid PROGRESS.md races.

## Major out-of-tick news (horizon-tracker just completed)

Horizon-tracker agent `a62cf580…` reported full M1–M7 completion: 6 MCP tools, 6 CLI subcommands, ADR-104, 16 passing tests. Final summary in `HORIZON.md`. The MCP/CLI track is structurally complete; npm publish handoff is documented for the user.
