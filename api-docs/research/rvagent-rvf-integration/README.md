# rvAgent + RVF integration for agentic flows in RuView

**Status**: Research (Exploration) — Pre-Proposal
**Date**: 2026-05-24
**Author**: ruv

---

## TL;DR

`vendor/ruvector/crates/rvAgent/` ships a production-grade Rust AI-agent framework with eight composable crates (`rvagent-core`, `-middleware`, `-tools`, `-subagents`, `-backends`, `-a2a`, `-acp`, `-mcp`, `-cli`). The framework already speaks **RVF cognitive containers** as its native state-persistence and inter-agent transport. RuView already uses RVF in `v2/crates/wifi-densepose-sensing-server/src/rvf_container.rs`.

**Integration thesis**: the two systems share a serialization substrate. Wiring `rvAgent` swarms into RuView turns the existing sensing pipeline into the substrate that an agentic flow can read from, reason about, and respond to — without writing a new agent runtime.

Concrete value:

1. **Operator-facing agents** that interpret BFLD / pose / vitals events live ("the kitchen has had no presence for 6 h but the kettle stayed on — page the carer").
2. **In-process subagent coordination** for the multi-cog Cognitum Seed appliance — `cog-pose-estimation`, `cog-person-count`, `cog-ha-matter`, and the new BFLD pipeline can negotiate via rvAgent's CRDT state merging instead of ad-hoc IPC.
3. **Witness chains** (ADR-028 / ADR-110) get an upstream consumer — rvAgent's audit-trail middleware persists per-decision attestations into the same RVF container an operator already verifies.
4. **Local SONA learning** — rvAgent's 3-loop adaptive learning slots in alongside the per-home RuVector thresholds already proposed in ADR-116, with the same in-RAM-only privacy posture BFLD enforces (ADR-118 I2).

---

## 1. What rvAgent ships

| Crate | Role | Key types |
|-------|------|-----------|
| `rvagent-core` | State machine + COW state cloning + budget tracking | `AgentState`, `Message`, `AgiContainer`, `Arena`, `Budget`, `Graph` |
| `rvagent-middleware` | 14 built-in middlewares (security, witness, sanitizer, sona, hnsw) | `PipelineConfig`, `build_default_pipeline()` |
| `rvagent-tools` | Tool definitions + dispatch | `Tool`, `ToolInput`, `ToolOutput` |
| `rvagent-subagents` | Spawn isolated subagents with O(1) state clone | `Subagent`, CRDT merge |
| `rvagent-backends` | LLM provider abstraction (Anthropic, OpenAI, local) | `Backend` trait |
| `rvagent-mcp` | MCP server integration | MCP-style tool registry |
| `rvagent-a2a` / `-acp` | Agent-to-agent transport, agent communication protocol | wire format |
| `rvagent-cli` | Operator CLI | argv parsing |

Selling points relevant to RuView:

- **O(1) state cloning via `Arc`** → can spawn one subagent per sensing zone without copying gigabytes of context.
- **Parallel tool execution** → multiple sensor queries (BFLD presence, vitals BPM, pose) issued in parallel from one rvAgent decision step.
- **Path confinement + env-var sanitization** → operator-facing agents that touch the host filesystem (e.g., reading `data/recordings/`) stay sandboxed.
- **Witness chains** in `rvagent-middleware::witness` → already RVF-formatted; round-trips cleanly with ADR-028.

## 2. What RVF already does in RuView

`v2/crates/wifi-densepose-sensing-server/src/rvf_container.rs` defines the on-disk container format used for:

- ADR-110 witness attestations (`SEG_MANIFEST`, `SEG_META`).
- Soul Signature graphs (`docs/research/soul/specification.md` §3).
- BFLD class-1 (derived) frames once the operator opts into research mode (ADR-118 §1.4).

Each RVF blob is content-addressed (BLAKE3 of the canonical byte representation) and carries a typed segment manifest. The format is intentionally extension-friendly — segment types are `u8` enums, new types can land without breaking older readers.

## 3. The integration surface

Three concrete touchpoints, each shippable independently.

### 3.1 RVF as the rvAgent ↔ RuView wire

rvAgent's `AgiContainer` (`rvagent-core/src/agi_container.rs`, 627 LOC) already produces RVF-compatible blobs as its persistent state format. RuView only needs to define **two segment types** in `rvf_container.rs`:

- `SEG_AGENT_STATE = 0x08` — serialized `rvagent_core::AgentState` (the cloned-on-write tree from `cow_state.rs`).
- `SEG_DECISION = 0x09` — a single agent decision step: tool calls issued, outputs received, witness signature.

With these two segments, an rvAgent session and a RuView sensing session can interleave entries in the same RVF blob. The witness-bundle script (ADR-028) iterates segments by type, so it would attest both halves with one signing pass.

### 3.2 BFLD events as rvAgent tool inputs

`wifi-densepose-bfld::BfldEvent` (iter 13) is already JSON-serializable via `to_json()`. Wrapping it as an `rvagent_tools::ToolOutput` is a 20-line shim: the agent issues a `read_bfld_state()` tool, the runtime returns the latest event JSON, the agent reasons over it. The full event surface (presence/motion/count/identity_risk/zone_id) becomes available as agent context without any new IPC.

`BfldEvent → ToolOutput` mapping:
```rust
impl From<BfldEvent> for ToolOutput {
    fn from(e: BfldEvent) -> Self {
        ToolOutput::json(e.to_json().expect("BfldEvent JSON"))
    }
}
```

### 3.3 cog-* as rvAgent subagents

`cog-pose-estimation`, `cog-person-count`, `cog-ha-matter`, and (proposed) `cog-bfld` already share a packaging convention (ADR-100). Each cog can register as a subagent with rvAgent's hub: the cog implements the `Subagent` trait, exports its tool surface, and inherits the parent agent's CRDT state. The queen agent (`rvagent-queen.md` persona) routes operator queries across the cog mesh.

Concrete example:
- Operator query: "is grandma awake yet?"
- Queen agent fans out to: `cog-bfld` (presence in bedroom), `cog-quantum-vitals` (HR baseline shift), `cog-pose-estimation` (sitting/standing transition).
- Each cog returns within budget; queen synthesizes the answer; witness chain logs the decision for compliance audit.

## 4. Open questions

1. **Workspace inclusion**: is `vendor/ruvector/crates/rvAgent/` already on the v2 workspace path, or does it need to be added as a path dep under `wifi-densepose-bfld` / a new `wifi-densepose-agent` crate?
2. **Async runtime**: rvAgent backends are tokio-based. The BFLD `Publish` trait is intentionally sync (iter 22). A small adapter (sync `Publish` ↔ async `Backend`) probably belongs in a `wifi-densepose-agent` crate, not in BFLD itself.
3. **Privacy class composition**: what's the rvAgent equivalent of BFLD's `PrivacyClass`? `rvagent-middleware::sanitizer` strips at the tool-output boundary; should it consume `PrivacyClass` from the originating BFLD event so the agent never even sees a class-3 identity field?
4. **Soul Signature interaction**: rvAgent's `SoulMatchOracle` integration (ADR-121 §2.6) could be the bridge from the Soul Signature graph (`docs/research/soul/`) to the agent decision layer. Worth a dedicated sub-section.
5. **MCP**: `rvagent-mcp` exposes tools to external MCP clients. Should the BFLD `BfldPipelineHandle::send` surface land as an MCP tool here, or stay private to in-process rvAgent flows?

## 5. Proposed next steps (decision deferred)

- **D1**: Open ADR-124 — "rvAgent + RVF integration for RuView agentic flows" — capturing the segment-type assignments, the cog-subagent contract, and the privacy-class composition rule.
- **D2**: Scaffold `v2/crates/wifi-densepose-agent` with the sync ↔ async adapter and one example tool (`read_bfld_state`).
- **D3**: Add `SEG_AGENT_STATE` and `SEG_DECISION` to `rvf_container.rs` as `#[cfg(feature = "agent")]` segments so the v0 ship doesn't pull rvAgent's transitive deps by default.
- **D4**: Land a one-page demo in `examples/agent-bedroom-check/` showing the queen-agent flow end-to-end against the `BfldPipelineHandle`.

## 6. References

- rvAgent: `vendor/ruvector/crates/rvAgent/README.md`, `rvagent-core/src/agi_container.rs`, `rvagent-middleware/docs/UNICODE_SECURITY.md`
- Agent personas: `vendor/ruvector/crates/rvAgent/.ruv/agents/{rvagent-coder,rvagent-queen,rvagent-tester,rvagent-security}.md`
- RVF container: `v2/crates/wifi-densepose-sensing-server/src/rvf_container.rs`
- ADR-028 (witness): `docs/adr/ADR-028-esp32-capability-audit.md`
- ADR-100 (cog packaging), ADR-110 (witness chain), ADR-116 (cog-ha-matter)
- ADR-118 (BFLD): `docs/adr/ADR-118-bfld-beamforming-feedback-layer-for-detection.md`
- Soul Signature: `docs/research/soul/specification.md`
- BFLD impl branch: `feat/adr-118-bfld-impl`, currently at iter 25 (`e8b4fdbc8`)
