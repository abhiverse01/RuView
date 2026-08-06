# BFLD Security Threat Model

## 1. Adversary Classes

### A1 — Passive Sniffer (Curious Neighbor)

**Capability**: WiFi adapter in monitor mode; consumer laptop running Wi-BFI or
tcpdump with CBFR filter. No special access, no relationship to the target network.

**Goal**: Determine occupancy or identity of persons in an adjacent apartment/office.

**Effort**: Low. Wi-BFI is pip-installable. Monitor mode is available on commodity
Linux laptops. No prior knowledge of the target network required — CBFR frames are
broadcast in all directions.

**Relevance to BFLD**: A1 is the LeakyBeam threat (NDSS 2025). BFLD cannot prevent
A1 from capturing BFI from the air. BFLD's job is to ensure its own output does not
make A1's work easier by publishing identity-correlated data on reachable channels.

### A2 — Targeted Stalker

**Capability**: A1 capabilities plus knowledge of the target's device MAC address
(obtainable from BSSID probe requests) and time correlation with known schedules.

**Goal**: Track a specific individual's presence across time or across locations.

**Effort**: Medium. Requires sustained monitoring (hours to days) and a correlation
step.

**Relevance to BFLD**: If rf_signature_hash were stable over time, A2 could correlate
hash sequences across sessions to confirm a specific person's schedule. The daily hash
rotation (Invariant 3) severs this correlation.

### A3 — ISP / Operator

**Capability**: Access to MQTT broker, HA instance, or cloud integration receiving
BFLD events.

**Goal**: Build behavioral profiles of occupants across many homes/installations.

**Effort**: Low if raw or identity-correlated fields are published to the broker.

**Relevance to BFLD**: BFLD restricts what reaches the broker. An operator cannot
accidentally publish identity-correlated data because the privacy gate blocks it at
the node boundary.

### A4 — Nation-State / Law Enforcement

**Capability**: Compelled access to cloud storage, MQTT broker logs, or HA history.
Physical access to the BFLD node with forensic tools.

**Goal**: Retrospectively identify who was present at a location and when.

**Effort**: Depends on what data was logged. If BFLD's invariants hold, the broker
holds only: presence events (boolean), motion scores (float), person counts (integer),
and rotated hashes. None of these are individually re-identifiable.

**Relevant mitigation**: The daily hash rotation means that even log retention is
privacy-preserving: a hash from Monday and a hash from Tuesday, even from the same
person at the same node, are in disjoint hash spaces.

### A5 — Compromised AP Firmware

**Capability**: Malicious AP firmware that modifies the sounding schedule to extract
more identity-discriminative BFI, or that responds to specially crafted packets with
high-resolution channel feedback.

**Goal**: Improve passive capture quality from the node's BFI stream.

**Relevance to BFLD**: BFLD ingests BFI as captured from the air. If the AP is
compromised to produce unusually high-resolution BFI, BFLD's identity_risk_score
will correctly detect the elevated separability and flag the frames at higher risk.
The system is self-normalizing to the quality of what is captured.

### A6 — Supply-Chain Compromise of RuView Node

**Capability**: Modified BFLD binary with the privacy gate removed or with an
exfiltration path added.

**Goal**: Long-term silent collection of identity embeddings or raw BFI.

**Mitigation**: ADR-028's witness-bundle pattern — deterministic SHA-256 of the
pipeline output. A compromised binary would produce different output for the same
input, failing the verify.py check. The BFLD acceptance criterion 6 (deterministic
frame hashes) is the direct countermeasure.

---

## 2. Attack Trees

### AT-1: Passive BFI Capture → Identity Inference

```
Attacker Goal: Re-identify a specific person via BFI
|
+-- Step 1: Place WiFi adapter in monitor mode (A1)
|     |
|     +-- CBFR frames arrive unencrypted (established by NDSS 2025 / BFId)
|
+-- Step 2: Parse Phi/Psi angles using Wi-BFI or equivalent
|     |
|     +-- No modification of target device required (Wi-BFI passive)
|
+-- Step 3: Collect 5-60 seconds of frames
|     |
|     +-- BFId: 5s sufficient at 10 Hz sounding rate for >90% accuracy
|
+-- Step 4: Run identity classifier (BFId architecture or similar)
|     |
|     +-- Requires enrollment (prior reference capture)
|     |     |
|     |     +-- OR: exploit BFLD's rf_signature_hash as a correlation anchor
|     |               (mitigated by daily rotation — AT-2 below)
|
+-- Outcome: Identity label with >90% confidence
```

BFLD mitigation: BFLD does not prevent AT-1 at the air interface. It ensures that
BFLD's own output does not provide the "correlation anchor" in step 4.

### AT-2: Cross-Site Correlation via rf_signature_hash Leak

```
Attacker Goal: Confirm person X visited site A and site B on the same day
|
+-- Prerequisite: Attacker has read access to MQTT broker at both sites
|
+-- Step 1: Collect rf_signature_hash sequences from site A and site B
|
+-- Step 2: Look for matching hashes within the same day_epoch
|     |
|     +-- BLOCKED: site_salt is site-specific and secret.
|           blake3(salt_A ‖ day ‖ features) != blake3(salt_B ‖ day ‖ features)
|           even if features are identical.
|           Two sites with the same person produce hashes in disjoint spaces.
|
+-- Outcome: No match possible. Attack fails structurally.
```

### AT-3: Timing Side-Channel on identity_risk_score

```
Attacker Goal: Infer when a known person is present by monitoring risk score changes
|
+-- Prerequisite: Read access to MQTT topic ruview/<node_id>/bfld/identity_risk/state
|
+-- Step 1: Baseline: collect identity_risk_score during known-empty periods
|
+-- Step 2: Monitor for anomalous spikes correlated with known schedules
|     |
|     +-- Partial mitigation: risk score is not published by default.
|     |     Operator must explicitly enable it.
|     |
|     +-- Residual risk: even with publication enabled, the score measures risk of
|           identification, not identity itself. A high risk score means "this frame
|           is identity-discriminative" not "person X is present."
|
+-- Mitigation: MQTT ACL restricts identity_risk to local broker by default.
+-- Mitigation: privacy_class=3 (restricted) zeros the risk score on output.
```

### AT-4: MQTT Topic Enumeration

```
Attacker Goal: Discover what BFLD data is published and harvest it
|
+-- Step 1: Connect to broker without TLS (if TLS not configured)
|
+-- Step 2: Subscribe to ruview/# wildcard
|
+-- Mitigation: Default mosquitto ACL denies wildcard subscription to anonymous clients.
+-- Mitigation: TLS + client certificates recommended for all BFLD deployments.
+-- Mitigation: ruview/<node_id>/bfld/raw/state is disabled by default.
```

### AT-5: Matter Cluster Abuse

```
Attacker Goal: Extract identity-correlated data via the Matter protocol integration
|
+-- Step 1: Join the Matter fabric as a legitimate controller
|
+-- Step 2: Read clusters exposed by the BFLD Matter endpoint
|     |
|     +-- Available: OccupancySensing (presence), MotionSensor (motion),
|           PeopleCount (person_count)
|     |
|     +-- NOT AVAILABLE: identity_risk_score, rf_signature_hash, raw_bfi,
|           identity_embedding — these are rejected at the Matter boundary.
|
+-- Outcome: Attacker gets presence/motion/count — same as any occupancy sensor.
      No identity-correlated data is accessible via Matter.
```

---

## 3. Trust Boundary Diagram

```
┌────────────────────────────────────────────────────────────────────────┐
│                          BFLD NODE (local)                             │
│                                                                        │
│  WiFi air interface                                                    │
│       │ CBFR frames (unencrypted, passively sniffable by any A1)       │
│       ▼                                                                │
│  ┌──────────────┐    raw BFI   ┌──────────────┐                       │
│  │  BFI         │──────────────│  Feature     │                       │
│  │  Extractor   │  (local RAM) │  Extractor   │                       │
│  └──────────────┘              └──────┬───────┘                       │
│                                       │ features (not BFI)             │
│                                       ▼                                │
│                               ┌──────────────┐    embedding           │
│                               │  Identity    │──────────────┐         │
│                               │  Risk Engine │  (local RAM  │         │
│                               └──────┬───────┘   ring buf)  │         │
│                                      │ risk_score            │         │
│                                      ▼                        │         │
│  ┌───────────────────────────────────────────────────────┐   │         │
│  │                Privacy Gate                           │   │         │
│  │  privacy_class check | hash rotation | field masking  │   │         │
│  └───────┬──────────────────────────────────────────────┘   │         │
│          │ filtered BfldFrame                  [embedding     │         │
│          │ (no raw BFI, no embedding)           NEVER exits  │         │
│          ▼                                      this box]    │         │
│  ┌──────────────┐                                            │         │
│  │  MQTT        │ presence/motion/person_count/risk(opt)     │         │
│  │  Emitter     │────────────────────────────────────────►  │         │
│  └──────────────┘  [TLS recommended]                         │         │
│                                                              │         │
└──────────────────────────────────────────────────────────────┘─────────┘
         │
         │ MQTT (TLS)
         ▼
┌─────────────────────┐         ┌──────────────────────────────────────┐
│  Local Broker       │         │  cognitum-v0 federation endpoint     │
│  (mosquitto)        │──────►  │  (identity fields STRIPPED at node   │
└────────┬────────────┘         │   boundary before federation)        │
         │                      └──────────────────────────────────────┘
         │
         ▼
┌─────────────────────┐         ┌──────────────────────────────────────┐
│  Home Assistant     │──────►  │  Matter Fabric                       │
│  (presence/motion/  │         │  (OccupancySensing / MotionSensor /  │
│   person_count only)│         │   PeopleCount ONLY)                  │
└─────────────────────┘         └──────────────────────────────────────┘
```

---

## 4. Threat Profile per privacy_class Value

| privacy_class | Value | Data exposed outbound | Residual threats |
|--------------|-------|----------------------|-----------------|
| raw | 0 | Derived angles + amplitude proxy + phase proxy + SNR. Never BFI matrix. | Angle sequences are identity-discriminative; use only in controlled research environments. Never default. |
| derived | 1 | All BFLD output fields including identity_risk_score and rf_signature_hash. | Risk score timing side-channel (AT-3). Hash must remain rotated. |
| anonymous | 2 | presence, motion, person_count, zone_activity, confidence. No identity-correlated fields. | Temporal occupancy patterns may leak schedule information. Not identity. |
| restricted | 3 | presence only (binary). All other fields zeroed or suppressed. | Minimal. On/off presence is equivalent to a passive IR sensor. |

---

## 5. Witness / Attestation Strategy

Following ADR-028's pattern, BFLD should produce a deterministic proof bundle:

1. **Reference input**: a fixed seed synthetic BFI matrix (512 bytes, PRNG seed=117)
   stored alongside the test suite.
2. **Expected output hash**: SHA-256 of the serialized `BfldFrame` produced from that
   input, committed to the repository.
3. **CI check**: `verify_bfld.py` — same structure as `archive/v1/data/proof/verify.py`
   — runs in CI and locally. A compromised binary (A6 threat) would change the output
   hash and immediately fail this check.
4. **Witness log**: extend `docs/WITNESS-LOG-028.md` with a BFLD section covering the
   privacy gate and hash rotation.

This attestation does not prevent a runtime compromise, but it raises the cost
significantly: a supply-chain attacker must either (a) match the expected output hash
while also exfiltrating data (computationally infeasible for a hash adversary), or
(b) accept that the tampered binary will be detected on the next verify run.
