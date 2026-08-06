# R10 — Through-foliage wildlife sensing: physics-grounded feasibility

**Status:** physics + per-species gait taxonomy landed · **2026-05-22**

## The 10-20 year vision

Wildlife conservation runs on stale, expensive data: camera traps, scat-DNA surveys, point counts. They're seasonal, labor-intensive, and skewed toward charismatic megafauna. WiFi CSI at 2.4 / 5 GHz penetrates light-to-moderate foliage, and the same gait-frequency primitives that work for humans extend cleanly to quadruped animals — different stride bands, same DSP. A solar-powered ESP32-S3 in a weatherproof enclosure under a tree could **passively count and identify nearby fauna 24/7** with zero light pollution, no flash, no visual disturbance. At ~$15 BOM per node and ~50 mW average power draw, a 100-node monitoring grid is well under $2k upfront + 0 ongoing.

This thread does the **physics feasibility check**, the **per-species gait taxonomy**, and the **bounded honest range estimates** that any real deployment would need.

## Through-foliage propagation (ITU-R P.833-9)

Vegetation attenuation is modelled as `A_v(d) = A_max · (1 − e^(−γd)) · √f`:

| Foliage density | A_max | γ |
|---|---|---|
| Sparse (orchard, savanna) | 20 dB | 0.10 m⁻¹ |
| Moderate (suburban tree cover) | 35 dB | 0.20 m⁻¹ |
| Dense (rainforest canopy) | 50 dB | 0.35 m⁻¹ |

Combined with **free-space path loss** (`FSPL = 32.45 + 20·log10(f·d)` for f in GHz, d in m) and an ESP32-S3 link budget:

```
Tx power (FCC max):          +20 dBm
Tx antenna (PCB):            +2 dBi
Rx antenna (PCB):            +2 dBi
Rx sensitivity (HT20 MCS0): -97 dBm
                            ─────
Total link budget:          121 dB
SNR margin for CSI DSP:     10 dB
Usable budget:              111 dB
```

## Bounded sensing range

`examples/research-sota/r10_foliage_attenuation.py` solves for the distance at which `FSPL + foliage_attenuation = 111 dB`:

| Frequency | Sparse | Moderate | Dense |
|---|---:|---:|---:|
| 2.4 GHz | **99.6 m** | **12.0 m** | **4.1 m** |
| 5 GHz | 19.9 m | 5.2 m | 2.1 m |

**The 2.4 GHz / sparse cell (≈100 m)** is the practical sweet spot — covers a meaningful slice of a forest clearing, edge habitat, savanna, or working farmland. 5 GHz is essentially useless past 20 m once foliage thickens.

For comparison, a typical camera trap covers ~10 m (PIR-trigger range). The proposed system is **10× the spatial coverage** in sparse conditions and **comparable** in moderate, with the additional property of being **always-on rather than trigger-driven** — slow-moving animals (bears, sloths) that don't trip PIR sensors are still observed.

## Per-species gait-frequency taxonomy

Biomechanics literature (Schmitt 2003, Heglund 1988, Gambaryan 1974) gives canonical stride frequencies. The DSP bandpass that the existing `wifi-densepose-signal::vital_signs` already uses for human breathing/heart-rate maps cleanly onto these:

| Species | Stride frequency (Hz) | DSP filter |
|---|---|---|
| Bear, sloth, wild boar | 0.5 – 1.5 | low-band |
| Human walking | 1.2 – 2.5 | mid-band |
| Elk, raccoon, wolf | 1.5 – 3.5 | mid-band |
| Deer | 1.8 – 4.0 | mid-band |
| Fox | 2.0 – 4.5 | mid-band |
| Squirrel | 4.0 – 10.0 | upper-band |
| Mouse, songbird | 5.0 – 15.0 | upper-band |

The bands overlap, so frequency alone isn't a clean classifier — but combined with **temporal pattern** (deer have a 4-beat asymmetric gait, wolves a 4-beat symmetric, bears a 4-beat alternating-pair) and **body-size envelope** (large vs small Doppler shift), per-species classification is plausible from CSI alone.

## What this depends on

For full classification we need labelled wildlife CSI data, which doesn't exist anywhere in the repo or 2026 published SOTA. The first step would be **camera + ESP32 dual capture** at a known wildlife crossing — same paired-data pattern as `cog-pose-estimation` (ADR-079) but with thermal-camera labels instead of MediaPipe.

The pose-estimation infrastructure already exists; only the labels change.

## What this DOES enable today

Even without species classification:

1. **Presence + count.** The `cog-person-count` v0.0.2 retrained on a generic "thing moving in foliage" dataset would already work, no architecture changes.
2. **Crude size-class.** Doppler shift magnitude correlates with body mass × stride velocity. Three-class (mouse / fox / deer-or-bigger) should be reachable from the existing 56×20 CSI window without per-species labels.
3. **Activity rhythm.** Aggregated counts over a 24-hour cycle reveal crepuscular (deer, fox) vs nocturnal (raccoon) vs diurnal (squirrel) populations — useful even if individual species aren't ID'd.

## Honest scope

- **This is a feasibility note, not a measurement.** No real wildlife data has been collected with this pipeline. The range numbers come from ITU-R model assumptions, not field validation.
- **Foliage models are 1-D simplifications** of a 3-D problem. Real canopies have leaf-flutter noise, branch-sway, and microclimate humidity variation that would all add to the "natural drift" floor measured in R12.
- **Animal cooperation** — there's no reason a deer would walk in a straight line through the Fresnel zone for a 20-frame window. Most observations would be partial.
- **Regulatory.** 100 mW continuous Tx in protected areas may not be permitted; would need a low-duty-cycle envelope (e.g. 1-second-per-minute capture window).

## What this DOES NOT prove

- That a specific species can actually be ID'd from CSI alone in field conditions.
- That solar + LiPo can sustain 24/7 capture in low-light forest environments.
- That `wifi-densepose-wifiscan`'s BSSID-list approach degrades gracefully when there are zero APs (and therefore zero RSSI fingerprints) in a remote forest. (Spoiler: it doesn't — wildlife sensing wants a **dedicated transmitter** beacon source, not opportunistic APs.)

## Vertical applications (10-20 year)

- **Endangered-species population census.** Count + activity-rhythm signature for IUCN red-list species. Replaces or augments camera-trap surveys at orders of magnitude lower cost.
- **Wildlife corridor verification.** Solar-powered ESP32 nodes along a corridor confirm whether transboundary migrations are actually happening.
- **Invasive-species early warning.** Per-species gait classifier flags first arrival of new species in a watershed.
- **Poaching detection.** Human gait (1.2-2.5 Hz) is well-separated from wildlife in the gait taxonomy. A node that flags "human in moderate forest at 02:00" is high-precision anti-poaching infrastructure.
- **Livestock-on-rangeland tracking.** Sparse-foliage 100 m range covers a typical paddock perimeter. Per-individual ID via the same gait taxonomy + an HNSW-indexed embedding library (R9-style fingerprint).
- **Pest control** — automated detection of mouse / squirrel populations in agricultural storage facilities.

## Connection back

- **R5** (saliency) — per-species classifiers would need their own saliency maps; the count-saliency may not transfer. Same task-specific issue surfaced in R12.
- **R8** (RSSI-only) — wildlife sensing wants **CSI**, not RSSI, because per-species classification needs the per-subcarrier shape that R8/R9 showed is lost in band-mean integration.
- **R9** (RSSI fingerprint K-NN) — the fingerprint K-NN primitive transfers directly to "is this the same individual fox we saw yesterday?" identity questions, with CSI as input not RSSI.
- **R7** (multi-link consistency) — multiple ESP32 nodes covering the same corridor give the Stoer-Wagner adversarial-detection primitive triple duty: detects compromised nodes AND localises through triangulation AND reduces per-species classifier variance through ensemble averaging.

## What's next on this thread

- Synthetic gait waveform generation: convolve species-canonical stride patterns with the existing CSI motion-band model, see whether per-species frequency separability survives in the model output.
- Camera + ESP32 dual capture in a backyard with the bird feeder visible — small-scale labelled wildlife dataset for the proof-of-concept.
- ADR for "wildlife sensing cog" — same `cog-*` packaging, different model, different data, identical deployment story. Could ship as `cog-wildlife` once labelled data exists.
