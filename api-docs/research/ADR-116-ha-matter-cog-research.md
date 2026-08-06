---
title: "ADR-116 Research: Home Assistant + Matter Cognitum Seed Cog"
date: 2026-05-23
author: ruv
status: research-complete
relates-to: ADR-110, ADR-115
sources:
  - https://csa-iot.org/newsroom/matter-1-4-enables-more-capable-smart-homes/
  - https://csa-iot.org/newsroom/matter-1-4-2-enhancing-security-and-scalability-for-smart-homes/
  - https://docs.espressif.com/projects/esp-matter/en/latest/esp32c6/certification.html
  - https://docs.espressif.com/projects/esp-matter/en/latest/esp32s3/optimizations.html
  - https://matter-survey.org/cluster/0x0406
  - https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/
  - https://www.hacs.xyz/docs/publish/integration/
  - https://www.derekseaman.com/2025/11/aqara-fp300-the-ultimate-presence-sensor-home-assistant-edition.html
  - https://www.tommysense.com/
  - https://github.com/francescopace/espectre
  - https://kendallpc.com/fdas-2026-guidance-on-general-wellness-devices-policy-for-low-risk-devices-key-compliance-and-regulatory-insights-for-digital-health-companies/
  - https://www.troutman.com/insights/fdas-2026-guidance-on-general-wellness-devices-policy-for-low-risk-devices/
  - https://community.st.com/t5/stm32-summit-q-a/what-is-the-usual-cost-for-a-matter-certification/td-p/652346
  - https://github.com/p01di/esp32c6-thread-border-router
  - https://libraries.io/npm/ruvllm-esp32
  - https://github.com/ruvnet/RuView/blob/main/docs/adr/ADR-069-cognitum-seed-csi-pipeline.md
  - https://www.matteralpha.com/news/home-assistant-2025-12-adds-enhancements-to-matter-sensor-doorlock-and-covering
  - https://docs.nordicsemi.com/bundle/ncs-3.1.0/page/nrf/protocols/matter/getting_started/testing/thread_one_otbr.html
---

# ADR-116 Research Dossier: Home Assistant + Matter Integration as a Cognitum Seed Cog

**Research question**: How far can we take HA + Matter integration for WiFi-DensePose / RuView, specifically packaged as a Cognitum Seed cog running on the ESP32-S3 Seed appliance?

**Baseline**: ADR-110 (ESP32-C6 mesh firmware, v0.7.0-esp32) and ADR-115 (HA-DISCO MQTT + HA-FABRIC Matter scaffold, v0.7.0) are both merged to main. This research scopes ADR-116.

---

## 1. Matter / Thread Frontier

### 1.1 Current specification state (May 2026)

Matter 1.4 (released November 2024) added Solar Power, Battery Storage, Heat Pump, Water Heater, and Mounted Load Control device types — primarily energy-management expansion. It did NOT add health, wellness, vitals, or biometric device types. The cluster relevant to WiFi-DensePose is the **Occupancy Sensing cluster (0x0406)**, which has been present since Matter 1.1 and reached revision 5 in Matter 1.4.

Matter 1.4.2 (current patch release as of research date) focused on security: vendor-ID cryptographic verification of Fabric Admins, Access Restriction Lists (ARLs) for network infrastructure devices, Certificate Revocation Lists (CRLs), and Wi-Fi-only commissioning without BLE. The Wi-Fi-only commissioning path (no BLE requirement) is directly relevant to the Seed, which hosts its own AMOLED UI and can display QR codes natively.

**Occupancy Sensing cluster 0x0406 feature flags** (Matter 1.4 revision 5): PIR, Ultrasonic, PhysicalContact, ActiveInfrared, **Radar**, **RFSensing**, Vision, Prediction, OccupancyEvent. The `RFSensing` feature flag added in 1.3 is the correct semantic tag for CSI-based WiFi detection — we are not PIR or Radar in the classical sense. Home Assistant 2025.12 added configurable `HoldTime` for occupancy sensors and support for `CurrentSensitivityLevel`, both attributes our MQTT path already carries.

**Breathing rate and heart rate have no Matter cluster today.** The spec does not define a BiomedicalSensing or VitalSigns device type. Until the CSA adds one (no public work item found as of May 2026), vitals must stay on MQTT. This is a hard architectural constraint for the Matter path.

### 1.2 Thread Border Router on ESP32-C6

The ESP32-C6 carries 802.15.4 natively (the same radio used for Thread and Zigbee). Espressif ships a working single-chip Thread Border Router reference design for C6 in `esp-matter`, confirmed by community hardware tests (p01di/esp32c6-thread-border-router on GitHub). The C6 can operate as a Thread BR while simultaneously sensing on 2.4 GHz Wi-Fi — the two radios share the same front-end but schedule airtime independently under ESP-IDF. ADR-110 already initializes the 802.15.4 subsystem (`c6_timesync.c`) for cross-node time sync; adding TBR functionality is a matter of enabling `CONFIG_OPENTHREAD_BORDER_ROUTER=y` in the C6 sdkconfig overlay, adding the `esp_openthread_border_router_init()` call, and exposing the backbone interface (Wi-Fi STA).

**Thread 1.4 (TREL)**, shipped with Apple tvOS 26 in late 2025, adds Thread Radio Encapsulation Link — Thread traffic tunneled over Wi-Fi as a fallback backhaul. The C6's Wi-Fi 6 radio supports this. TREL removes the hard dependency on a BR for cross-subnet Thread commissioning, which means a C6-equipped Seed node could participate in a Thread fabric without a dedicated BR appliance.

### 1.3 Matter Commissioner / Root mode

In Matter terms, a Commissioner is a distinct role from an Accessory (end device) or Bridge. The Matter spec allows a device to be simultaneously a Fabric member (commissioned) and a Commissioner (able to commission other devices). The `chip-tool` in `connectedhomeip` is the canonical embeddable commissioner implementation. Running chip-tool on the S3 (512 KB SRAM + 8 MB PSRAM) is feasible but borderline — the commissioner stack requires Thread discovery, BLE central, and certificate-chain verification, adding approximately 400–600 KB RAM footprint on top of the application. On the S3 with 8 MB PSRAM mapped to heap this is tractable; on the C6 (320 KB SRAM, no PSRAM) it is not.

**Practical recommendation**: the Cognitum Seed (S3 + PSRAM + full appliance OS) is the right place to host a Matter commissioner, not the C6 sensing nodes. The Seed can use its existing bearer-token API surface and its cognitum-fleet process (port 9002) as the orchestration layer that opens commissioning windows and bootstraps C6 nodes into the Fabric. C6 nodes remain Accessories only.

### 1.4 CSA certification path

Certification requires: (1) CSA membership (~$22,500/year for full member; lower tiers exist), (2) an Authorized Test Laboratory (ATL) engagement (~$10,000–$19,540 per product for lab fees and certification application), (3) PICS/PIXIT XML submission, (4) hardware shipping to the ATL, and (5) registration on the Distributed Compliance Ledger (DCL). Espressif provides pre-certified radio modules (ESP32-C6-MINI-1, ESP32-S3-MINI-1) which can reduce retesting scope under CSA's Rapid Recertification program — only clusters/device-types added beyond the pre-certified baseline require full ATL re-test. Using `esp-matter` with a pre-certified Espressif module, the realistic total cost for bridge certification is **$30,000–$42,000 first year, $22,500/year thereafter** for a full CSA member, or less if using a pass-through arrangement via an ODM partner that already holds membership.

**Alternative**: publish as "Works with Home Assistant" (free, no CSA ATL, just integration tests) and defer CSA certification to v1.1 when commercial customers require it. The `RFSensing` device class and OccupancySensing cluster are already well-supported in the HA Matter integration without certification.

**Key sources**: [Espressif Matter certification guide](https://docs.espressif.com/projects/esp-matter/en/latest/esp32c6/certification.html), [CSA certification process overview](https://csa-iot.org/certification/), [ST community cost discussion](https://community.st.com/t5/stm32-summit-q-a/what-is-the-usual-cost-for-a-matter-certification/td-p/652346), [Nordic Rapid Recertification notes](https://devzone.nordicsemi.com/f/nordic-q-a/116005/csa-iot-rapid-recertification-program), [ESP32-C6 single-chip TBR](https://github.com/p01di/esp32c6-thread-border-router).

---

## 2. HACS Distribution

### 2.1 What HACS unlocks beyond MQTT auto-discovery

MQTT auto-discovery (HA-DISCO, shipped in ADR-115) creates entities automatically but the integration surface is constrained:

| Capability | MQTT auto-discovery | HACS Python integration |
|---|---|---|
| Config flow (UI setup without YAML) | no — user edits MQTT broker settings manually | yes — wizard walks user through seed URL, token, privacy options |
| Repairs API | no | yes — surfaces structured error reasons ("node offline", "firmware mismatch") as HA repair cards |
| Diagnostics download | no | yes — button in HA device page exports a JSON bundle for bug reports |
| Re-authentication flow | no | yes — handles token expiry without user needing to touch YAML |
| Device registry deep links | partial — via_device works | yes — full device info page, firmware version, last-seen, signal quality |
| Service actions | no | yes — `wifi_densepose.set_privacy_mode`, `wifi_densepose.calibrate_zone` as typed HA services |
| Config entry options | no | yes — change polling interval, privacy mode, zone layout from HA UI |
| Translations (i18n) | no | yes — strings.json enables localized entity names and setup UI |
| Integration quality scale tier | n/a | bronze is minimum; gold (diagnostics + repairs + discovery) is the target |
| HACS listing | not applicable | yes — users install via HACS Store in one click |

### 2.2 Quality Scale targets

HA's quality scale has four tiers. **Bronze** (19 rules) is the minimum: config_flow, unique entity IDs, test coverage, basic documentation. **Silver** adds 95%+ test coverage and re-authentication. **Gold** adds repairs flows, diagnostics, reconfiguration flows, device categories and translations — this is the target for a v1 HACS integration because it meets the bar set by well-regarded third-party integrations like Z-Wave JS and ESPresense. **Platinum** adds strict typing, async dependency injection, and websession management — worth pursuing but not on the v1 critical path.

### 2.3 HACS submission requirements

HACS requires: public GitHub repo, repo description, topic tags, README, single custom component at `custom_components/wifi_densepose/`, `manifest.json` with `domain`, `documentation`, `issue_tracker`, `codeowners`, `name`, `version` fields, and a `brand/icon.png`. No formal approval process — listing is automatic once requirements are met via HACS default repositories submission. HA's `hassfest` CI tool validates the manifest structure and can be added to the repo's CI pipeline as a workflow step.

The `hacs.integration_blueprint` template (github.com/jpawlowski/hacs.integration_blueprint) provides a well-maintained starting point with all boilerplate including config flow, repairs, diagnostics, and translations scaffolding.

**Key sources**: [HA quality scale rules](https://developers.home-assistant.io/docs/core/integration-quality-scale/rules/), [HACS publish guide](https://www.hacs.xyz/docs/publish/integration/), [HACS 2.0 announcement](https://www.home-assistant.io/blog/2024/08/21/hacs-the-best-way-to-share-community-made-projects-just-got-better/), [integration blueprint](https://github.com/jpawlowski/hacs.integration_blueprint).

---

## 3. Cog Architecture for the Seed

### 3.1 Current cog packaging model

Based on ADR-069 and the cognitum-v0 appliance surface observed in the fleet:

- Cogs are signed binaries distributed via GCS buckets and cataloged at `GET /api/v1/edge/registry` (ADR-102).
- Each binary is verified against an **Ed25519 signature** before installation (ADR-100). The device-bound keypair lives in NVS on the Seed.
- Cog binaries are platform-specific: `aarch64` for Pi-based Seed appliances, `x86_64` for the desktop appliance, and (from ADR-069) the feature-vector packet format (`edge_feature_pkt_t`, magic `0xC5110003`) defines the ESP32 side of the protocol. The cog runs on the Seed appliance, not directly on the ESP32.
- The registry catalog at `seed.cognitum.one/store` lists 105 cogs with capability declarations. The Seed's `cognitum-ota-registry` (port 9003) handles OTA delivery.
- Capability declarations include dependency lists, required Seed version, permission scopes (network, storage, MCP tool invocations), and resource budgets (max RAM, max CPU).

### 3.2 Proposed HA+Matter cog architecture

The cog runs as a long-lived process on the Seed (aarch64 binary, supervised by `cognitum-agent`). It owns two surfaces:

**Surface A — MQTT bridge**: connects to a user-configured Mosquitto broker (or uses the Seed's internal broker), republishes telemetry from the Seed's `ruview-vitals-worker` (port 50054) as HA auto-discovery messages. This reuses the HA-DISCO logic already in `wifi-densepose-sensing-server` but runs as a Seed-native cog rather than requiring the user to run the sensing-server separately. The cog registers a `ha_mqtt` MCP tool (114-tool catalog) so automations running on other cogs can call `ha_mqtt.publish_state(entity_id, state)`.

**Surface B — Matter bridge**: wraps `esp-matter` / `matter-rs` as a Matter Accessory Bridge. The Seed acts as a WiFi-connected Matter Bridge — one Fabric node with N dynamic endpoints, one per sensing zone. Device types used: `OccupancySensor` (0x0107, clusters: `OccupancySensing 0x0406` with `RFSensing` feature flag + `BooleanState 0x0045`), `ContactSensor` for fall events, and a vendor-specific numeric attribute for person count on the Bridge root endpoint. The Seed's AMOLED display shows the Matter QR code for commissioning — no phone or scanning app required.

**Surface C — HA HACS integration (optional for users without MQTT)**: a Python package in `custom_components/wifi_densepose/` that speaks directly to the Seed's REST API (`/api/v1/`, bearer token from cognitum-agent on port 80) and bootstraps config flow, entities, repairs, and diagnostics as described in §2.

**Deployment topology**: Seed acts as hub for all sensing nodes (ESP32-S3 and C6). Nodes stream feature vectors to the Seed over UDP (ADR-069 protocol). The cog translates these into HA entities, Matter endpoints, and (via Surface C) HACS entity objects. One cog install covers an unlimited number of ESP32 nodes behind that Seed.

### 3.3 Should the cog speak MQTT or publish Matter directly?

**MQTT to local HA is the lower-risk, faster path**: it requires no Matter SDK linkage, no CSA certification, and reuses the existing HA-DISCO logic. Matter direct publishing requires the Seed to hold a valid Fabric certificate (obtained through the commissioning flow with the user's HA or Apple Home controller), manage operational credentials, and handle rekey events. The overhead is manageable on the Seed (S3 processor + Pi aarch64 appliance stack), but the development and QA cost is 3-4x higher. The recommended architecture is: **MQTT as primary, Matter as secondary** — matching ADR-115's dual-protocol decision but now native to the cog.

**Key sources**: [ADR-069 CSI pipeline](https://github.com/ruvnet/RuView/blob/main/docs/adr/ADR-069-cognitum-seed-csi-pipeline.md), [ESP32 Matter Bridge example](https://project-chip.github.io/connectedhomeip-doc/examples/bridge-app/esp32/README.html), [Tasmota Matter internals](https://tasmota.github.io/docs/Matter-Internals/), [cognitum-v0 fleet stack].

---

## 4. Local-First AI: ruvllm + RuVector on the Seed

### 4.1 Hardware budget

The Cognitum Seed (ESP32-S3 variant: 8 MB PSRAM + 16 MB flash; Pi 5 variant: 8 GB RAM, Hailo AI hat) has two distinct execution environments. For on-Seed inference the numbers differ dramatically:

| Target | RAM headroom for inference | Flash/storage | Typical INT8 model ceiling |
|---|---|---|---|
| ESP32-S3 (8 MB PSRAM) | ~5 MB after OS + MQTT + Matter stack | 16 MB flash | 3–5 MB quantized model (e.g., MobileNetV2-class) |
| Pi 5 Seed (8 GB RAM, Hailo-10) | ~6 GB free | NVMe | 40 TOPS hardware acceleration; 7B INT4 models feasible |
| cognitum-v0 Pi 5 (Hailo via ruvector-hailo-worker) | 6 GB RAM + Hailo | NVMe | 40 TOPS; Hailo HEF deployment |

For a **semantic-primitives inference cog running on the ESP32-S3 Seed**, the target is an INT8-quantized classifier that takes the 8-dimensional feature vector (`edge_feature_pkt_t`) as input and outputs 10 semantic primitive probabilities. This is a trivially small model (8 → 64 hidden → 10 outputs, ~10 KB quantized) — it fits entirely in SRAM without needing PSRAM. The ruvllm-esp32 library (npm: `ruvllm-esp32 0.3.3`, cargo: `ruvllm-esp32 0.3.2`) confirms this path: INT8 quantization, HNSW vector search, and SONA self-optimizing adaptation in under 100 µs per query.

### 4.2 SONA fine-tuning loop

The ruvllm SONA (Self-Optimizing Neural Architecture) adapter performs online gradient descent on LoRA-style adapter weights in under 100 µs per query. For the 10-semantic-primitive classifier, this means the Seed can fine-tune its thresholds per-home using occupant feedback without any cloud round-trip:

1. User confirms a false positive via HA notification (e.g., "that was not a fall, I just sat down quickly").
2. Feedback is recorded via the cog's `ha_mqtt.feedback` MCP tool.
3. SONA runs one gradient step on the LoRA adapter weights for the `fall_risk_elevated` primitive.
4. New weights are written to NVS on the ESP32-S3. The witness chain records the adaptation event with a timestamp.

For the Pi 5 Seed with Hailo-10 (40 TOPS), this extends to full 7B-class LoRA fine-tuning using the Hailo HEF pipeline already running at port 50051 (`ruvector-hailo-worker`). The `ruvllm-microlora-adapt` MCP tool in the cog catalog covers this path.

**Latency budget**: 8-dim → 10-output classifier: <1 ms on S3 SRAM (well within 20 Hz update cadence). SONA one-step gradient: <100 µs per adaptation event. Total per-inference overhead: negligible.

### 4.3 RuVector embeddings for room-level semantic context

The Seed's RuVector 2.0.4 integration (ADR-016) maintains HNSW embeddings of CSI feature vectors. The semantic primitives (sleeping, distress, meeting, etc.) can be implemented as HNSW nearest-neighbor lookups against a learned embedding space rather than threshold classifiers — this is more robust to room geometry variation. The `embeddings_rabitq_search` tool (RaBitQ approximate NN) supports sub-millisecond search on the ESP32-S3 PSRAM-hosted index. At 8 dimensions and 1,000 stored vectors, the HNSW index occupies approximately 200 KB — comfortably within PSRAM budget.

**Key sources**: [ruvllm-esp32 on libraries.io](https://libraries.io/npm/ruvllm-esp32), [ESP32-S3 TinyML optimization guide](https://zediot.com/blog/esp32-s3-tinyml-optimization/), [edge LLM deployment 2025](https://kodekx-solutions.medium.com/edge-llm-deployment-on-small-devices-the-2025-guide-2eafb7c59d07), [LoRA-Edge paper](https://arxiv.org/pdf/2511.03765).

---

## 5. Multi-Seed Federation

### 5.1 Discovery mechanisms

Three viable discovery layers for two Seeds in adjacent rooms:

**mDNS**: each Seed already advertises `_ruview._tcp` and `_matter._tcp` on the LAN. A second Seed can discover the first via `mdns-sd` query at startup and register it as a peer node. The cognitum-fleet service (port 9002) already implements fleet orchestration; adding peer-to-peer node registration is an extension of that model. **Caveat**: mDNS is link-local and does not cross VLANs. For multi-VLAN deployments (common in prosumer and commercial setups), a Tailscale overlay (the project already has a fleet on Tailscale — see CLAUDE.local.md) provides routable discovery at the cost of adding the Tailscale daemon to the cog's dependency list.

**Matter multi-admin**: once both Seeds are commissioned to the same Matter Fabric (e.g., via HA's Matter integration), the Fabric provides a shared namespace. However, Matter does not define a cross-device occupancy-handoff event — it only publishes per-device state. Handoff logic must live in HA automations or in the Seed cog's federation layer.

**Direct ESP-NOW mesh (ADR-110)**: the C6 nodes already run ESP-NOW with 99.56% RX reliability. Two Seeds each hosting C6 nodes can use ESP-NOW as the real-time cross-node synchronization bus — one C6 detects motion entering a room, broadcasts the event over ESP-NOW, the adjacent C6 primes its detector, and the Seed coordinator reconciles the two Occupancy states. This is the lowest-latency path (sub-millisecond over ESP-NOW vs. hundreds of milliseconds over MQTT → HA automation → MQTT).

### 5.2 Conflict resolution for simultaneous fall detection

When two sensing nodes both fire `fall_detected=true` within a short window, the cog applies a simple deduplication rule: the detection with the higher `presence_score` wins, and a 5-second exclusion window is applied on the lower-scoring node (matching the fall debounce logic from the firmware — 3-frame consecutive + 5 s cooldown). The winner's event is forwarded to HA as the canonical fall event. The loser is recorded in the witness chain with a `DEDUP_SUPPRESSED` tag for audit.

For cross-room occupancy, the cog maintains a **single-occupancy graph**: if node A detects person_count=1 and node B simultaneously detects person_count=1, and the two nodes are configured as adjacent rooms, the cog checks whether person_count in the home (sum of all node counts) is consistent with known occupant count (configurable, defaults to household size from HA's `persons` entity). Inconsistency triggers a `multi_room_transition` event published to HA rather than both nodes claiming simultaneous presence.

### 5.3 Witness chain for cross-Seed events

ADR-069 defines a SHA-256 tamper-evident witness chain per node. For cross-Seed events, the chain must include a cross-reference: each Seed's witness head at the time of the event is included in the other's chain entry. The cog implements this via a shared `witness_sync` MCP tool that both Seeds call before writing a cross-node event. This produces a bifurcated chain that any third party can verify for temporal consistency.

**Key sources**: [Matter multi-admin guide](https://mattercoder.com/codelabs/how-to-use-multi-admin/), [ESP-NOW mesh ADR-110 witness log](../WITNESS-LOG-110.md), [HA mDNS cross-VLAN thread](https://niksa.dev/posts/ha-vlan/), [home-assistant-matter-hub mDNS issue](https://github.com/t0bst4r/home-assistant-matter-hub/issues/237).

---

## 6. Competitor Analysis

### 6.1 Aqara FP2 and FP300

**FP2** (mmWave, Wi-Fi): presence, person count (up to 5), 30 zones with 320 detection areas, fall detection. HA integration via native Zigbee or Matter (Thread firmware). Matter mode is severely limited per user testing — configurable parameters are stripped and sensitivity settings are unavailable. Zigbee mode (via Zigbee2MQTT) is the recommended HA path. **No vitals (HR/BR), no pose.** Privacy story: local processing, no cloud required for automations.

**FP300** (5-in-1: mmWave + PIR + light + temperature + humidity, Matter-over-Thread): presence (binary only), temperature, humidity, light level. No person count, no fall detection, no vitals. Thread firmware gives 5 HA entities. Matter mode is functional but configuration-limited. Battery-powered (2× CR2450, ~2 years in Thread mode). **Verdict**: Aqara's Matter story is hardware-first but software-limited. Their Matter device class choice is `OccupancySensor` with standard PIR/Radar bitmap — no `RFSensing` flag.

### 6.2 TOMMY (tommysense.com)

Wi-Fi CSI sensing for HA. Uses ESP32 nodes. Exposes zones as binary sensors (MQTT, port 1886) and as Matter `OccupancySensor` endpoints (QR-based pairing). Motion and presence only — no vitals, no pose, no fall detection. Privacy: fully local, one periodic license-check outbound call. Closed-source algorithm and firmware; open-source HA integration. **Pricing**: free trial (1 zone, 2-min pause per 2 min of detection), Pro (unlimited zones, continuous). **Key gap vs RuView**: no HR/BR, no pose keypoints, no fall detection, no witness chain, no SONA adaptation.

### 6.3 ESPectre (github.com/francescopace/espectre)

Open-source CSI motion detection with HA integration (HACS). ESP32-only. Motion detection via RSSI phase variance analysis — no person counting, no vitals, no fall detection. Python-based HA custom component. No Matter support. **Verdict**: proof-of-concept quality; not a commercial competitor but demonstrates demand for the HACS distribution path.

### 6.4 Frigate NVR

Video-based local AI NVR. MQTT integration with HA creates binary sensors (`binary_sensor.frigate_<camera>_person_motion`), person count sensors, and clip/snapshot sensors per camera. All inference on-device (Coral EdgeTPU or Hailo). **Privacy**: fully local, no cloud. Frigate's MQTT entity catalog per camera: 1 camera stream entity, N object detection binary sensors (person, car, dog, etc.), N object count sensors. No vitals, no pose skeleton. Matter support: none in Frigate itself. **Key privacy contrast vs RuView**: Frigate requires cameras (video pixels), RuView uses RF only — privacy advantage in bedrooms, bathrooms, and care settings.

### 6.5 RoomMe (Intellithings)

Bluetooth LE room presence using smartphone proximity. Supports HomeKit and some smart-device ecosystems. No native HA integration, no MQTT, no Matter. High per-unit cost ($69). No vitals, no fall detection. Not a real competitor for the CSI/mmWave presence category.

### 6.6 Competitor entity catalog comparison

| Feature | RuView (ADR-115) | Aqara FP2 | Aqara FP300 | TOMMY | Frigate |
|---|---|---|---|---|---|
| Presence (binary) | yes | yes | yes | yes | yes (person class) |
| Person count | yes | yes (5 max) | no | no | yes (per class) |
| HR / BR | yes | no | no | no | no |
| Pose keypoints | yes (17-pt) | no | no | no | no |
| Fall detection | yes | yes | no | no | no |
| Semantic primitives | yes (10) | no | no | no | no |
| Multi-room handoff | yes (cog) | no | no | no | no |
| Privacy mode | yes (wire-strip) | local only | local only | local only | local only |
| HACS integration | roadmap | no | no | yes | yes |
| Matter native | yes (bridge) | yes (limited) | yes | yes | no |
| Witness chain | yes | no | no | no | no |

**Key sources**: [Aqara FP300 HA review](https://www.derekseaman.com/2025/11/aqara-fp300-the-ultimate-presence-sensor-home-assistant-edition.html), [TOMMY product page](https://www.tommysense.com/), [ESPectre GitHub](https://github.com/francescopace/espectre), [Frigate NVR docs](https://frigate.video/), [mmWave presence sensors 2026 comparison](https://www.linknlink.com/blogs/guides/best-mmwave-presence-sensors-home-assistant-2026).

---

## 7. Regulatory Frontier

### 7.1 FDA classification landscape (2026 update)

The FDA issued updated General Wellness Device guidance on January 6, 2026. Key clarifications relevant to WiFi-DensePose:

**Wellness device criteria** (functions that keep the product outside FDA jurisdiction): the device must (a) have low inherent risk to user safety, (b) make no reference to specific diseases or conditions, and (c) not provide diagnostic or treatment outputs. Examples in the guidance: heart rate monitoring, sleep tracking, activity/recovery metrics, oxygen saturation trends — all qualify as wellness when marketed without diagnostic claims.

**Claims that trigger medical device classification**: any output labeled as "abnormal, pathological, or diagnostic"; recommendations concerning clinical thresholds or treatment; ongoing clinical monitoring or alerts for medical management; substitution for an FDA-approved device. A fall detection feature framed as "alert a caregiver when you might have fallen" is materially different from one framed as "diagnose fall injury" — the former qualifies as wellness under the 2026 guidance; the latter does not.

**The defensible wellness-device position for RuView**: (a) market fall detection as an "activity anomaly notification" not a "medical fall diagnosis"; (b) include explicit disclaimers against diagnostic or clinical use in app-store descriptions, labeling, and HA integration documentation; (c) avoid "medical-grade" accuracy claims for HR/BR readings; (d) position the device as a "smart home occupancy and wellness assistant" rather than a "patient monitoring system."

### 7.2 HIPAA applicability

HIPAA applies only when an entity is a HIPAA "covered entity" (healthcare providers, health plans, clearinghouses) or their "business associate." A consumer smart home product sold direct-to-homeowners is not automatically a covered entity. However, HIPAA applicability is triggered if the Seed's data flows into a covered entity's system (e.g., a care facility's EHR). The privacy-mode flag in ADR-115 (stripping HR/BR/pose at the wire, publishing only semantic state digests) creates a technical barrier to PHI transmission that supports a "not a covered entity" position.

**All 50 US states** impose data breach notification requirements regardless of HIPAA status. The witness chain (SHA-256 tamper-evident audit log per node) satisfies most state-level data-integrity requirements.

### 7.3 Matter Health-Check device class

Matter currently has no "Health" or "Wellness" device class in the formal taxonomy. The closest is `OccupancySensor` with the `RFSensing` feature flag. The device type `0x0107` (OccupancySensor) in the DCL will not trigger any health-device regulatory scrutiny. Using this device type keeps the Seed in the same regulatory category as a smart motion sensor — well outside the medical device perimeter.

**Key sources**: [FDA 2026 General Wellness guidance (Kendall PC)](https://kendallpc.com/fdas-2026-guidance-on-general-wellness-devices-policy-for-low-risk-devices-key-compliance-and-regulatory-insights-for-digital-health-companies/), [Troutman Pepper Locke analysis](https://www.troutman.com/insights/fdas-2026-guidance-on-general-wellness-devices-policy-for-low-risk-devices/), [IEEE Spectrum FDA device rules](https://spectrum.ieee.org/fda-medical-device-rules), [FDA wellness tracker / cybersecurity interlock (Troutman)](https://www.troutman.com/insights/wellness-trackers-medical-status-and-cybersecurity-how-fda-ftc-and-state-laws-interlock/).

---

## 8. Frontier Features Worth Shipping

### 8.1 HACS marketplace listing

**Build cost**: medium (4–6 weeks for a gold-tier integration). **User impact**: very high — one-click install removes the MQTT broker prerequisite for non-power-users.

Architecture: Python package at `custom_components/wifi_densepose/`, config flow that discovers Seeds via mDNS (`_ruview._tcp`) or manual IP, bearer token authentication against `GET /api/v1/status`, full entity catalog matching ADR-115 §3.1 (21 entities per node), repairs for offline nodes, diagnostics export, translations for EN/FR/DE/ES. Start from `hacs.integration_blueprint` template. Submit via HACS default repositories GitHub submission.

### 8.2 Matter Bridge with OccupancySensor / ContactSensor / BooleanState

**Build cost**: high (6–8 weeks including CI test harness with chip-tool simulator). **User impact**: high for Apple Home / Google Home users who don't run HA.

Device type mapping:
- Presence → `OccupancySensor (0x0107)` with `OccupancySensing (0x0406)`, `RFSensing` feature flag set, `HoldTime` attribute wired to sensing-server's zone dwell time.
- Fall detected → `ContactSensor (0x0015)` used as event source (state: `true` for 5 s after fall, then auto-reset) — closest available device type until a FallEvent device type exists in the spec.
- Person count → vendor-specific attribute on the Bridge root endpoint (`VendorSpecificAttributeCount`, cluster 0xFFF1_xxxx namespace).

Memory on S3: baseline Matter stack ~1.5 MB flash, ~195 KB DRAM + PSRAM heap; BLE freed post-commissioning recovers ~100 KB. 16 dynamic endpoints (default maximum, configurable per `NUM_DYNAMIC_ENDPOINTS`) costs ~550 bytes DRAM each. For 8 zones: 8 × 550 = 4.4 KB additional DRAM — well within budget. Wi-Fi-only commissioning (Matter 1.4.2) eliminates BLE requirement, simplifying the Seed hardware path.

### 8.3 Cognitum Seed cog manifest + signing

**Build cost**: low (1–2 weeks). **User impact**: enables one-tap install from the Cognitum Seed store.

Manifest structure (based on ADR-069/ADR-100 patterns):
```json
{
  "id": "cog-ha-matter-v1",
  "version": "1.0.0",
  "platforms": ["aarch64", "x86_64"],
  "min_seed_version": "0.8.1",
  "capabilities": ["network.mqtt", "network.matter", "api.ruview_vitals"],
  "resource_budget": {"ram_mb": 128, "cpu_percent": 15},
  "signing_key_id": "ed25519:ruv-cog-signing-v1",
  "registry_url": "https://seed.cognitum.one/store/cog-ha-matter",
  "ha_integration_repo": "https://github.com/ruvnet/hass-wifi-densepose"
}
```
Binary signing uses the existing Ed25519 keypair infrastructure from ADR-100. The `cognitum-ota-registry` (port 9003) handles delivery. The cog declaration includes the companion HACS integration GitHub URL so the Seed UI can prompt the user to install the HACS companion if they have HA detected on the LAN.

### 8.4 Local SONA fine-tuning loop for per-home thresholds

**Build cost**: low (2–3 weeks, given ruvllm-esp32 already provides the primitives). **User impact**: high — eliminates false positives that are the top complaint for presence/fall sensors in HA forums.

Implementation: HA sends feedback events via an MQTT command topic (`homeassistant/wifi_densepose/<node>/cmd/feedback`). The cog's SONA adapter processes the feedback as a labeled training example and runs one gradient step. After 20 feedback events, it triggers a witness-chain-attested weight checkpoint. The HACS integration surfaces this as a "Improve detection accuracy" button in the HA device page, pointing users to a simple thumbs-up/thumbs-down UI on the last 10 events.

### 8.5 Multi-room presence handoff

**Build cost**: medium (3–4 weeks). **User impact**: high — eliminates the "ghost occupancy" problem where HA thinks two rooms are occupied when a person walks from one to the other.

Implementation: the cog runs a presence graph across all Seeds in the fleet. Nodes declare themselves adjacent via the manifest or via HA area assignment. When person_count transitions (room A: 1→0, room B: 0→1) within a configurable window (default 3 s), the cog publishes a single `multi_room_transition` event to HA with `from_zone` and `to_zone` fields, and holds the `person_count=1` in the destination room rather than briefly showing 0 in both. This is a cog-side state machine, not an HA automation — it runs at 20 Hz loop cadence.

### 8.6 Energy disaggregation: pairing vitals with HA energy entities

**Build cost**: medium (3–4 weeks). **User impact**: medium-high for sustainability-focused users.

Non-Intrusive Load Monitoring (NILM) in HA already exists as a community blueprint (github.com/tronikos NILM blueprint). The opportunity for RuView is the inverse: rather than using energy to infer occupancy, use RuView's presence data to validate NILM's occupancy assumptions. When RuView reports presence_score < 0.1 (no one home) but the NILM model predicts an active appliance load inconsistent with unoccupied state (e.g., a TV left on), HA can surface a "phantom load detected" notification. The cog publishes a `phantom_load_candidate` event when this condition holds for more than 5 minutes. Pairs with HA's Energy dashboard (introduced in 2021, stable since 2023) and the `homeassistant/sensor/<node>/phantom_load/config` MQTT discovery topic.

### 8.7 Privacy-mode "audit logs only"

**Build cost**: low (1 week, extends existing `--privacy-mode` flag from ADR-115). **User impact**: high for HIPAA-adjacent deployments (care facilities, eldercare) and for GDPR-jurisdiction users.

Three privacy tiers:
- `none`: full telemetry (HR, BR, pose, presence, count) published to MQTT and Matter.
- `semantic` (default): HR/BR/pose stripped at wire; semantic primitives (10 states) published only.
- `audit-only`: no MQTT state messages; only SHA-256 digests of events logged to the witness chain on the Seed. HA receives heartbeat-only availability messages. Suitable for deployments where the home network is untrusted or subject to external logging.

The audit-only mode is a defensible HIPAA/GDPR position for integrators deploying in care settings — the Seed holds the event record, the network carries nothing personally identifiable.

---

## Recommended Scope for HA+Matter Cog v1

Ranked by **build cost × user impact** (low cost + high impact first):

| Priority | Feature | Build effort | User impact | Ships in |
|---|---|---|---|---|
| 1 | **Privacy-mode audit-only tier** (§8.7) | 1 week | High (care/GDPR deployments) | v0.7.1 |
| 2 | **Seed cog manifest + signing** (§8.3) | 1–2 weeks | High (Seed store distribution) | v0.7.1 |
| 3 | **Local SONA fine-tuning loop** (§8.4) | 2–3 weeks | High (false-positive reduction) | v0.7.1 |
| 4 | **HACS integration (gold tier)** (§8.1) | 4–6 weeks | Very high (removes MQTT prereq) | v0.7.2 |
| 5 | **Multi-room presence handoff** (§8.5) | 3–4 weeks | High (ghost occupancy fix) | v0.7.2 |
| 6 | **Matter Bridge OccupancySensor + ContactSensor** (§8.2) | 6–8 weeks | High (Apple/Google Home reach) | v0.8.0 |
| 7 | **Energy disaggregation phantom-load** (§8.6) | 3–4 weeks | Medium-high (sustainability niche) | v0.8.0 |
| 8 | **Thread Border Router on C6** (§1.2) | 2–3 weeks (config only) | Medium (Thread-fabric users) | v0.8.0 |
| 9 | **CSA Matter certification** (§1.4) | $30–42k + 3–6 months | Medium (commercial badge) | post-v1.0 |

**Deferred**: Seed-as-Matter-Commissioner (feasible on S3 appliance but requires full chip-tool port; defer to v1.0), full HA quality-scale platinum tier (gold is sufficient for v1 HACS listing), NILM phantom-load (ships as experimental blueprint first, then proper integration).

**Recommended v0.7.1 sprint**: privacy-mode audit tier + cog manifest + SONA fine-tuning = 4–5 weeks total, fully within the existing Rust + ESP32 codebase with no new dependencies. This sprint closes the most impactful gap (care deployments + per-home personalization) before the heavier HACS/Matter work begins.

---

*Research methodology: 8 parallel web search passes, 12 targeted page fetches, cross-referenced against ADR-115 and ADR-110 source files. Evidence grade: High for Matter cluster specifications, FDA guidance, HACS requirements, and ESP32-S3 memory numbers. Medium for CSA certification cost estimates (sourced from forum discussion, not official price list). Low for ruvllm SONA per-home fine-tuning feasibility (derived from library documentation, not benchmarked on Seed hardware). Open question: whether ESP32-S3 PSRAM heap is sufficient for the full Matter Bridge stack alongside the existing sensing-server runtime — a build-and-measure step is needed before committing to the v0.8.0 Matter bridge sprint.*
