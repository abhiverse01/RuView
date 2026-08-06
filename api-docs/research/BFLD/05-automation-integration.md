# BFLD Automation & Ecosystem Integration

## 1. Home Assistant Integration

### 1.1 Entities Exposed by BFLD

BFLD extends the sensing-server's existing HA entity set (ADR-115, 21 entities) with
the following new entities:

| Entity | Type | HA Platform | privacy_class | Default |
|--------|------|-------------|--------------|---------|
| `binary_sensor.bfld_presence` | Boolean | binary_sensor | 2 — anonymous | ON |
| `sensor.bfld_motion` | Float 0..1 | sensor | 2 — anonymous | ON |
| `sensor.bfld_person_count` | Integer | sensor | 1 — derived | ON |
| `sensor.bfld_confidence` | Float 0..1 | sensor | 2 — anonymous | ON |
| `sensor.bfld_identity_risk` | Float 0..1 | sensor (diagnostic) | 1 — derived | OFF |
| `sensor.bfld_zone_activity` | String | sensor | 2 — anonymous | ON |

`bfld_identity_risk` is classified as a diagnostic entity in the HA model — it is
hidden by default in the UI and not included in recorder history unless explicitly
enabled. This matches the operator opt-in posture for class-1 fields.

### 1.2 MQTT Discovery Payload (example for presence sensor)

```json
{
  "name": "BFLD Presence",
  "unique_id": "bfld_presence_<node_id_hash>",
  "state_topic": "ruview/<node_id>/bfld/presence/state",
  "device_class": "occupancy",
  "payload_on": "true",
  "payload_off": "false",
  "device": {
    "identifiers": ["ruview_<node_id_hash>"],
    "name": "RuView BFLD Node",
    "model": "wifi-densepose-bfld",
    "manufacturer": "RuView"
  }
}
```

Topic: `homeassistant/binary_sensor/bfld_<node_id_hash>/presence/config`

### 1.3 HA Blueprints

**Blueprint 1: Presence-driven lighting**

Trigger: `binary_sensor.bfld_presence` changes to `on`.
Condition: Time is between sunset and sunrise.
Action: Turn on `light.living_room` at 40% brightness.
Exit: `binary_sensor.bfld_presence` off for 5 minutes → turn off light.

This blueprint uses only class-2 (anonymous) data. No identity information is required.

**Blueprint 2: Motion-aware HVAC**

Trigger: `sensor.bfld_motion` rises above 0.3 (active movement threshold).
Action: Set `climate.living_room` to comfort mode.
Trigger: `sensor.bfld_motion` stays below 0.1 for 20 minutes (room settled).
Action: Set `climate.living_room` to eco mode.

**Blueprint 3: Identity-risk anomaly notification**

Trigger: `sensor.bfld_identity_risk` rises above 0.8 (high-risk threshold).
Condition: privacy mode is NOT enabled.
Action: Notify user via HA mobile app: "BFLD: High identity-leakage risk detected.
Consider enabling privacy mode."

This blueprint is the only one that touches a class-1 field. The notification is
a privacy-protective action — it alerts the operator that the sensing environment
has changed (e.g., new router firmware, new AP nearby, changed room geometry) in
a way that makes the RF channel more identity-discriminative.

---

## 2. Matter Exposure

Matter clusters expose the absolute minimum set of BFLD outputs. The constraint is
intentional: Matter fabrics can include cloud bridges, and identity-correlated data
must never reach cloud endpoints.

### 2.1 Permitted Matter Clusters

| Matter Cluster | Cluster ID | BFLD Source | Notes |
|----------------|-----------|-------------|-------|
| Occupancy Sensing | 0x0406 | `presence` | `OccupancySensing` attribute `Occupancy` bit 0 |
| Motion Detection | 0x040E (proposed) | `motion` | Published as motion event cluster |
| People Count | — (vendor extension) | `person_count` | No standard cluster yet; use vendor attribute |

### 2.2 Rejected Matter Fields

The following BFLD fields MUST NOT be exposed via Matter regardless of operator
configuration:

- `identity_risk_score`
- `rf_signature_hash`
- `raw_bfi`
- `identity_embedding`
- `compressed_angle_matrix`
- Any future field classified at privacy_class < 2

This rejection is enforced in the `cog-ha-matter` crate (`v2/crates/cog-ha-matter/`),
which filters `BfldFrame` events before populating Matter attribute reports.

### 2.3 Matter Endpoint Configuration

```
Endpoint 1: BFLD Occupancy
  - Cluster: Occupancy Sensing (0x0406)
    - Attribute 0x0000 Occupancy: 0x01 (bitmask, bit 0 = presence)
    - Attribute 0x0001 OccupancySensorType: 0x03 (Other = WiFi RF)
  - Cluster: Basic Information (0x0028)
    - NodeLabel: "BFLD-<node_id_short>"
    - ProductName: "wifi-densepose-bfld"
```

---

## 3. MQTT Topic Structure and ACL Recommendations

### 3.1 Topic Tree

```
ruview/<node_id>/bfld/
    presence/state          # "true" | "false" — class 2
    motion/state            # "0.42" — class 2
    person_count/state      # "1" — class 1
    identity_risk/state     # "0.71" — class 1, disabled by default
    raw/state               # disabled by default, class 0 metadata only
    zone_activity/state     # "living_room" — class 2
    confidence/state        # "0.88" — class 2
    events/bfld_update      # Full JSON event payload — class 2 fields only by default
```

### 3.2 Mosquitto ACL Recommendations

```
# /etc/mosquitto/acl.conf (example)

# BFLD node publishes to its own subtree
user bfld_node_<node_id>
topic write ruview/<node_id>/bfld/#

# Home Assistant reads presence, motion, count, zone, confidence
user homeassistant
topic read ruview/+/bfld/presence/state
topic read ruview/+/bfld/motion/state
topic read ruview/+/bfld/person_count/state
topic read ruview/+/bfld/zone_activity/state
topic read ruview/+/bfld/confidence/state
topic read ruview/+/bfld/events/bfld_update

# HA diagnostic access (operator opt-in required to add this rule):
# topic read ruview/+/bfld/identity_risk/state

# DENY all wildcard subscriptions for anonymous clients:
# (mosquitto default: anonymous clients get no access)

# DENY raw topic for all non-admin users:
# raw/state is never written by default; no read ACL needed
```

### 3.3 TLS Configuration

BFLD should use TLS for all MQTT connections. The BFLD node connects as a TLS client;
the broker must present a certificate matching the expected CA. The sensing-server
already supports mTLS (ADR-115). BFLD inherits this configuration.

---

## 4. Node-RED and OpenHAB Compatibility

BFLD publishes standard MQTT payloads with consistent topic structure. No Node-RED
or OpenHAB plugin is required; standard MQTT input/output nodes work directly.

**Node-RED example flow**:

```json
[
  {"id": "bfld-in", "type": "mqtt in",
   "topic": "ruview/+/bfld/presence/state", "qos": "1"},
  {"id": "filter", "type": "switch",
   "property": "payload", "rules": [{"t": "eq", "v": "true"}]},
  {"id": "notify", "type": "http request",
   "url": "http://ha/api/events/bfld_presence_on"}
]
```

**OpenHAB MQTT binding** (items file):

```
Switch BfldPresence "BFLD Presence" {mqtt="<[broker:ruview/node1/bfld/presence/state:state:default]"}
Number BfldMotion  "BFLD Motion"   {mqtt="<[broker:ruview/node1/bfld/motion/state:state:default]"}
```

---

## 5. cognitum-v0 Federation

The cognitum-v0 appliance (Pi 5, running ruview-mcp-brain on port 9876,
cognitum-rvf-agent on port 9004, ruvector-hailo-worker on port 50051 — see
CLAUDE.local.md) is the fleet coordinator for multi-room correlation.

BFLD events from individual nodes flow to cognitum-v0 via the federation path.
The critical constraint: **identity fields are stripped at the node boundary before
federation**. The stripping happens in the local BFLD emitter (`mqtt.rs`), not in
cognitum-v0. By the time a BFLD event reaches the broker that cognitum-v0 subscribes to,
it contains only class-2 (anonymous) or class-3 (restricted) fields.

### 5.1 Federation Topics

```
# Node-local (not federated):
ruview/<node_id>/bfld/identity_risk/state
ruview/<node_id>/bfld/raw/state

# Federated (forwarded to cognitum-v0 broker):
ruview/<node_id>/bfld/presence/state
ruview/<node_id>/bfld/motion/state
ruview/<node_id>/bfld/person_count/state
ruview/<node_id>/bfld/events/bfld_update
```

### 5.2 cognitum-rvf-agent Role

The `cognitum-rvf-agent` (port 9004) handles cross-node RVF (RuView Frame) container
events. For BFLD, it receives federated presence/motion/count events and can correlate
them for multi-room occupancy (e.g., "person moved from living room node to kitchen
node"). It does not receive or need identity information to perform this correlation —
it uses temporal and spatial proximity, not identity.

### 5.3 Hailo Inference (Future)

The `ruvector-hailo-worker` (port 50051) on cognitum-v0 runs vector similarity on the
Hailo-8 AI accelerator. A future extension could offload BFLD's identity_risk_score
computation to the Hailo worker, keeping the identity embedding local to cognitum-v0
while giving individual nodes the benefit of a larger enrollment pool for risk
calibration. This is explicitly out of scope for the current BFLD spec — it is noted
here as an integration-compatible extension point.
