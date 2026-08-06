# ADR-122: BFLD RuView Surface — Home Assistant, Matter, MQTT Exposure

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-24 |
| **Deciders** | ruv |
| **Parent** | [ADR-118](ADR-118-bfld-beamforming-feedback-layer-for-detection.md) |
| **Relates to** | [ADR-031](ADR-031-ruview-sensing-first-rf-mode.md) (sensing-first), [ADR-100](ADR-100-cog-packaging-specification.md) (cog packaging), [ADR-115](ADR-115-home-assistant-integration.md) (HA-DISCO + HA-MIND), [ADR-116](ADR-116-cog-ha-matter-seed.md) (Matter cog), [ADR-120](ADR-120-bfld-privacy-class-and-hash-rotation.md) (privacy class) |
| **Companion research** | [`docs/research/soul/`](../research/soul/) — Soul Signature deployments expose enrolled-match diagnostics only over HA, never Matter. See §2.7. |
| **Tracking issue** | TBD |

---

## 1. Context

ADR-115 shipped the RuView Home Assistant surface (21 entities, MQTT auto-discovery, mTLS, privacy mode) on the `wifi-densepose-sensing-server` Rust binary. ADR-116 is packaging this as the `cog-ha-matter` Cognitum Seed cog. BFLD must integrate into this surface without expanding the privacy-sensitive footprint already in production.

The integration must:

1. **Extend HA-DISCO** to advertise BFLD entities via the existing MQTT-discovery scheme.
2. **Reject identity fields at the Matter boundary** — Matter exposes occupancy/motion/people-count only, never `identity_risk_score` or `rf_signature_hash`.
3. **Route MQTT topics by privacy class** — class-2/3 events on the public topic tree, class-1 events on a gated `research/` subtree, class-0 events nowhere.
4. **Federate cleanly into cognitum-v0** — BFLD events from multiple nodes flow through `cognitum-rvf-agent` (port 9004 per CLAUDE.local.md) for cross-node analytics, but identity-derived fields are stripped at the **publishing-node boundary**, not at the federation hub.

---

## 2. Decision

### 2.1 HA entity surface (six new entities per node)

The cog republishes the existing 21 ADR-115 entities and adds:

| Entity ID | Type | Source field | Class gate | Diagnostic |
|-----------|------|--------------|------------|------------|
| `binary_sensor.<node>_bfld_presence` | occupancy | `BfldEvent.presence` | ≥ 2 | no |
| `sensor.<node>_bfld_motion` | gauge `[0,1]` | `BfldEvent.motion` | ≥ 2 | no |
| `sensor.<node>_bfld_person_count` | int | `BfldEvent.person_count` | ≥ 2 | no |
| `sensor.<node>_bfld_zone_activity` | enum | `BfldEvent.zone_activity` | ≥ 2 | no |
| `sensor.<node>_bfld_identity_risk` | gauge `[0,1]` | `BfldEvent.identity_risk_score` | == 2 only | **yes** |
| `sensor.<node>_bfld_confidence` | gauge `[0,1]` | `BfldEvent.confidence` | ≥ 2 | yes |

The `identity_risk` entity is exposed only under privacy class 2 and is flagged `entity_category: diagnostic` so HA dashboards do not promote it to a main-card sensor by default. Under class 3 it is computed but not published (per ADR-121 §2.4).

MQTT discovery payload follows the ADR-115 schema, plus a `bfld_version` attribute matching the `BfldFrameHeader::version` field.

### 2.2 MQTT topic tree

```
ruview/<node_id>/bfld/presence/state              # class >= 2
ruview/<node_id>/bfld/motion/state                # class >= 2
ruview/<node_id>/bfld/person_count/state          # class >= 2
ruview/<node_id>/bfld/zone_activity/state         # class >= 2
ruview/<node_id>/bfld/confidence/state            # class >= 2
ruview/<node_id>/bfld/identity_risk/state         # class == 2 only
ruview/<node_id>/bfld/raw                         # class 1, OFF by default
ruview/<node_id>/bfld/availability                # online/offline marker
```

`raw` (class-1 derived BFI) is **not present** in the discovery payload at all — operators must explicitly subscribe and acknowledge the research-mode caveat. The publishing crate emits `MQTT_RAW_DISABLED` to availability when `privacy_class < 1`.

### 2.3 Mosquitto ACL example

```
# Default-deny everything not explicitly granted
pattern read  ruview/+/bfld/+/state
pattern read  ruview/+/bfld/availability

# Public roles cannot read identity_risk or raw
user public
deny  read ruview/+/bfld/identity_risk/state
deny  read ruview/+/bfld/raw

# Operator role can read identity_risk for diagnostics
user operator
allow read ruview/+/bfld/identity_risk/state

# Research role can read raw (requires class-1 operation)
user research
allow read ruview/+/bfld/raw
```

The cog ships a default ACL template under `cog-ha-matter/etc/mosquitto.acl.d/bfld.conf` for operators who use the embedded broker (ADR-116 §2.2).

### 2.4 Matter cluster boundary

`cog-ha-matter` exposes BFLD via **three Matter clusters** only:

| Matter cluster | Source entity | Notes |
|---|---|---|
| Occupancy Sensing (0x0406) | `binary_sensor.<node>_bfld_presence` | reports binary occupancy + uncertainty (mapped from `confidence`) |
| Boolean State (0x0045) | `sensor.<node>_bfld_motion >= 0.3` | thresholded; raw motion not exposed |
| Occupancy Sensing extension | `sensor.<node>_bfld_person_count` | uses occupancy-sensor count where Matter spec supports |

**Explicitly NOT exposed via Matter**:

- `identity_risk_score`
- `rf_signature_hash`
- `identity_embedding`
- `raw` BFI
- `zone_activity` (zone IDs are site-specific and Matter is a cross-site surface)
- `confidence` (HA-only diagnostic)

The Matter filter is implemented in `cog-ha-matter/src/matter/bfld_filter.rs` as a `MatterSink` trait impl that rejects classes 0 and 1 at compile time (via ADR-120 §2.2 marker types).

### 2.5 Federation with cognitum-v0

`cognitum-rvf-agent` (port 9004) receives BFLD events from multiple nodes. The events arriving at the federation hub are **already class-2/3** — identity-derived fields were stripped at each publishing node. The hub does not see and cannot reconstruct raw BFI or identity embeddings.

The federation contract:

| At publishing node | At cognitum-rvf-agent |
|---|---|
| Strip class-0/1 fields per ADR-120 | Receive class-2/3 events only |
| Rotate `rf_signature_hash` per ADR-120 §2.3 | Aggregate counts; **do not** correlate hashes across sites |
| Sign event with node Ed25519 key | Verify signature; reject unsigned events |

A `federation-witness` script (extending ADR-028) runs nightly on the hub and proves that no class-0/1 fields appeared in any received event over the previous 24 h.

### 2.6 HA blueprints (shipped with the cog)

Three operator-ready blueprints under `cog-ha-matter/blueprints/`:

1. **Presence-driven lighting** — `binary_sensor.*_bfld_presence` ⇒ `light.turn_on/off` with configurable hold time.
2. **Motion-aware HVAC** — `sensor.*_bfld_motion > 0.3` ⇒ raise HVAC setpoint by ΔT.
3. **Identity-risk anomaly notification** — `sensor.*_bfld_identity_risk` exceeds rolling z-score threshold ⇒ HA `notify.*` to the operator with the originating node and the 7-day baseline.

### 2.7 Soul Signature deployment posture

When the cog is compiled with `--features soul-signature`, two additional HA entities are exposed **at class 1 only**, and **never** over Matter:

| Entity ID | Type | Source | Class gate | Matter |
|-----------|------|--------|------------|--------|
| `sensor.<node>_soul_match_id` | string (opaque `person_id`) | Soul Signature match oracle | == 1 only | **rejected** |
| `sensor.<node>_soul_match_score` | gauge `[0,1]` | Match similarity | == 1 only | **rejected** |
| `sensor.<node>_soul_enrollment_quality` | gauge `[0,1]` | Mirror of `identity_risk_score` during enrollment | == 1 only | **rejected** |

These entities are part of the consent-based diagnostic surface for operators running Soul Signature deployments (care homes with explicit GDPR Art. 9 basis, employment with consent, etc.). The Matter cluster boundary in §2.4 already rejects them by type — the `MatterSink` impl only accepts class-2/3 frames, so `soul_match_id` is structurally unreachable through Matter.

Class-3 deployments **disable Soul Signature** entirely: the `match_against_enrolled()` call returns `MatchOutcome::Suppressed` and no soul entities are published. This makes class 3 the correct setting for any deployment where consent is uncertain or where regulators require Soul Signature to be unavailable.

A fourth blueprint ships only when `--features soul-signature` is enabled:

4. **Enrolled-person arrival notification** — `sensor.*_soul_match_id` transitions to a non-null value ⇒ HA `notify.*` to the enrolled person's configured contact (typically themselves or a designated caregiver). Default off; operator must opt in per enrolled person.

---

## 3. Consequences

### Positive

- Six new HA entities give operators a complete BFLD diagnostic dashboard without leaking identity.
- Matter exposure is structurally narrow — the cluster-filter implementation cannot accidentally expose identity fields because the type system rejects them.
- The default ACL template gives operators a working privacy posture out of the box.
- The federation contract makes it explicit that the hub cannot reconstruct identity even from the union of all node events.

### Negative

- The `identity_risk` HA entity exists only under class 2. Operators who run class 3 deployments cannot see the score even in their own dashboard. This is correct but may surprise care-home installers; documentation must be clear.
- Three Matter clusters is conservative — some HA users may want the count exposed as a percentage or rate, which Matter does not support natively.
- HA-blueprint coverage is intentionally small; operators wanting custom automations must work through the YAML surface.

### Neutral

- The federation witness script runs nightly. A short-duration leak between witnesses is possible but bounded — any successful exfiltration of class-1 fields would still need to be reconstructed into identity, which the daily hash rotation breaks.

---

## 4. Alternatives Considered

### Alt 1: Expose `identity_risk` over Matter (Generic Sensor cluster)

Rejected: Matter is a cross-vendor surface; exposing identity-risk there leaks the score to every Matter controller in the home, including third-party hubs the operator may not control. Keep it HA-internal.

### Alt 2: One unified MQTT topic `ruview/<node>/bfld` with JSON payload

Rejected: per-entity topics are the HA-DISCO convention (ADR-115) and let ACLs be field-specific. A unified topic forces an all-or-nothing read policy.

### Alt 3: Federate raw BFI to cognitum-v0 for cross-node analytics

Rejected: violates ADR-120 I1 (raw never leaves the node). Aggregates are sufficient for cross-node analytics; raw centralization is a hard no.

### Alt 4: Default `entity_category: diagnostic = false` for `identity_risk`

Rejected: promoting `identity_risk` to a main-card sensor would surprise operators with an identity-adjacent gauge on their main dashboard. Diagnostic category is the right default.

---

## 5. Acceptance Criteria

- [ ] **AC1**: HA auto-discovery publishes six new entities per node on first connect; HA recognizes all six.
- [ ] **AC2**: Under privacy class 3, `sensor.<node>_bfld_identity_risk` is absent from the MQTT discovery payload.
- [ ] **AC3**: `MatterSink::publish` rejects any frame at compile time when the source has `privacy_class < 2`.
- [ ] **AC4**: The default mosquitto ACL denies `read ruview/+/bfld/identity_risk/state` to the `public` user role.
- [ ] **AC5**: Three HA blueprints install cleanly into a fresh HA install and trigger their configured actions against a mock BFLD event stream.
- [ ] **AC6**: The federation-witness script detects an injected class-1 field in a synthetic event and exits non-zero.
- [ ] **AC7**: Matter occupancy-sensing cluster reports presence within 1 s of an HA `binary_sensor.*_bfld_presence` state change.

---

## 6. References

- ADR-115 (HA-DISCO entity scheme)
- ADR-116 (`cog-ha-matter` cog packaging)
- ADR-120 (privacy class enforcement)
- ADR-121 (identity risk source)
- ADR-100 (cog packaging spec)
- Mosquitto ACL reference: https://mosquitto.org/man/mosquitto-conf-5.html
- Matter spec — Occupancy Sensing cluster (0x0406)
- Cognitum V0 appliance dashboard: `http://cognitum-v0:9000/`
