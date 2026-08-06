# ADR-124: rvagent — MCP (stdio + Streamable HTTP) + ruvector npm/TypeScript library for RuView with ruflo integration

| Field | Value |
|-------|-------|
| **Status** | Proposed |
| **Date** | 2026-05-24 |
| **Deciders** | ruv |
| **Codename** | **SENSE-BRIDGE** — a typed bridge between the RuView sensing stack and the MCP agent ecosystem |
| **Relates to** | [ADR-055](ADR-055-integrated-sensing-server.md) (sensing-server), [ADR-095](ADR-095-rvcsi-edge-rf-sensing-platform.md) (rvCSI), [ADR-097](ADR-097-adopt-rvcsi-as-ruview-csi-runtime.md) (rvCSI adoption), [ADR-115](ADR-115-home-assistant-integration.md) (HA-DISCO), [ADR-116](ADR-116-cog-ha-matter-seed.md) (Seed cog), [ADR-117](ADR-117-pip-wifi-densepose-modernization.md) (PIP-PHOENIX), [ADR-118](ADR-118-bfld-beamforming-feedback-layer-for-detection.md) (BFLD) |
| **Tracking issue** | TBD |

---

## 1. Context

### 1.1 The access-layer gap

The RuView / wifi-densepose Rust stack exposes sensing data through three surfaces: a Tokio/Axum HTTP REST API and WebSocket at `wifi-densepose-sensing-server` (ADR-055); an MQTT namespace under `ruview/<node_id>/*` (ADR-115); and an rvCSI edge runtime (ADR-095/096). None of these surfaces speaks Model Context Protocol (MCP).

MCP is the dominant inter-process contract through which AI assistants (Claude, GPT, Codex) invoke external capabilities in 2026. Without an MCP bridge, RuView's sensing primitives are invisible to AI-driven automation workflows. An agent cannot ask "who is in the room?" or "subscribe me to fall alerts" without bespoke HTTP integration code in every consuming agent.

Two concrete user stories that SENSE-BRIDGE resolves:

1. A developer has a Claude Code session and wants to call `vitals.get_heart_rate` from a prompt — today this requires them to write an HTTP fetch, parse JSON, and handle WebSocket reconnect logic; with SENSE-BRIDGE they install `@ruvnet/rvagent` and the tool is available immediately via `claude mcp add rvagent`.
2. A ruflo-orchestrated multi-agent swarm needs real-world presence data to gate a workflow: SENSE-BRIDGE gives the swarm an MCP tool call with the same `mcp__claude-flow__*` signature pattern already used for all other ruflo tools (CLAUDE.md §Ruflo Automation Primitives).

### 1.2 What rvagent is today

Research of the ruvnet npm registry profile and the ruflo GitHub repository (issue #1689) establishes that **rvagent is not yet a published standalone npm package** as of 2026-05-24. The name "rvagent" appears in the ruflo project exclusively as a WASM artifact (`rvagent_wasm_bg.wasm`, 588 KB) bundled with the RuFlo Web UI (PR #1687). That artifact exports 13 WASM functions including `callMcp`, `executeTool`, `listTools`, `listGalleryTemplates`, `searchGalleryTemplates`, and `loadGalleryTemplate`. It is an in-browser MCP client runner, not a RuView-specific MCP server.

There is no `rvagent` package on the npm registry as of this writing. The npm name is therefore available (Q1 in §8). The package name to register is `@ruvnet/rvagent` (scoped form, reduces name-squatting risk) or `rvagent` (unscoped form, simpler `npx` invocation). This ADR proposes `@ruvnet/rvagent`.

The WASM `callMcp` / `executeTool` surface of the existing ruflo rvagent is the functional model for what the new npm package should expose in TypeScript — but the new package is a **server**, not a client, and its tools are RuView-domain-specific rather than general ruflo-gallery tools.

### 1.3 MCP transport landscape as of 2026-05-24

The MCP specification shipped version `2025-03-26` (Streamable HTTP) and `2025-06-18` (current stable) replacing the legacy `2024-11-05` HTTP+SSE transport. Key facts relevant to this ADR:

- **stdio** remains the recommended local transport. Clients launch the MCP server as a subprocess; the server reads JSON-RPC from stdin and writes to stdout. This is the path `claude mcp add <name> -- npx @ruvnet/rvagent stdio` uses (CLAUDE.md §Quick Setup mirrors this pattern for the claude-flow MCP server).
- **Streamable HTTP** (colloquially "SSE" in earlier documentation) replaces the deprecated pure-SSE transport. A single HTTP endpoint at e.g. `POST /mcp` accepts JSON-RPC requests and may respond with `Content-Type: text/event-stream` for streaming, or `application/json` for single-turn responses. The server must validate `Origin` headers and bind to `127.0.0.1` by default (MCP spec security requirement).
- The `@modelcontextprotocol/sdk` npm package (latest stable at time of writing) ships `Server`, `StdioServerTransport`, and `StreamableHTTPServerTransport`. A single `Server` instance can be connected to both transports simultaneously by calling `server.connect(transport)` for each.
- The legacy `SSEServerTransport` from protocol version `2024-11-05` is deprecated but still ship-able for backwards compatibility with older Claude desktop clients. SENSE-BRIDGE will support it behind an `--legacy-sse` flag for a single release cycle, then remove it.

### 1.4 ruvector npm surface

The `ruvector` npm package (version 0.2.x, latest 0.2.25 as of ~2026-05-01) is a napi-rs WASM/Node.js binding of the RuVector Rust crate. It provides:

- HNSW in-memory vector index (sub-0.5 ms query latency, 50 K+ QPS single-threaded)
- 50+ attention mechanisms from the RuVector Rust crate
- FlashAttention-3 SIMD path
- Graph Neural Network support via `@ruvector/gnn`
- Full TypeScript types; ships both ESM and CJS

The `ruvector` package is already a dependency in the existing Rust workspace's napi-rs node bindings (`ruvector-node` crate, version 0.1.29 on crates.io). The npm package and the Rust crate are developed in the same repository (`github.com/ruvnet/ruvector`). SENSE-BRIDGE can depend on `ruvector` directly without needing to add new Rust FFI — the vector ops needed (HNSW index of pose keypoints, embedding storage for AETHER person re-ID) are already exposed in the npm package's public surface.

### 1.5 ruflo integration context

The project's `CLAUDE.md` documents the 3-tier model routing (ADR-026) and the `mcp__claude-flow__*` tool namespace. ruflo exposes 314 native MCP tools. SENSE-BRIDGE adds a new domain namespace `mcp__rvagent__*` that represents RuView sensing capabilities, parallel to but separate from the ruflo tools. The boundary is:
- **ruflo**: agent orchestration, memory, swarm coordination, hooks, task management
- **rvagent / SENSE-BRIDGE**: RuView-specific sensing — presence, vitals, pose, BFLD, semantic primitives

ruflo can call rvagent tools via the standard MCP tool-call mechanism; rvagent does not depend on ruflo at runtime (but may optionally use ruflo memory namespaces for persistence).

---

## 2. Decision

Ship `@ruvnet/rvagent` as a standalone npm TypeScript library that:

1. Exposes a **dual-transport MCP server** (stdio + Streamable HTTP) wrapping RuView sensing primitives.
2. Uses `ruvector` (npm) as the vector storage layer for pose embeddings and AETHER-class semantic search, with no reimplementation of vector ops in TypeScript.
3. Mirrors the Python `wifi_densepose.client.*` surface (ADR-117 P4 — `python/wifi_densepose/client/ws.py`, `mqtt.py`, `primitives.py`) in TypeScript for parity across runtimes.
4. Integrates as a ruflo plugin via the `ruflo-plugin` manifest convention, exposing tools in the `mcp__rvagent__*` namespace callable by ruflo agents.
5. Ships strict TypeScript source, ESM + CJS dual output, Node.js 20+ minimum, type definitions in the tarball, zero bundler required.

---

## 3. Transport comparison

| Dimension | stdio | Streamable HTTP |
|---|---|---|
| **Launch mechanism** | Client forks `npx @ruvnet/rvagent stdio` as subprocess | Client POSTs to `http://host:port/mcp` |
| **Primary use case** | Claude Code, Cursor, IDE plugins — local developer flow | Remote agents, ruflo swarms on separate hosts, browser-based dashboards |
| **Connection state** | One client per server process; process dies with client | Multiple clients per server process; stateless or session-keyed |
| **Streaming** | Newline-delimited JSON on stdout | `text/event-stream` response body |
| **Auth** | None needed (process-level isolation) | Bearer token or mTLS required (per MCP spec security rules) |
| **RuView sensing-server connectivity** | Server process holds a single WebSocket + MQTT connection to sensing-server; results forwarded to client via JSON-RPC | Server process holds a connection pool; session affinity via `Mcp-Session-Id` header |
| **Tailscale fleet** | Works on local node only | Works across Tailscale fleet (cognitum-v0, cognitum-seed-1, ruvultra) with DNS name |
| **Origin validation** | Not applicable | Required; server MUST reject cross-origin requests unless CORS policy explicitly permits |
| **Resumability** | Not applicable (process is co-located) | Optional `Last-Event-ID` header for stream resumption after reconnect |
| **Logging** | stderr — captured by Claude Code, displayed in conversation | Structured JSON to stdout, shipped to ruflo observability (ADR-observability) |
| **Process lifecycle** | Ephemeral — exits when Claude Code session ends | Long-lived — suitable for always-on sensing daemon |
| **When to choose** | Single developer, local ESP32 (COM9), quick scripting | Fleet deployment, multi-agent ruflo swarms, web dashboards |

Both transports are served by the same `Server` instance from `@modelcontextprotocol/sdk`. The only difference is the `Transport` class passed to `server.connect()`.

---

## 4. MCP tool catalog

All tools are in the `ruview` namespace. Input schemas below are TypeScript interface stubs; output types mirror the Python dataclasses from `python/wifi_densepose/client/ws.py` and `primitives.py`.

### 4.1 Tool catalog table

| Tool name | Input interface | Return shape | RuView surface wrapped |
|---|---|---|---|
| `ruview.presence.now` | `{ node_id?: string }` | `{ node_id: string; present: boolean; n_persons: number; confidence: number; timestamp_ms: number }` | `EdgeVitalsMessage.presence` / `EdgeVitalsMessage.n_persons` (ws.py:74-88) |
| `ruview.vitals.get_breathing` | `{ node_id?: string; window_s?: number }` | `{ node_id: string; breathing_rate_bpm: number \| null; confidence: number; timestamp_ms: number }` | `EdgeVitalsMessage.breathing_rate_bpm` (ws.py:82) |
| `ruview.vitals.get_heart_rate` | `{ node_id?: string; window_s?: number }` | `{ node_id: string; heartrate_bpm: number \| null; confidence: number; timestamp_ms: number }` | `EdgeVitalsMessage.heartrate_bpm` (ws.py:83) |
| `ruview.vitals.get_all` | `{ node_id?: string }` | `EdgeVitalsResult` (all fields of `EdgeVitalsMessage` except `raw`) | Full `EdgeVitalsMessage` (ws.py:74-88) |
| `ruview.pose.latest` | `{ node_id?: string }` | `{ node_id: string; persons: PosePersonResult[]; confidence: number; timestamp_ms: number }` | `PoseDataMessage` (ws.py:91-98) |
| `ruview.pose.subscribe` | `{ node_id?: string; duration_s: number; callback_url?: string }` | `{ subscription_id: string; started_at: number; expires_at: number }` | WS stream — streams `PoseDataMessage` events for `duration_s` seconds |
| `ruview.primitives.get` | `{ node_id?: string; primitive: SemanticPrimitiveKind }` | `SemanticPrimitiveResult` | `SemanticPrimitive` + `SemanticPrimitiveEvent` (primitives.py:36-75) |
| `ruview.primitives.list_active` | `{ node_id?: string }` | `{ primitives: SemanticPrimitiveResult[] }` | All 10 ADR-115 semantic primitives (primitives.py:36-45) |
| `ruview.primitives.subscribe` | `{ node_id?: string; primitive?: SemanticPrimitiveKind; duration_s: number }` | `{ subscription_id: string; expires_at: number }` | MQTT topic `homeassistant/+/wifi_densepose_<node>/+/state` (mqtt.py:8-9) |
| `ruview.bfld.last_scan` | `{ node_id?: string }` | `{ node_id: string; identity_risk_score: number; privacy_class: number; n_frames: number; timestamp_ms: number }` | MQTT `ruview/<node_id>/bfld/scan_result` (ADR-118/ADR-121) |
| `ruview.bfld.subscribe` | `{ node_id?: string; duration_s: number }` | `{ subscription_id: string; expires_at: number }` | MQTT `ruview/<node_id>/bfld/*` |
| `ruview.node.list` | `{ }` | `{ nodes: NodeInfo[] }` | MQTT discovery + REST `/api/nodes` |
| `ruview.node.status` | `{ node_id: string }` | `NodeStatusResult` | REST `/api/status` or MQTT will-message |
| `ruview.vector.search_pose` | `{ query_embedding: number[]; k?: number; node_id?: string }` | `{ matches: VectorMatch[] }` | `ruvector` HNSW index of stored pose keypoints (ADR-016) |
| `ruview.vector.store_pose` | `{ pose: PosePersonResult; node_id: string }` | `{ vector_id: string }` | `ruvector` HNSW upsert |

### 4.1a Policy / governance tools (RUVIEW-POLICY)

**Added 2026-05-24 per maintainer review.** Once tools can answer "who is in the room?", the library is no longer middleware — it is environmental intelligence infrastructure, and that changes the trust model. Every sensing tool above MUST route through this policy layer before returning data. The layer is enforced server-side in the MCP server, not client-side, so a malicious or misconfigured agent cannot bypass it.

| Tool name | Input interface | Return shape | Purpose |
|---|---|---|---|
| `ruview.policy.can_access_vitals` | `{ agent_id: string; node_id: string; vital: "breathing" \| "heart_rate" \| "all" }` | `{ allowed: boolean; reason: string; expires_at?: number }` | Gate every `ruview.vitals.*` call. Default-deny when no policy is registered for the (agent_id, node_id) pair. |
| `ruview.policy.can_query_presence` | `{ agent_id: string; scope: "node" \| "fleet"; node_id?: string; zone?: string }` | `{ allowed: boolean; reason: string; redactions?: string[] }` | Fleet-scope presence queries (e.g. "is anyone home?") require explicit scope grant; node-scope is the safer default. |
| `ruview.policy.can_subscribe` | `{ agent_id: string; topic: string; duration_s: number }` | `{ allowed: boolean; max_duration_s: number; reason: string }` | Subscriptions can be denied entirely or capped to a shorter duration than requested (e.g. agent asks for 1 h, policy returns 5 min). |
| `ruview.policy.redact_identity_fields` | `{ payload: Record<string, unknown>; agent_id: string }` | `{ payload: Record<string, unknown>; redacted_fields: string[] }` | Server-side redaction pass applied to every tool return value. Strips `sta_mac`, raw BFLD matrices, and any keypoint set marked `privacy_class >= 2` per ADR-120. Called automatically by the MCP server; agents never see the un-redacted payload. |
| `ruview.policy.audit_log` | `{ agent_id?: string; since_ts?: number }` | `{ events: PolicyAuditEvent[] }` | Returns the policy-decision audit trail for a maintainer-tier agent. Other agents are denied even if they hold valid tool grants — auditability of the auditor is itself a policy decision. |

Policy storage is a local JSON file (`~/.config/rvagent/policy.json` on Unix, `%APPDATA%\rvagent\policy.json` on Windows) backed by a CLI editor (`npx @ruvnet/rvagent policy grant ...`). Schema mirrors the ADR-010 claims-based authorization model where it exists in the Rust workspace, but the npm library keeps a self-contained store so SENSE-BRIDGE can ship without the full claims infrastructure on day one.

**Default policy when no file exists**: deny `ruview.vitals.*` and `ruview.policy.audit_log`; allow `ruview.presence.now` and `ruview.node.list` (coarse, non-biometric); allow `ruview.primitives.list_active` with `redact_identity_fields` applied. This is the "explore safely" default so a new install can sanity-check the agent is wired up without leaking biometric data.

### 4.2 MCP resource catalog

Resources provide read-only data that can be embedded in the LLM context window.

| Resource URI | Description | MIME type |
|---|---|---|
| `ruview://nodes` | JSON list of all discovered nodes (IP, firmware version, capabilities) | `application/json` |
| `ruview://nodes/{node_id}/config` | Node configuration (channel, MAC filter, privacy class) | `application/json` |
| `ruview://nodes/{node_id}/vitals/latest` | Latest `EdgeVitalsMessage` for the node | `application/json` |
| `ruview://nodes/{node_id}/pose/latest` | Latest `PoseDataMessage` | `application/json` |
| `ruview://nodes/{node_id}/bfld/latest` | Latest BFLD scan result | `application/json` |
| `ruview://primitives/schema` | JSON schema for the 10 semantic primitives (ADR-115) | `application/json` |
| `ruview://fleet/topology` | Tailscale-fleet topology (host, TS IP, role) — sourced from local CLAUDE.local.md fleet table | `text/markdown` |

### 4.3 MCP prompt templates

| Prompt name | Description | Arguments |
|---|---|---|
| `ruview.diagnose_node` | Walk the user through node connectivity check, firmware version, and live vitals stream | `{ node_id: string }` |
| `ruview.presence_report` | Summarize presence + persons over a time window in natural language | `{ node_id: string; window_s: number }` |
| `ruview.vitals_alert_rule` | Generate an HA automation YAML fragment for a vitals threshold alert | `{ primitive: SemanticPrimitiveKind; threshold: number }` |
| `ruview.bfld_privacy_audit` | Produce a compliance-ready privacy audit paragraph from the last BFLD scan | `{ node_id: string }` |

---

## 5. Dependency graph

```
@ruvnet/rvagent (npm / TypeScript)
├── @modelcontextprotocol/sdk    ^1.x  — MCP Server, StdioServerTransport,
│                                        StreamableHTTPServerTransport, McpError
├── ruvector                     ^0.2  — HNSW vector index, embedding storage
│                                        (napi-rs native bindings; NO reimplementation)
├── zod                          ^3.x  — Input schema validation for all tool inputs
├── ws                           ^8.x  — WebSocket client to sensing-server /ws/sensing
│   └── @types/ws
├── mqtt                         ^5.x  — MQTT client for ruview/<node_id>/* topics
│                                        (replaces paho-mqtt; mqtt.js is the npm standard)
├── node-fetch / undici           —     — HTTP client for REST endpoints on sensing-server
└── tsup                         (dev) — ESM + CJS dual build

Runtime back-ends (NOT bundled — must be reachable at runtime):
├── wifi-densepose-sensing-server (Rust binary)
│   ├── REST API   :3000  /api/*
│   ├── WebSocket  :8765  /ws/sensing
│   └── MQTT       via local broker or ruview/<node_id>/*
├── MQTT broker    (mosquitto or broker at cognitum-v0:1883)
└── ruvector HNSW index (in-process via napi-rs; no separate service)
```

Key integration boundary: **ruvector is purely in-process**. The HNSW index lives in the `@ruvnet/rvagent` Node.js process memory, populated from pose keypoints received over the sensing-server WebSocket. There is no separate vector service. This matches the architecture of `wifi-densepose-ruvector` (Rust crate in the workspace) which is also in-process.

---

## 6. Python client surface parity table

The Python client in `python/wifi_densepose/client/` (ADR-117 P4) is the canonical reference for the TS surface. TypeScript should mirror it so users see the same domain model across runtimes.

| Python class / enum | File | TypeScript equivalent in @ruvnet/rvagent |
|---|---|---|
| `SensingMessage` | `ws.py:54-60` | `interface SensingMessage` |
| `ConnectionEstablishedMessage` | `ws.py:63-70` | `interface ConnectionEstablishedMessage extends SensingMessage` |
| `EdgeVitalsMessage` | `ws.py:74-88` | `interface EdgeVitalsMessage extends SensingMessage` |
| `PoseDataMessage` | `ws.py:91-98` | `interface PoseDataMessage extends SensingMessage` |
| `SensingClient` (asyncio) | `ws.py:160` | `class SensingClient` (EventEmitter-based, async iterator) |
| `SemanticPrimitive` (enum) | `primitives.py:36-45` | `enum SemanticPrimitive` |
| `SemanticPrimitiveEvent` | `primitives.py:60-75` | `interface SemanticPrimitiveEvent` |
| `SemanticPrimitiveListener` | `primitives.py:84-155` | `class SemanticPrimitiveListener` |
| `RuViewMqttClient` | `mqtt.py:56` | `class RuViewMqttClient` (wraps mqtt.js `MqttClient`) |
| `_topic_matches` | `mqtt.py:237-257` | `function topicMatches(pattern, topic)` |

---

## 7. Implementation plan

```
P1 ──► P2 ──► P3 ──► P4 ──► P5
npm    MCP    MCP    ruvector  npm
scaffold stdio  SSE   integration  publish + ruflo bridge
```

### P1 — Scaffold (1 week)

**Goal**: an installable npm package skeleton that compiles and passes CI.

- [ ] Create `npm/rvagent/` directory in the repo (mirrors `python/wifi_densepose/`). Do not add to `v2/` Rust workspace.
- [ ] `package.json`: name `@ruvnet/rvagent`, version `0.1.0-alpha.1`, `type: "module"`, exports map with `./package.json`, `.` (ESM + CJS), `./stdio`, `./http`.
- [ ] `tsconfig.json`: `strict: true`, `target: ES2022`, `module: NodeNext`, `moduleResolution: NodeNext`.
- [ ] `tsup.config.ts`: dual `esm + cjs` build, `dts: true`.
- [ ] Add `@modelcontextprotocol/sdk`, `ruvector`, `zod`, `ws`, `mqtt`, `tsup` as deps / devDeps.
- [ ] CI job: `npm ci && npm run build` on `ubuntu-latest` with Node 20, 22.
- [ ] Stub `src/index.ts` that exports package version string. Import succeeds.

### P2 — MCP stdio server (2 weeks)

**Goal**: `npx @ruvnet/rvagent stdio` connects to a running sensing-server over WebSocket + MQTT and exposes the tool catalog from §4.1 over stdio transport.

- [ ] `src/server.ts` — create `McpServer` instance, register all tools from §4.1 with Zod input schemas. Tools that require a live sensing-server connection return a structured error `{ error: "SENSING_SERVER_UNAVAILABLE" }` rather than throwing, so the LLM gets useful context.
- [ ] `src/transports/stdio.ts` — `StdioServerTransport` entrypoint. Reads `RUVIEW_HOST` and `RUVIEW_PORT` env vars (default `localhost:8765` WS, `localhost:3000` REST, `localhost:1883` MQTT).
- [ ] `src/sensing/ws-client.ts` — TypeScript port of `python/wifi_densepose/client/ws.py`. Async generator yielding `SensingMessage` variants. Reconnect with exponential back-off (the Python client explicitly does not reconnect — the TS one should, because the stdio process is long-lived).
- [ ] `src/sensing/mqtt-client.ts` — TypeScript port of `python/wifi_densepose/client/mqtt.py` using `mqtt.js ^5`. Per-pattern callbacks, `topicMatches` wildcard helper.
- [ ] `src/sensing/primitives.ts` — `SemanticPrimitive` enum + `SemanticPrimitiveListener`. Mirror of `primitives.py`.
- [ ] Tool implementations for the 5 highest-priority tools: `ruview.presence.now`, `ruview.vitals.get_all`, `ruview.pose.latest`, `ruview.primitives.get`, `ruview.node.list`.
- [ ] Resource implementations: `ruview://nodes`, `ruview://nodes/{node_id}/vitals/latest`.
- [ ] Integration test: spin up `sensing-server --mock-frames` in Docker; assert `npx @ruvnet/rvagent stdio` receives a `ruview.vitals.get_all` tool call response with non-null `breathing_rate_bpm`.
- [ ] `claude mcp add rvagent -- npx @ruvnet/rvagent stdio` smoke-test (manual).

### P3 — MCP Streamable HTTP server (2 weeks)

**Goal**: `npx @ruvnet/rvagent serve --port 3100` starts an HTTP server that serves the full MCP tool catalog over Streamable HTTP (and optionally legacy SSE for backwards compat).

- [ ] `src/transports/http.ts` — `StreamableHTTPServerTransport` backed by an Express 5 or Hono app (Hono preferred for lightweight edge deployability).
- [ ] Session management: issue `Mcp-Session-Id` UUIDs on `POST /mcp` initialize; reject subsequent requests without session header with HTTP 400.
- [ ] Origin validation: configurable `RUVIEW_ALLOWED_ORIGINS` env var; default reject all cross-origin requests (MCP spec security requirement §Streamable HTTP §Security Warning).
- [ ] Auth: optional `RUVIEW_BEARER_TOKEN` env var. If set, require `Authorization: Bearer <token>` on all requests. This mirrors `v2/crates/wifi-densepose-sensing-server/src/bearer_auth.rs`.
- [ ] Legacy SSE compatibility: `--legacy-sse` flag mounts the deprecated `SSEServerTransport` on `/sse` + `/message` for Claude Desktop clients on protocol version `2024-11-05`. Document this as a single-release compat shim.
- [ ] Remaining tools from §4.1: `ruview.vitals.get_breathing`, `ruview.vitals.get_heart_rate`, `ruview.pose.subscribe`, `ruview.primitives.list_active`, `ruview.primitives.subscribe`, `ruview.bfld.last_scan`, `ruview.bfld.subscribe`, `ruview.node.status`.
- [ ] Prompt template registrations from §4.3.
- [ ] Integration test: `curl -X POST http://localhost:3100/mcp` with a `tools/list` request; assert the response lists all 15 tools.
- [ ] Docker Compose entry for local fleet testing: `rvagent` HTTP container talking to `sensing-server` and `mosquitto` containers.

### P4 — ruvector integration (1 week)

**Goal**: `ruview.vector.search_pose` and `ruview.vector.store_pose` tools work end-to-end with a live HNSW index.

- [ ] `src/vector/index.ts` — wrapper around `ruvector` napi-rs bindings. Initialise an HNSW index at server startup; expose `store(id, embedding)` and `search(embedding, k)`.
- [ ] Pose-to-embedding pipeline: when a `PoseDataMessage` arrives from the WS client, extract the 17-keypoint array, normalise to `[-1, 1]` per keypoint coordinate, flatten to a 34-dimensional float vector, store in HNSW with `node_id:person_index:timestamp_ms` as the ID.
- [ ] `src/vector/aether.ts` — AETHER-style cross-viewpoint search (ADR-024): given a pose embedding query, search HNSW index across all stored poses and return the top-k matches with their source node IDs. This enables cross-node person re-identification via the MCP tool without any network call between nodes.
- [ ] Verify that the `ruvector` napi-rs binary loads correctly on Node 20 linux/x86_64, macos/arm64, and windows/amd64. Document any platform-specific caveats.
- [ ] Index persistence: optional `RUVIEW_VECTOR_DB_PATH` env var. If set, persist the HNSW index to disk using `ruvector`'s serialise API. If unset, in-memory only (default for stdio transport).
- [ ] Integration test: feed 100 synthetic pose frames with known clustering, assert `ruview.vector.search_pose` retrieves nearest neighbours with recall >0.9.

### P5 — npm publish + ruflo bridge (1 week)

**Goal**: `npm install @ruvnet/rvagent` works for consumers; ruflo agents can call `mcp__rvagent__*` tools through the standard claude-flow MCP registration.

- [ ] Populate `package.json` with `publishConfig: { access: "public" }`, `engines: { node: ">=20" }`, `files` whitelist (`dist/`, `src/`, `README.md`).
- [ ] Publish `@ruvnet/rvagent@0.1.0-alpha.1` to npm under the `@ruvnet` scope.
- [ ] ruflo plugin manifest: create `.claude/plugins/rvagent/plugin.json` following the ruflo `plugin/` convention in the ruflo repo. The manifest registers the HTTP transport URL (configurable) and maps `mcp__rvagent__*` tool calls to the rvagent MCP server.
- [ ] `ruview` skill in `.claude/agents/` (CLAUDE.md §Available Agents): an agent description that documents the rvagent tool namespace for ruflo orchestration.
- [ ] `claude mcp add rvagent -- npx @ruvnet/rvagent stdio` tested against claude-flow MCP server on the local dev machine (ruvzen host on CLAUDE.local.md fleet).
- [ ] Document the fleet deployment pattern: run `npx @ruvnet/rvagent serve` on cognitum-v0 (Tailscale IP 100.77.59.83, port 50060 range to avoid conflict with existing services; see CLAUDE.local.md services table). Register the URL as a remote MCP server in `.claude/settings.json`.
- [ ] Publish announcement: link from project README (`docs/` link, not root README per CLAUDE.md rules).

---

## 8. Open questions

**Q1. npm package name availability**
`rvagent` (unscoped) does not appear in the npm registry as of 2026-05-24 based on search results. `@ruvnet/rvagent` is definitely available (the `@ruvnet` scope is owned by ruvnet per the npm profile page). Should the package be published unscoped (`rvagent`) for simpler `npx rvagent stdio` invocation, or scoped (`@ruvnet/rvagent`) for namespace clarity? The decision should be made before P5 because the npm name is permanent.

**Q2. ruvector binary compatibility on Windows**
The `ruvector` npm package is a napi-rs native addon. The project's primary development machine (ruvzen) is Windows 11. It is not confirmed whether `ruvector@0.2.25` ships a prebuilt Windows binary in its npm tarball or requires a Rust toolchain to compile. If no Windows binary is shipped, developers on ruvzen would need the Rust toolchain installed to use `@ruvnet/rvagent`. This must be confirmed before P5 by running `npm install ruvector` on ruvzen.

**Q3. ruvector TypeScript API stability**
ruvector `0.2.x` is not a 1.0 release. The HNSW insert and search API surface may change between minor versions. SENSE-BRIDGE P4 should pin `ruvector@~0.2.25` and document the version constraint explicitly. The question is whether ruvector publishes a changelog with breaking-change notices.

**Q4. MCP tool call latency budget — RESOLVED**
Raw sensing frequency ≠ agent interaction frequency. If a tool call ever waits on the next CSI frame, agent orchestration latency becomes physically coupled to RF acquisition jitter, which is unacceptable at scale. The library MUST take option (a) — return from a continuous local cache:

1. **Continuous local cache**: on startup the rvagent MCP server opens one WebSocket + one MQTT subscription per configured sensing-server endpoint and ingests every frame into an in-memory `Map<node_id, EdgeVitalsMessage>` (plus parallel maps for `PoseDataMessage` and BFLD). Cache hits return in <1 ms regardless of CSI frame rate.
2. **Event-driven invalidation**: the cache entry's `received_at` timestamp is bumped on every received frame. The cache itself is never purged on a timer — only overwritten when fresh data lands, so a node that went quiet still serves its last-known value.
3. **Bounded freshness windows**: each tool accepts an optional `max_age_ms` argument (default 1000). If the cached `received_at` is older than `max_age_ms`, the tool returns `{ value: null, reason: "stale", last_seen_ms: N, threshold_ms: max_age_ms }` rather than blocking. The agent decides whether to accept the staleness, raise to the user, or escalate to a `ruview.node.status` health check.

This pattern is required because P3's Streamable HTTP transport may serve dozens of concurrent agent sessions — see Q8. A shared cache + per-session freshness contract scales; per-session WS connections do not.

P2 must implement this cache; P3 must verify that fanning the same cache to N concurrent HTTP sessions still maintains <1 ms median tool-call latency under load.

**Q5. Subscription tool lifetime management**
Tools `ruview.pose.subscribe`, `ruview.primitives.subscribe`, and `ruview.bfld.subscribe` return a `subscription_id` and stream events. In the stdio transport there is one client, so this is straightforward. In the HTTP transport with multiple sessions, subscription state must be tracked per `Mcp-Session-Id`. When a session expires (HTTP 404) or is deleted via HTTP DELETE, the subscription must be cleaned up. The lifecycle mechanism is not fully designed — this is a known gap that P3 must close.

**Q6. AETHER embedding dimension**
The ADR proposes a 34-dimensional pose embedding (17 keypoints × 2 coordinates). The actual AETHER embedding model (ADR-024) uses a learned contrastive encoder, not raw keypoints. If the AETHER ONNX model is available in the Rust workspace at P4 time, the embedding should use it. If not, the raw-keypoint approach is a reasonable placeholder. The question is whether `wifi-densepose-nn` exposes the AETHER encoder in a form that can be called from Node.js without bundling libtorch in the npm package.

**Q7. ruflo plugin manifest format**
The ruflo plugin convention (`plugin/` directory in the ruflo repo) is not fully documented in a public spec as of this writing. The manifest format was inferred from the `ruflo-plugins.gif` directory listing and referenced in issue #952. Before P5, the actual plugin manifest schema must be confirmed from the ruflo repo so SENSE-BRIDGE does not ship an incompatible manifest.

**Q8. MQTT vs direct WebSocket for Streamable HTTP transport**
In the stdio transport, rvagent holds a single WebSocket + single MQTT connection to the sensing-server. In the Streamable HTTP transport (potentially serving dozens of agent sessions), maintaining one connection per session is not scalable. The recommended pattern is a single shared connection per (sensing-server endpoint), multiplexed to all sessions. The implementation complexity of this fan-out is non-trivial and is not fully specified here.

**Q9. Legacy SSE deprecation timeline**
The MCP `2024-11-05` SSE transport is deprecated in the current spec but Claude Desktop versions prior to the spec `2025-03-26` update still use it. SENSE-BRIDGE proposes `--legacy-sse` for one release cycle. The question is which specific Claude Desktop version drops legacy SSE support, and whether any of the active fleet nodes (cognitum-v0, cognitum-seed-1) run a Claude Desktop version old enough to need it.

**Q10. Node.js vs Bun runtime**
The ruflo monorepo uses `bun` as the primary runtime (per `bunfig.toml` in `v3/`). Should `@ruvnet/rvagent` also support Bun? Bun's napi-rs compatibility for native addons like `ruvector` is improving but not guaranteed for 0.2.x. The P1 CI should test on Node 20 first; Bun support can be declared as a stretch goal for P5.

---

## 9. Alternatives considered

### Alt-A — Python-only client (extend ADR-117 with MCP bindings)

Add `wifi_densepose.mcp` as a P6 module in the PIP-PHOENIX wheel (ADR-117). The Python MCP SDK (`mcp[cli]`) supports both stdio and HTTP transports and the PyO3 bindings give direct access to the sensing types.

**Rejected because**: Python is not the dominant runtime for MCP server hosting in 2026 — the ecosystem tooling (Claude Desktop, Claude Code `mcp add`, ruflo) is TypeScript-first. A Python MCP server requires the full pip install including PyO3 bindings, which is a heavier install than `npx @ruvnet/rvagent stdio`. The ruflo plugin format is TypeScript. ADR-117 is already sizeable; adding MCP to it conflates two distinct concerns (Python developer library vs. AI agent interface). Python MCP remains a viable future addition (Q10 for a future ADR) but is not the right first-ship target.

### Alt-B — Pure WebSocket/REST client without MCP framing

Ship a TypeScript client library `@ruvnet/ruview-client` that wraps the sensing-server WebSocket and REST API without the MCP layer. Consumers who want MCP integration would wrap it themselves.

**Rejected because**: it solves the connectivity problem but not the agent integration problem. Without MCP framing, Claude Code and ruflo agents cannot discover or call RuView capabilities through the standard `mcp__*` namespace — they would need custom prompt injection or bespoke tool definitions per agent. The whole value proposition of this ADR is that a single `claude mcp add rvagent` command makes all RuView primitives discoverable to any MCP-capable AI assistant. Splitting the library forces every consumer to re-add the MCP layer.

### Alt-C — Embed MCP server inside the existing wifi-densepose-sensing-server Rust binary

Add an MCP endpoint to the existing Axum server in `v2/crates/wifi-densepose-sensing-server/` (`v2/crates/wifi-densepose-sensing-server/src/main.rs`). This would use the `rmcp` Rust crate (Model Context Protocol SDK for Rust) and expose MCP over an additional port.

**Rejected because**: (a) it couples the release cycle of the npm-hosted MCP interface to the firmware/Rust release cycle, which are on separate cadences — a new MCP tool that merely adds a JSON field should not require a firmware rebuild; (b) the ruflo plugin ecosystem is TypeScript and expects npm packages, not Rust binaries; (c) the ruvector vector layer is a napi-rs Node.js native module and cannot be called directly from a Rust process without going through the napi-rs server-side API, adding unnecessary complexity; (d) the sensing-server binary is already 15-30 MB stripped — adding the MCP endpoint and its JSON-RPC machinery would further bloat it. This alternative is worth revisiting if the Rust `rmcp` crate matures and the vector layer migrates fully to native Rust, but it is not appropriate for the first implementation.

### Alt-D — Wrapping the existing ruflo WASM rvagent in a RuView shim

The ruflo WASM rvagent (`rvagent_wasm_bg.wasm`) already exports `callMcp` / `executeTool` / `listTools`. One could define a RuView shim that registers custom tools into the ruflo WASM rvagent gallery.

**Rejected because**: the ruflo WASM rvagent is an in-browser MCP *client* runner for the ruflo gallery, not a general-purpose MCP server that can expose sensing data. Its 13 exported functions are focused on template management and ruflo-gallery operations. Patching sensing tools into a browser WASM module is the wrong architecture for a server-side sensing bridge. The naming overlap is a reason to publish the new package promptly and clearly document the distinction.

---

## 10. Compatibility

### 10.1 Backwards compatibility with ADR-117 (PIP-PHOENIX) Python client

SENSE-BRIDGE does not replace the Python client. Both can coexist:
- Python integrators use `from wifi_densepose.client import SensingClient` (ADR-117).
- TypeScript / MCP integrators use `import { SensingClient } from "@ruvnet/rvagent"`.
- MCP-capable AI assistants use `claude mcp add rvagent -- npx @ruvnet/rvagent stdio`.

All three talk to the same sensing-server backend; there is no shared state between the Python and TypeScript clients beyond what the sensing-server itself maintains.

### 10.2 Sensing-server API contract

SENSE-BRIDGE depends on the sensing-server WebSocket protocol documented in `v2/crates/wifi-densepose-sensing-server/src/main.rs` (referenced in `python/wifi_densepose/client/ws.py:6-13`). The three message types (`connection_established`, `pose_data`, `edge_vitals`) are stable across v0.7.x releases. If the sensing-server adds new message types, SENSE-BRIDGE follows the same pattern as the Python client: unknown `type` values yield a plain `SensingMessage` rather than an error, ensuring forward compatibility.

### 10.3 MCP protocol version

SENSE-BRIDGE targets MCP protocol version `2025-06-18` (current stable). It will include backwards compatibility with `2025-03-26` (Streamable HTTP without session management) and optionally `2024-11-05` (legacy SSE via `--legacy-sse` flag). Protocol version `2025-06-18` requires the `MCP-Protocol-Version` header on HTTP requests; SENSE-BRIDGE validates this per spec.

### 10.4 Node.js version

Minimum Node.js 20 LTS. Node 22 is supported and recommended for production (active LTS as of 2026). The `ruvector` napi-rs bindings must be confirmed compatible with both (Q2). Node 18 is EOL and explicitly not supported.

### 10.5 MQTT broker compatibility

SENSE-BRIDGE uses `mqtt.js ^5` which implements MQTT 3.1.1 and MQTT 5.0. The `mosquitto` local broker (CLAUDE.local.md §Local mosquitto) and cognitum-v0's MQTT stack (CLAUDE.local.md fleet table) are both compatible. TLS mode is optional via `RUVIEW_MQTT_TLS=1` env var.

---

## 11. Consequences

### 11.1 Positive consequences

- Any MCP-capable AI assistant can query RuView presence, vitals, pose, and BFLD data with zero custom integration code after `claude mcp add rvagent`.
- ruflo multi-agent swarms gain first-class access to real-world sensing data, enabling swarms to gate decisions on physical events (fall detected → page caregiver workflow).
- The TypeScript surface provides a second reference implementation of the sensing-server client protocol alongside the Python client (ADR-117), validating the protocol design against two independent consumers.
- The ruvector HNSW integration enables cross-node person re-identification entirely within the rvagent process — no additional network calls between sensing nodes.

### 11.2 Negative consequences / risks

| Risk | Likelihood | Severity | Mitigation |
|---|---|---|---|
| **ruvector napi-rs not building on Windows** | Medium | Medium | Confirm in P1 CI; if binaries not prebuilt, document requirement of Rust toolchain on Windows |
| **MCP protocol churn** — spec updated twice in 2025; another update in 2026 possible | Medium | Low | Pin `@modelcontextprotocol/sdk` to a minor range; wrap SDK calls behind an internal `transport.ts` abstraction so changes are isolated |
| **Subscription lifecycle bugs** — zombie subscriptions if session cleanup is missed | High | Medium | Implement per-session resource registry with TTL; all subscriptions auto-expire after `duration_s` even if session is not explicitly deleted |
| **sensing-server WS disconnect** — stdio process dies if not reconnecting | Low | High | Implement exponential back-off reconnect in `ws-client.ts`; emit `{ error: "RECONNECTING" }` tool responses during gap |
| **npm name collision** — `rvagent` taken by another publisher before P5 | Low | Medium | Publish `@ruvnet/rvagent` scoped; use that name throughout |
| **ruflo plugin manifest incompatibility** — format not publicly specced | Medium | Medium | Confirm format in P5 preparation; use the minimal required fields only |
| **Sensing-tool surface becomes a surveillance API** — "who is in the room" is a privacy-charged primitive | High | High | RUVIEW-POLICY layer (§4.1a) gates every sensing call; default-deny for biometric tools; redaction applied server-side so agents cannot opt out |

### 11.3 Strategic implication: ambient-sensing normalization layer

The MCP tool catalog in §4 is RuView-WiFi-CSI-specific today. The shape of the catalog — `presence.now`, `vitals.get_*`, `pose.latest`, `primitives.*`, `bfld.*` — is **modality-agnostic at the semantic layer**: the same tools could be backed by any sensing modality that produces the same questions.

If the project later adds BLE, mmWave (e.g. the ESP32-C6 + Seeed MR60BHA2 already on COM4 per CLAUDE.md), LiDAR, thermal, camera, radar, or UWB inputs, the rvagent MCP surface stays the same. Only the source-multiplexer behind `cache.ts` changes — it now ingests from multiple modalities and resolves conflicts (e.g. WiFi CSI says "presence: true" but mmWave says "presence: false" → fusion policy decides; this is the kind of decision the RUVIEW-POLICY layer can also gate).

This positions the npm package not as "a WiFi client" but as the **semantic-environment API**: agents ask "is anyone here?" without caring which radio answered. The competitive landscape (Aqara FP2, ESPHome LD2410) exposes raw telemetry; SENSE-BRIDGE exposes environmental cognition.

The follow-on ADR (call it ADR-13x — RUVIEW-FUSION) would formalize the per-modality adapter contract. It is intentionally out of scope for ADR-124 — this ADR ships the WiFi-CSI path only — but the tool catalog and policy layer are designed to absorb additional modalities without API churn.

---

## 12. Acceptance criteria

The following must all pass before ADR-124 is considered Accepted:

- [ ] `npm install @ruvnet/rvagent` succeeds on Node 20/22, linux/x86_64, macos/arm64, windows/amd64 with no Rust toolchain required (ruvector prebuilts must ship).
- [ ] `npx @ruvnet/rvagent stdio` starts and responds to a `tools/list` JSON-RPC request with the 15 tools from §4.1.
- [ ] `npx @ruvnet/rvagent serve --port 3100` starts; `curl -X POST http://localhost:3100/mcp -H "Content-Type: application/json" -d '{"jsonrpc":"2.0","method":"tools/list","id":1}'` returns the tool list.
- [ ] `ruview.vitals.get_all` with a running `sensing-server --mock-frames` returns `breathing_rate_bpm` and `heartrate_bpm` values within 5 seconds.
- [ ] `ruview.vector.store_pose` followed by `ruview.vector.search_pose` with the same embedding returns the stored pose as the top-1 match.
- [ ] `claude mcp add rvagent -- npx @ruvnet/rvagent stdio` followed by `/mcp` in a Claude Code session shows the rvagent tools listed.
- [ ] All MCP tool input schemas are validated via Zod; an invalid input returns an MCP `INVALID_PARAMS` error, not an unhandled exception.
- [ ] TypeScript strict-mode compilation (`tsc --noEmit`) passes with zero errors.
- [ ] `npm run build` produces both ESM (`dist/esm/`) and CJS (`dist/cjs/`) outputs with `.d.ts` type declarations.
- [ ] The published npm tarball size is `≤ 10 MB` including the ruvector napi-rs binary for the current platform.

---

## 13. References

### This repo

- `python/wifi_densepose/client/ws.py` — WebSocket client (ADR-117 P4): connection protocol, message types `connection_established`, `pose_data`, `edge_vitals`
- `python/wifi_densepose/client/mqtt.py` — MQTT client (ADR-117 P4): topic namespaces, wildcard matching
- `python/wifi_densepose/client/primitives.py` — Semantic primitive enum and listener (ADR-117 P4): 10 ADR-115 primitives
- `v2/crates/wifi-densepose-sensing-server/src/main.rs` — Axum server: REST API, WebSocket endpoint `/ws/sensing`
- `v2/crates/wifi-densepose-sensing-server/src/bearer_auth.rs` — Bearer token auth pattern for HTTP server
- `v2/crates/wifi-densepose-sensing-server/src/semantic/` — 10 semantic primitive modules
- `v2/crates/wifi-densepose-sensing-server/src/mqtt/` — MQTT publisher, discovery, topic routing
- `docs/adr/ADR-055-integrated-sensing-server.md` — Sensing-server architectural context
- `docs/adr/ADR-095-rvcsi-edge-rf-sensing-platform.md` — rvCSI edge runtime
- `docs/adr/ADR-115-home-assistant-integration.md` — MQTT topic structure, 10 semantic primitives, 21 HA entities
- `docs/adr/ADR-117-pip-wifi-densepose-modernization.md` — PIP-PHOENIX: Python client and PyO3 bindings (the Python-runtime parallel to this ADR)
- `docs/adr/ADR-118-bfld-beamforming-feedback-layer-for-detection.md` — BFLD crate: `BfldEvent` MQTT topics
- `docs/adr/ADR-024-contrastive-csi-embedding-model.md` — AETHER person re-ID embeddings
- `docs/adr/ADR-016-ruvector-integration.md` — RuVector integration in the Rust workspace
- `CLAUDE.md` — Project config: 3-tier model routing (ADR-026), ruflo MCP tools, `mcp__claude-flow__*` namespace
- `CLAUDE.local.md` — Fleet table: Tailscale hosts, cognitum-v0 services table, local mosquitto pattern

### External

- [Model Context Protocol specification 2025-06-18](https://modelcontextprotocol.io/specification/2025-06-18/basic/transports) — Transports: stdio and Streamable HTTP
- [MCP TypeScript SDK — github.com/modelcontextprotocol/typescript-sdk](https://github.com/modelcontextprotocol/typescript-sdk) — `Server`, `StdioServerTransport`, `StreamableHTTPServerTransport`
- [@modelcontextprotocol/sdk on npm](https://www.npmjs.com/package/@modelcontextprotocol/sdk)
- [ruvector on npm](https://www.npmjs.com/package/ruvector) — v0.2.25, napi-rs HNSW vector DB
- [ruvnet npm profile](https://www.npmjs.com/~ruvnet) — confirms `@ruvnet` scope ownership
- [RuVector GitHub](https://github.com/ruvnet/ruvector) — Rust source + napi-rs node bindings
- [ruflo (claude-flow) GitHub](https://github.com/ruvnet/ruflo) — ruflo plugin manifest convention, `v3/` structure
- [ruflo issue #1689](https://github.com/ruvnet/ruflo/issues/1689) — documents existing rvagent WASM exports (`callMcp`, `executeTool`, `listTools`) and distinguishes them from this ADR's server-side rvagent
- [Why MCP Deprecated SSE — fka.dev](https://blog.fka.dev/blog/2025-06-06-why-mcp-deprecated-sse-and-go-with-streamable-http/) — rationale for Streamable HTTP over legacy SSE
- [MCP TypeScript SDK dual-transport patterns — dev.to](https://dev.to/zoricic/understanding-mcp-server-transports-stdio-sse-and-http-streamable-5b1p)
