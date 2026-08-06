# ADR-125: RuView ↔ Apple Home native HAP bridge — direct HomeKit accessory advertisement from the Seed

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-25 |
| **Deciders** | ruv |
| **Codename** | **APPLE-FABRIC** — RuView speaks HomeKit directly so Apple HomePod / Apple TV act as the discovery + automation surface with zero Home-Assistant middle layer |
| **Relates to** | [ADR-115](ADR-115-home-assistant-integration.md) (HA-DISCO MQTT publisher), [ADR-116](ADR-116-cog-ha-matter-seed.md) (cog-ha-matter §P7 left HAP/Matter as a feature-flag stub), [ADR-118](ADR-118-bfld-beamforming-feedback-layer-for-detection.md) (BFLD presence + identity-risk events), [ADR-122](ADR-122-bfld-ruview-ha-matter-exposure.md) (BFLD HA/Matter exposure) |
| **Tracking issue** | TBD |

---

## 1. Context

### 1.1 The misunderstanding worth correcting once

A naive integration tries to **push** data to a HomePod — open a socket, send a JSON-RPC, call an MQTT topic on `homepod.local`. Apple intentionally does not expose that surface. The HomePod is not an endpoint; it is the **Home Hub + Matter Controller + HomeKit Controller + Siri endpoint** for the Apple Home ecosystem on the LAN. It **discovers** accessories that advertise themselves on the local network via Bonjour/mDNS using the HomeKit Accessory Protocol (HAP) or Matter.

The correct direction of flow is therefore:

```text
RuView / Seed
      ↓                  (advertise HAP / Matter accessory on LAN)
HomeKit / Matter accessory
      ↓                  (mDNS discovery)
HomePod
      ↓                  (forwards to Apple Home automation graph)
Apple Home ecosystem (iPhone, Watch, Mac, Siri, automations)
```

### 1.2 What we ship today and where it stops

ADR-115 ships an **MQTT auto-discovery publisher** that talks to Home Assistant. ADR-116's `cog-ha-matter` Cognitum cog wraps that publisher into a Seed-installable artifact with mDNS, an embedded rumqttd broker, RuVector-backed thresholds, and an Ed25519 witness chain. ADR-122 explicitly extends the same publisher with the BFLD presence / identity-risk / Soul-Match topics so a Home Assistant install sees them as auto-discovered entities. The current path to HomePod therefore runs:

```text
RuView sensing-server ──► cog-ha-matter (MQTT HA-DISCO + HA-MIND)
                              ↓
                       Home Assistant broker
                              ↓
                       Home Assistant HomeKit Bridge add-on
                              ↓
                              HomePod
```

This works and the auto-discovery is real, but it introduces a hard dependency: an operator must run Home Assistant, install its HomeKit Bridge integration, and pair the bridge in the Apple Home app. The Seed alone does not appear in Apple Home.

ADR-116 §P7 anticipated this — the `cog-ha-matter` `Cargo.toml` already carries a `matter = []` feature stub with the comment "matter-rs is added in P7; intentionally absent in P1 to keep the dep surface small until the SDK choice is validated." This ADR closes that box.

### 1.3 Why now

Three forces line up in 2026-05:

1. **The BFLD privacy gate (ADR-118 / 120 / 121) is shipped.** Class-2 and class-3 frames are the only ones eligible to cross the Matter boundary (ADR-122 §2.4). Without that gate we could not safely expose RuView signals to a consumer ecosystem. With it, every Anonymous / Restricted event is safe to advertise as a HomeKit sensor.
2. **`@ruvnet/rvagent` (ADR-124) is on npm.** The MCP surface that lets agents query RuView is live. A first-class Apple-Home presence widens RuView's reach from "agents that speak MCP" to "anyone with an iPhone and a HomePod" — the consumer wedge.
3. **The Cognitum Seed Docker image now bundles `cog-ha-matter`** (this branch's `Dockerfile.rust` change, see #794) — the runtime where a HAP advertiser would live is finally a single-image deployment.

### 1.4 Strategic framing

The combination is asymmetric:

| Layer | RuView contributes | Apple Home contributes |
|-------|---------------------|------------------------|
| Sensing | Passive RF presence, breathing, heart rate, fall risk, BFLD identity-risk, through-wall occupancy, longitudinal wellness | (none — Apple has no native RF sensing surface) |
| Adoption | (limited — researcher-grade hardware today) | iPhone, Watch, Mac, HomePod, Apple TV installed base; consumer trust; voice; on-device intelligence |
| UX | (utility CLI + a Web UI) | Home app, Siri, automation engine, notifications, accessibility |
| Trust | Ed25519 witness chain, privacy class gate, local-first | Apple HomeKit local pairing, end-to-end encrypted, no cloud requirement |

RuView supplies the **invisible cognition layer** Apple cannot provide on its own; Apple supplies the **distribution and UX** that an open sensing stack cannot bootstrap. Direct HAP integration removes the only structural barrier between those two layers — Home Assistant as a mandatory intermediary.

---

## 2. Decision

Ship a **native HomeKit / Matter accessory** in the Seed runtime so a freshly-imaged Cognitum Seed appears in the Apple Home app under `Add Accessory → More Options` with **zero Home-Assistant dependency**.

Concretely:

1. Add a `hap-accessory` workspace component that advertises a set of HomeKit characteristics over mDNS using HAP-1.1 (HomeKit Accessory Protocol).
2. The component subscribes to `wifi-densepose-sensing-server`'s WebSocket / BFLD `MqttEvent` stream and maps each privacy-class-2/3 event onto a HomeKit characteristic update.
3. The same Docker image that ships `sensing-server` and `cog-ha-matter` ships the new advertiser as a third entrypoint:

```bash
docker run --network host ruvnet/wifi-densepose:latest hap-accessory --privacy-mode
```

`--network host` (or a macvlan bridge) is required because HAP pairing depends on the accessory and the controller seeing each other's mDNS broadcasts on the same L2 segment — same constraint Home Assistant's HomeKit Bridge has.

### 2.1 Two implementation tracks (decided here together; ship 2.1.a first)

#### 2.1.a — **HAP-python sidecar** (fastest to ship, lands first)

Add a tiny Python entrypoint `bridges/hap-python/ruview_hap.py` using the well-maintained [`HAP-python`](https://github.com/ikalchev/HAP-python) library. The Dockerfile gets a thin Python runtime stage; the entrypoint script polls `sensing-server` over HTTP and pushes characteristic updates into the HAP loop.

```python
# bridges/hap-python/ruview_hap.py (≈80 LOC)
from pyhap.accessory import Accessory
from pyhap.accessory_driver import AccessoryDriver
from pyhap.const import CATEGORY_SENSOR
import urllib.request, json, threading, time

SENSING_URL = "http://127.0.0.1:3000/api/v1"

class RuViewSensor(Accessory):
    category = CATEGORY_SENSOR

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        s_motion = self.add_preload_service('MotionSensor')
        self.c_motion = s_motion.configure_char('MotionDetected')
        s_occ = self.add_preload_service('OccupancySensor')
        self.c_occ = s_occ.configure_char('OccupancyDetected')
        s_temp = self.add_preload_service('TemperatureSensor')
        self.c_temp = s_temp.configure_char('CurrentTemperature')
        threading.Thread(target=self._poll, daemon=True).start()

    def _poll(self):
        while True:
            try:
                v = json.loads(urllib.request.urlopen(f"{SENSING_URL}/vitals").read())
                self.c_motion.set_value(bool(v.get("motion_present")))
                self.c_occ.set_value(int(bool(v.get("occupancy"))))
                if "ambient_temp_c" in v:
                    self.c_temp.set_value(v["ambient_temp_c"])
            except Exception:
                pass
            time.sleep(1.0)

driver = AccessoryDriver(port=51826)
driver.add_accessory(accessory=RuViewSensor(driver, 'RuView Sense'))
driver.start()
```

Pairing flow on the operator's iPhone:

1. Open Apple Home → `Add Accessory` → `More Options`
2. Tap `RuView Sense` (appears via mDNS automatically)
3. Enter the setup code shown in `docker logs` (or pinned in env)
4. Done — Siri can say "Hey Siri, is anyone in the living room?"

Replace the `motion_present` / `occupancy` mappings progressively as RuView capabilities mature: BFLD class-2 `presence` event → `OccupancyDetected`; BFLD class-3 `identity_risk_score > threshold` → `SecuritySystemCurrentState`; `breathing_present` → `OccupancyDetected` (sleep room); `fall_risk` → a programmable switch that fires an Apple Home automation.

Acceptance criteria for 2.1.a:

- A1: `docker run ... hap-accessory --privacy-mode` advertises an `_hap._tcp` service that the HomePod sees within 30s (`dns-sd -B _hap._tcp local.` on a peer Mac shows `RuView Sense`).
- A2: Pairing from Apple Home succeeds and the entity appears in the Home app under the configured room.
- A3: `MotionDetected` flips within 2 s of an actual RF presence detection from a calibrated ESP32 source (`CSI_SOURCE=esp32`).
- A4: Restarting the container preserves the pairing (HAP state persisted under `/var/lib/ruview-hap/`).
- A5: Privacy: the entrypoint refuses to launch without `--privacy-mode` when `RUVIEW_BFLD_PRIVACY_CLASS` is unset, matching the structural invariant I1 (Raw BFI never exits the node — ADR-118 §2.2).

#### 2.1.b — **Rust-native HAP** (single binary, closes ADR-116 P7)

Wire one of the maintained Rust HAP crates into `cog-ha-matter` so the Python sidecar can be removed. Candidate crates:

- [`hap`](https://crates.io/crates/hap) (Sebastian Schmidt) — last published 0.1.0-pre.16, MIT, active in 2024, supports HAP-1.1, has examples for `MotionSensor`, `LightBulb`, `OccupancySensor`. **First choice.**
- [`accessory-server`](https://crates.io/crates/accessory-server) — narrower scope, fewer services
- A future `matter-rs` crate from project-chip — once stable (CHIP SDK Rust bindings are still emerging in 2026-05)

The `matter = []` feature stub in `cog-ha-matter/Cargo.toml` (added in ADR-116 P1) becomes:

```toml
[features]
default = []
mqtt = ["dep:rumqttc"]
matter = ["dep:hap"]          # ADR-125 §2.1.b
```

with a runtime subcommand `cog-ha-matter --mode hap` that mirrors the Python advertiser's accessory set. Single binary, no Python interpreter in the image, matches the all-Rust ethos of the Cognitum Seed (ADR-116 §1.4).

### 2.1.c — **Topology: one HAP bridge, N child accessories** (decided)

The advertiser publishes a **single HAP bridge** (`RuView Sense`) that owns N child accessories — one per logical sensor surface (presence-bedroom, presence-office, vitals-bedroom, semantic-events, …). Operators pair the bridge once; child accessories appear automatically and can be re-assigned to rooms in the Apple Home app.

The alternative — N independent accessories each advertised separately — was rejected. It forces operators to pair RuView once per room (`RuView Bedroom`, `RuView Office`, `RuView Wellness`, `RuView Presence`, …), which becomes messy after the second or third room, and diverges from how every reference HomeKit accessory in the Home app behaves (a Hue bridge with bulbs, an Eve Energy bridge, etc.). Single pairing also makes container restart / re-image trivial — one persisted pairing key, not N.

### 2.1.d — **Identity-risk mapping: semantic events, not probabilistic surveillance** (decided)

`identity_risk_score` is a continuous 0..1 confidence from the BFLD identity-features pipeline (ADR-121 §2.6). It must NOT cross the HomeKit boundary as a raw value, and must NOT be wired to `SecuritySystemCurrentState`. Apple-Home users read security-system state as **"intruder detected"** — exposing a probability there turns RuView into surveillance UX with all the false-positive blame that entails.

Instead, the bridge exposes **thresholded semantic events** that read like ambient awareness, not threat detection:

| Semantic event | HomeKit primitive | Trigger (illustrative) |
|----------------|--------------------|-------------------------|
| `Unknown Presence` | `MotionSensor` (programmable; stateful) | BFLD class-2 presence + no matching SoulMatch oracle hit (ADR-121 §2.6) for > 30 s |
| `Unexpected Occupancy` | `OccupancySensor` (programmable) | Occupancy in a room outside its operator-defined "expected schedule" window |
| `Unrecognized Activity Pattern` | Programmable `Switch` (stateful, momentary) | BFLD longitudinal drift gate (ADR-118 §2.3 / ADR-122 §2.7) fires Reject or Recalibrate |

What stays internal:

- Raw `identity_risk_score` (numeric 0..1) — never published
- Soul-Signature match probability — never published
- `rf_signature_hash` — never published (already enforced by ADR-118 §2.5 / ADR-122 §2.4 — this is the structural invariant restated at the HAP boundary)

The naming is the contract. "Unknown Presence" is *who's-here-and-it's-fine-but-worth-noting*; an end user will write an automation ("turn on the porch light when Unknown Presence is detected after 9pm") without ever thinking it accuses anyone of being an intruder. That semantic framing is the difference between RuView becoming the calm-tech ambient substrate Apple Home needs vs. another paranoid surveillance widget.

This is the part of the ADR that determines whether RuView's HomeKit story ages well or generates the wrong kind of headlines.

### 2.2 What we DO NOT do in 2.1.a or 2.1.b

- **No Matter (CHIP) controller code.** Matter is the long-term play but its SDK in Rust is not yet stable and the certificate provisioning is heavy. HAP-1.1 over Bonjour gives 95% of the UX for 10% of the complexity, today.
- **No direct connection to the HomePod.** As the framing in §1.1 makes explicit, RuView never opens a socket to the HomePod. It advertises; the HomePod discovers.
- **No iCloud account binding.** HAP pairing is local-network-only by design — RuView gets adoption without ever touching Apple ID, which is a privacy story we keep cleanly.
- **No Class-0 (`Raw`) BFI exposure.** Structural invariant I1 (ADR-118 §2.2) holds. Only privacy-class-2 (Anonymous) and class-3 (Restricted) frames may be mapped onto HomeKit characteristics. The advertiser refuses to start in any other mode.

### 2.3 Sequencing

1. **P1** (this ADR-125 + 1 PR) — HAP-python sidecar (§2.1.a) lands as a separate entrypoint in the same Docker image. AC A1–A5 are gates.
2. **P2** (follow-up PR after operator feedback from 5+ Apple Home pairings) — Rust-native HAP (§2.1.b). Replaces P1; P1's `bridges/hap-python/` becomes an archived reference implementation.
3. **P3** (when matter-rs stabilizes) — Matter Controller path (still RuView-as-accessory, but using the Matter clusters rather than HAP-1.1 services). The Cognitum Cog gains a Matter QR code; pairing flow widens to "any Matter-capable controller, not just Apple."

---

## 3. Consequences

### 3.1 Wins

- **Direct discoverability on Apple Home.** A Seed in the kitchen appears as `RuView Sense` in the Home app within seconds of `docker run`. No HA, no MQTT broker, no Home-Assistant HomeKit Bridge add-on.
- **Siri natively answers RuView questions.** "Hey Siri, is anyone in the kitchen?" — the question reaches the HomeKit characteristic without any custom skill or HA template sensor.
- **Apple-Home automations gain ambient triggers** RuView already produces (presence, breathing, fall, identity-risk) for free — they become first-class automation triggers in the Home app's UI.
- **Strategically corrects RuView's distribution problem.** The Apple Home installed base is the largest consumer surface for HomeKit-grade accessories. RuView's sensing IP becomes addressable to that base without an SDK port.
- **Closes ADR-116 §P7** — the long-flagged matter / HAP gap is now scheduled, not deferred indefinitely.

### 3.2 Costs

- **Python runtime in the Docker image (only for 2.1.a, until 2.1.b lands).** Adds ~30 MB to the runtime layer. Mitigation: P2 removes it; P1 isolates the Python dep in a side-stage so the sensing-server / cog-ha-matter layers stay clean.
- **Network-mode constraint.** HAP pairing needs the controller and accessory on the same L2 segment (mDNS broadcasts). Operators who run RuView in a container behind a NAT/bridge need `--network host` or a macvlan — same constraint HA's HomeKit Bridge has, but worth documenting.
- **Pairing state persistence.** HAP-python stores pairing data in a local file; that state must survive container restarts. Volume-mount `/var/lib/ruview-hap/` to a persistent location.

### 3.3 Risks

- **HAP-python maintenance.** The library is community-maintained; if it goes stale, P2 (Rust-native) absorbs the risk. 2.1.a is explicitly a stepping stone, not a long-term commitment.
- **Apple's evolving requirements.** HomeKit Accessory Certification is required to put a HAP logo on hardware, not to ship a software accessory that pairs locally. RuView's container deployment is squarely in the "uncertified developer accessory" lane, which Apple explicitly permits for local pairing. Worth restating in the operator README.
- **Privacy-class enforcement at the bridge boundary.** A bug that lets a class-0 BFI frame's data influence a HAP characteristic update would violate I1. Mitigation: the bridge consumes only the BFLD `MqttEvent` stream (which is already gated by `PrivacyGate` per ADR-120), never raw BFI; tests assert this in the same style as ADR-122 §4.3.

### 3.4 Reversibility

The advertiser is a separate entrypoint — pulling it out is `docker run` without the `hap-accessory` first-arg, identical to today's behavior. Zero impact on `sensing-server` and `cog-ha-matter` operations.

---

## 4. Acceptance test (P1 / §2.1.a)

```bash
# 1. Start a sensing server (simulated source so the test runs anywhere)
docker run -d --name rs -p 3000:3000 -e CSI_SOURCE=simulated \
    ruvnet/wifi-densepose:latest

# 2. Launch the HAP advertiser sidecar in privacy mode
docker run -d --name hap --network host \
    -v /var/lib/ruview-hap:/var/lib/ruview-hap \
    -e RUVIEW_BFLD_PRIVACY_CLASS=2 \
    ruvnet/wifi-densepose:latest hap-accessory --privacy-mode

# 3. From a Mac on the same LAN: should see RuView Sense as HAP
dns-sd -B _hap._tcp local.   # expect: "RuView Sense" within 30 s

# 4. From iPhone Home app: Add Accessory → More Options → RuView Sense
#    Enter setup code from `docker logs hap`
#    Expect: pairing completes, entity appears in selected Room

# 5. Cycle the container; re-open Home app: entity is still paired
docker restart hap
# Expect: no re-pairing prompt; characteristic updates resume
```

---

## 5. Open questions

Two questions from the original draft were resolved during review (§2.1.c and §2.1.d). Genuinely-open questions that follow-up PRs will close:

- **Setup-code derivation.** Derived deterministically from the Seed's Ed25519 witness key (so reinstalls re-use the same code, operator never re-enters), or random per launch (slightly better security, worse UX on container restarts)? Leaning deterministic + witness-key-derived; verify against Apple's HomeKit Accessory Protocol §5.6.5 (setup-code uniqueness) before committing.
- **ESP32 / Cognitum-Seed-class hardware as a direct HAP advertiser** (not via the host appliance). The current decision parks the bridge on the host runtime; a future ADR can evaluate whether an ESP32-S3 with 8MB flash has enough headroom to run HAP-1.1 directly, which would remove the host appliance from the path entirely for single-room deployments.

---

## 6. References

- ADR-115 — Home-Assistant integration (HA-DISCO MQTT publisher)
- ADR-116 — `cog-ha-matter` Seed cog (this is where the `matter` feature stub lives)
- ADR-118 — BFLD beamforming-feedback layer (privacy gate + class invariants)
- ADR-122 — BFLD RuView HA/Matter exposure (current MQTT-based bridge that this ADR's HAP-native path complements)
- HomeKit Accessory Protocol Specification (Non-Commercial Version), Apple — https://developer.apple.com/apple-home/
- HAP-python — https://github.com/ikalchev/HAP-python
- `hap` (Rust) — https://crates.io/crates/hap
