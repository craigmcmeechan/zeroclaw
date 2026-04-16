# Plan of Attack: Embedded gRPC Memory Daemon → Parity → Claude Code

> **Sequence**: Embed gRPC in ZeroClaw daemon → prove parity → then multi-agent.

---

## Guiding Principle

Embed the gRPC memory service **inside the ZeroClaw daemon process** as a tokio task — not a separate binary or Docker container. The root ZeroClaw instance already owns the SQLite databases; the gRPC endpoint simply exposes them to child/sandboxed agents over the network. This eliminates an entire deployment artifact, schema drift concerns, and config synchronization — while preserving every RPC design from the original `PLAN-sqlite-grpc-memory-service_3.md`.

The Claude Code PTY integration (documented in `devdocs/sandboxed-coding-agent.md`) depends on reliable multi-agent memory — this is the foundation.

### What changed (vs. the standalone-binary approach)

| Standalone (old) | Embedded daemon (new) |
|---|---|
| Separate `crates/memory-server/` binary | `src/gateway/grpc.rs` module inside ZeroClaw |
| Own Dockerfile, own Docker image (~8–15 MB) | No extra image — gRPC rides the existing daemon |
| `_catalog.db` + `_system.db` managed by server | Root instance's existing `brain.db`, `sessions.db`, etc. used directly |
| ConfigService for identity/provider/tool sync | Eliminated — root reads `config.toml` natively |
| Schema drift risk (server SQL vs ZeroClaw SQL) | Zero — same process creates and queries the DBs |
| `memory-schema` crate for parity | Optional nice-to-have, not a correctness requirement |
| DB Manager with `HashMap<name, RwLock<Connection>>` | Root's `Arc<dyn Memory>` + per-child ephemeral DBs |
| Separate deployment, monitoring, lifecycle | One process, one set of logs, one health check |

---

## Phase 0: Embedded gRPC Server in ZeroClaw Daemon

**Goal**: The ZeroClaw daemon exposes a gRPC endpoint (default `:50051`) that child/sandboxed agents can connect to for memory operations. The server runs as a tokio task inside the existing daemon process.

**Scope**: Protobuf definitions, `MemoryGrpcServer` service implementation proxying to the root's `Arc<dyn Memory>`, lightweight catalog for child agent DBs, access control. No separate binary, no Docker image, no ConfigService.

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│   ZeroClaw daemon (root instance)                       │
│                                                         │
│   ┌──────────┐  ┌──────────┐  ┌───────────────────┐    │
│   │ Gateway   │  │ Channels │  │ gRPC Memory       │    │
│   │ (HTTP)    │  │ (Tg/Disc)│  │ Server (:50051)   │    │
│   │ :3000     │  │          │  │                    │    │
│   └──────────┘  └──────────┘  │ ┌───────────────┐  │    │
│                               │ │MemoryService  │  │    │
│   ┌──────────────────────┐    │ │  → root Memory│  │    │
│   │ Root Memory           │◄──┤ ├───────────────┤  │    │
│   │ (SqliteMemory)        │   │ │CatalogService │  │    │
│   │   brain.db            │   │ │  → child DBs  │  │    │
│   │   knowledge_graph.db  │   │ └───────────────┘  │    │
│   │   sessions.db         │   └───────────────────┘    │
│   │   etc.                │         ▲                   │
│   └──────────────────────┘         │ gRPC               │
│                                    │                     │
└────────────────────────────────────┼─────────────────────┘
                                     │
              ┌──────────────────────┤
              │                      │
    ┌─────────┴────────┐   ┌────────┴─────────┐
    │ Docker: Claude    │   │ Docker: Claude    │
    │ Code Agent #1     │   │ Code Agent #2     │
    │                   │   │                   │
    │ GrpcMemory client │   │ GrpcMemory client │
    │ → ephemeral DB    │   │ → ephemeral DB    │
    └───────────────────┘   └───────────────────┘
```

### How it works

1. **Daemon startup** (`src/daemon/mod.rs`): If `config.grpc.enabled`, spawn `grpc::run_grpc_server()` as a new component supervisor alongside gateway, channels, heartbeat, scheduler.
2. **Root Memory proxy**: The gRPC server holds `Arc<dyn Memory>` — the same instance the root agent uses. MemoryService RPCs proxy to trait methods. Zero schema drift because it's the same object.
3. **Child agent DBs**: CatalogService can provision ephemeral `SqliteMemory` instances for sandboxed agents. Stored in `HashMap<String, Arc<SqliteMemory>>` with RwLock.
4. **Access control**: Simple agent-ID + token grant model. Root trusts itself; child agents get tokens at provision time.
5. **Cleanup**: Ephemeral DBs get reaped on inactivity (Touch heartbeat pattern from original plan).

### Tasks

| # | Task | Notes |
|---|------|-------|
| 0.1 | Add `tonic` + `prost` + `tonic-build` to `Cargo.toml` (optional, behind `grpc-memory` feature) | `prost` already exists as optional dep |
| 0.2 | Define `proto/catalog.proto` and `proto/memory.proto` | Subset of original plan — no `config.proto`, no `cloudflare.proto` |
| 0.3 | Create `src/gateway/grpc.rs` — `MemoryGrpcServer` struct | Holds `Arc<dyn Memory>` (root), `RwLock<HashMap<String, Arc<SqliteMemory>>>` (children), access grants |
| 0.4 | Implement MemoryService RPCs proxying to `dyn Memory` trait | `store→store()`, `recall→recall()`, `get→get()`, `list→list()`, `forget→forget()`, `purge_namespace→purge_namespace()`, etc. |
| 0.5 | Implement MemoryService vector/search RPCs | `VectorSearch`, `TextSearch`, `HybridSearch` — proxy to `recall()` with appropriate `SearchMode` |
| 0.6 | Implement MemoryService: `VectorUpsert`, `BulkUpsert` | `store_with_metadata()` path |
| 0.7 | Implement CatalogService: `Provision`, `Discover`, `DropDatabase`, `Touch` | Lightweight — manages ephemeral child DBs only, not root's DBs |
| 0.8 | Implement CatalogService: `GrantAccess`, `RevokeAccess`, `WhoHasAccess` | Token-based; root agent auto-granted |
| 0.9 | Ephemeral DB reaper: tokio interval task | TTL-based cleanup, `Touch` heartbeat extends TTL |
| 0.10 | Add `GrpcConfig` to `src/config/schema.rs` | `enabled`, `listen_addr`, `port`, `max_child_dbs`, `ephemeral_ttl_secs` |
| 0.11 | Wire into daemon startup in `src/daemon/mod.rs` | `spawn_component_supervisor("grpc", grpc::run_grpc_server(...))` |
| 0.12 | Integration tests: provision child DB → store → search → drop | Test from multiple concurrent tokio tasks acting as gRPC clients |

### What's eliminated (vs. standalone)

| Item | Why it's gone |
|---|---|
| `crates/memory-server/` binary | Server lives inside ZeroClaw binary |
| Dockerfile for memory-server | No separate container |
| `_catalog.db` | Child DB registry is in-memory `HashMap` (ephemeral by design) |
| `_system.db` + ConfigService | Root reads config.toml directly — no sync needed |
| `config.proto` | Eliminated entirely |
| `cloudflare.proto` | Was already deferred; now doubly irrelevant |
| Startup reconciliation (scan orphan DBs) | Root creates its own DBs; children are ephemeral / in-memory tracked |
| TOML import/export RPCs | Root uses config.toml natively |
| Schema drift risk | Same process, same `SqliteMemory` constructor, same inline DDL |

### Definition of Done — Phase 0

- `cargo test` passes with `grpc-memory` feature enabled
- `zeroclaw daemon` starts gRPC server on configured port when `grpc.enabled = true`
- Can provision a child DB, store memories, search, drop — all via gRPC
- Root Memory proxying works (store via gRPC → recall via gRPC returns same data)
- 10 concurrent gRPC clients exercised in integration test
- Feature-gated: no compilation impact when `grpc-memory` is disabled

---

## Phase 0.5: Schema Extraction (Optional — Nice-to-Have)

**Goal**: Single source of truth for database schemas via a `memory-schema` shared crate.

**Status**: **Deferred / optional**. With the embedded-daemon approach, schema drift is eliminated — the same process that creates `brain.db` also serves it over gRPC. The gRPC server never materializes its own copy of the schema; it proxies to `SqliteMemory` which owns the DDL.

This phase becomes valuable if/when:
- A **standalone** gRPC server is needed (multi-host deployment)
- Schema versioning is desired for operational auditing
- The inline DDL scattered across 10 `.rs` files becomes a maintenance burden on its own

### The Problem (still real, just not blocking)

Schema SQL is defined inline across **10 separate `.rs` files**, each creating their own database on startup:

| File | Database | Tables | FTS5 | Triggers | Inline Migrations |
|---|---|---|---|---|---|
| `src/memory/sqlite.rs` | `brain.db` | 2 | 1 | 3 | 4 `ALTER TABLE ADD COLUMN` |
| `src/memory/snapshot.rs` | `brain.db` (duplicate) | 2 | 1 | 0 | — |
| `src/memory/knowledge_graph.rs` | `knowledge_graph.db` | 2 | 1 | 3 | — |
| `src/memory/audit.rs` | `audit.db` | 1 | 0 | 0 | — |
| `src/memory/response_cache.rs` | `response_cache.db` | 1 | 0 | 0 | — |
| `src/channels/session_sqlite.rs` | `sessions.db` | 2 | 1 | 2 | 4 `ALTER TABLE ADD COLUMN` |
| `src/channels/whatsapp_storage.rs` | custom | 13 | 0 | 0 | — |
| `src/gateway/api_pairing.rs` | `devices.db` | 1 | 0 | 0 | — |
| `src/cron/store.rs` | `cron/jobs.db` | 2 | 0 | 0 | dynamic ALTERs |
| `src/heartbeat/store.rs` | `heartbeat/history.db` | 1 | 0 | 0 | — |

This is tech debt worth addressing independently, but it no longer blocks the gRPC work.

### If pursued later

The solution from the original plan remains valid: `crates/memory-schema/` crate with versioned `.sql` migration files, `rust-embed` bundling, `SchemaTemplate` enum, migration runner, parity gate test. See the original `PLAN-sqlite-grpc-memory-service_3.md` for full details.

---

## Phase 1: `GrpcMemory` Client Backend

**Goal**: `backend = "grpc"` in `config.toml` routes all Memory trait calls to the embedded gRPC server. This is what child/sandboxed agents use.

> **Key distinction**: The root ZeroClaw instance runs the gRPC **server** (Phase 0) and uses `SqliteMemory` directly. Child agents in Docker containers run with `backend = "grpc"` and use `GrpcMemory` to connect back to the root's gRPC endpoint. The root never uses `GrpcMemory` against itself.

### 1.1 Config Schema Changes

Add to `src/config/schema.rs`:

```rust
/// gRPC server configuration (embedded in daemon).
/// Controls whether the daemon exposes memory over gRPC.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct GrpcServerConfig {
    /// Whether to start the gRPC memory server. Default: false.
    #[serde(default)]
    pub enabled: bool,
    /// Listen address. Default: "0.0.0.0".
    #[serde(default = "default_grpc_host")]
    pub host: String,
    /// Listen port. Default: 50051.
    #[serde(default = "default_grpc_port")]
    pub port: u16,
    /// Max ephemeral child DBs before refusing new provisions.
    #[serde(default = "default_max_child_dbs")]
    pub max_child_dbs: usize,
    /// TTL in seconds for ephemeral databases. Default: 86400 (24h).
    #[serde(default = "default_ephemeral_ttl")]
    pub ephemeral_ttl_secs: u64,
}

/// gRPC client configuration (used by child agents).
/// Used when `[memory].backend = "grpc"`.
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct GrpcMemoryClientConfig {
    /// gRPC server URL (e.g. "http://host.docker.internal:50051").
    #[serde(default)]
    pub url: Option<String>,
    /// Database name (provisioned via CatalogService).
    /// Defaults to agent identity name.
    #[serde(default)]
    pub database: Option<String>,
    /// Agent identity for access grants.
    #[serde(default)]
    pub agent_id: Option<String>,
    /// Auto-provision DB on first connect. Default: true.
    #[serde(default = "default_true")]
    pub auto_provision: bool,
    /// Whether the DB is ephemeral. Default: true for child agents.
    #[serde(default = "default_true")]
    pub ephemeral: bool,
    /// Embedding mode: "client" or "server". Default: "client".
    #[serde(default = "default_embedding_mode_client")]
    pub embedding_mode: String,
}
```

Add `grpc_server: GrpcServerConfig` to top-level config (alongside gateway). Add `grpc: GrpcMemoryClientConfig` to `MemoryConfig`.

### 1.2 `GrpcMemory` Struct + `impl Memory`

New file: `src/memory/grpc.rs` (~250–350 lines)

```rust
pub struct GrpcMemory {
    catalog_client: CatalogServiceClient<Channel>,
    memory_client: MemoryServiceClient<Channel>,
    db_name: String,
    agent_id: String,
    embedder: Option<Arc<dyn EmbeddingProvider>>,
    heartbeat_handle: Option<JoinHandle<()>>,
}
```

**Trait method mapping** (unchanged from original — the RPCs are the same):

| Memory trait method | gRPC call |
|---|---|
| `store()` | MemoryService `Store` (or `Execute` INSERT) |
| `recall()` | `HybridSearch` / `TextSearch` / `VectorSearch` per search_mode |
| `get()` | `Query` (SELECT WHERE key = ?) |
| `list()` | `Query` (SELECT with filters) |
| `forget()` | `Execute` (DELETE WHERE key = ?) |
| `purge_namespace()` | `Execute` (DELETE WHERE namespace = ?) |
| `purge_session()` | `Execute` (DELETE WHERE session_id = ?) |
| `count()` | `Query` (SELECT COUNT(*)) |
| `health_check()` | gRPC channel connectivity check |
| `store_with_metadata()` | `VectorUpsert` |
| `recall_namespaced()` | `HybridSearch` with namespace filter |
| `store_procedural()` | `Execute` (INSERT with procedural category) |
| `export()` | `Query` with filter params |

**Constructor flow**:
1. Connect to gRPC server at configured URL
2. `CatalogService::Discover(agent_id)` — check for existing DBs
3. If `auto_provision` and no DB found: `CatalogService::Provision(db_name, ephemeral=true)`
4. `CatalogService::GrantAccess(db_name, agent_id)`
5. If ephemeral: spawn background `Touch` heartbeat task
6. Return `GrpcMemory`

### 1.3 Factory Wiring

In `src/memory/mod.rs` → `create_memory_with_storage_and_routes`:

```rust
"grpc" => {
    // Skip local hygiene/snapshot/hydration — root handles its own DBs
    let grpc_mem = grpc::GrpcMemory::connect(&config.grpc, embedder).await?;
    return Ok(Box::new(grpc_mem));
}
```

### 1.4 Backend Registration

In `src/memory/backend.rs`:

```rust
const GRPC_PROFILE: MemoryBackendProfile = MemoryBackendProfile {
    key: "grpc",
    label: "gRPC — connects to ZeroClaw root daemon's memory service",
    auto_save_default: true,
    uses_sqlite_hygiene: false,
    sqlite_based: false,
    optional_dependency: false,
};
```

Add to `MemoryBackendKind` enum, `classify_memory_backend()`, `memory_backend_profile()`, `SELECTABLE_MEMORY_BACKENDS`.

### 1.5 Feature Flag

Gate both server and client behind `grpc-memory` feature in `Cargo.toml`:

```toml
[features]
grpc-memory = ["dep:tonic", "dep:prost"]

[build-dependencies]
tonic-build = { version = "0.14", optional = true }
```

### Definition of Done — Phase 1

- `GrpcMemory` connects to the embedded gRPC server from Phase 0
- All Memory trait methods proxy correctly
- Child agent in Docker can `zeroclaw onboard --memory grpc --quick` and get working config
- Ephemeral DB heartbeat keeps DB alive during sessions
- No compilation impact when `grpc-memory` feature is disabled

---

## Phase 2: Adapt Memory-Adjacent Systems

**Goal**: Every system that touches memory works correctly when a child agent uses `GrpcMemory`.

> **Scope reduction**: With the embedded approach, the root instance's adjacent systems (hygiene, snapshot, consolidation, etc.) are unaffected — they continue using `SqliteMemory` directly. Phase 2 only concerns what happens in child agents using `GrpcMemory`.

### 2.1 Systems That Already Work (no changes needed)

These systems interact with Memory purely through the trait interface. If a child agent uses `GrpcMemory`, these all work transparently:

| System | Why it works |
|---|---|
| **RetrievalPipeline** | Wraps `Arc<dyn Memory>`, calls `recall()` / `recall_namespaced()` — trait-only |
| **Consolidation** | Calls `store()` / `store_procedural()` — trait-only |
| **Conflict Resolution** | Calls `recall()` for similarity search — trait-only |
| **AuditedMemory** | Decorator wraps any `Memory` impl; its local `audit.db` stays local |
| **Decay/Importance/Vector/Chunker** | Pure functions — no Memory calls |

### 2.2 Systems That Need Conditional Behavior

| System | Issue | Fix |
|---|---|---|
| **Snapshot export** | Opens `brain.db` directly | In child agents: use `memory.export()` trait method → gRPC `Query`. Root is unaffected (still uses local `brain.db`). |
| **Snapshot hydration** | Opens `brain.db` directly, INSERTs from markdown | In child agents: skip entirely. The child's ephemeral DB is server-managed; there's no "cold boot" scenario. |
| **Hygiene** | Opens `brain.db` directly for DELETE | In child agents: skip local hygiene. Root handles its own. Child DBs are ephemeral with TTL-based reaper. |
| **KnowledgeGraph** | Own SQLite connection, not behind Memory trait | Stays local for now. Child agents that need KG would need their own local instance or a future `KnowledgeGraphService`. |

### 2.3 Factory Side-Effect Cleanup

Refactor `create_memory_with_storage_and_routes` to skip local-only operations for gRPC:

```rust
let backend_kind = classify_memory_backend(&backend_name);

if !matches!(backend_kind, MemoryBackendKind::Grpc) {
    // Local-only maintenance
    if let Err(e) = hygiene::run_if_due(config, workspace_dir) { ... }
    if config.snapshot_enabled { snapshot::export_snapshot(...) }
    if config.auto_hydrate { snapshot::hydrate_from_snapshot(...) }
}
```

### Definition of Done — Phase 2

- Child agent with `backend = "grpc"` doesn't crash on missing `brain.db`
- Factory correctly skips hygiene/snapshot for gRPC backend
- Root instance behavior completely unchanged
- KnowledgeGraph documented as "local-only for now"

---

## Phase 3: Parity Testing

**Goal**: Prove that `GrpcMemory` (client) → embedded gRPC server → `SqliteMemory` produces identical behavior to using `SqliteMemory` directly.

> **Simpler than standalone**: Since the embedded server proxies to the real `SqliteMemory`, we're mostly validating the RPC serialization round-trip — not a separate re-implementation of SQLite logic. The risk of behavioral divergence is inherently lower.

### 3.1 Test Infrastructure

| Item | Detail |
|---|---|
| In-process gRPC server | Spawn embedded server on ephemeral port within test, no Docker needed |
| `tests/integration/memory_grpc.rs` | Integration tests using `GrpcMemory` against in-process server |
| Trait-level test harness | Parameterized tests: same assertions against `SqliteMemory` AND `GrpcMemory` |

### 3.2 Test Matrix

Every test passes for **both** `SqliteMemory` (direct) and `GrpcMemory` (via embedded server):

| Test Area | What to Validate |
|---|---|
| **Basic CRUD** | store → get → list → forget → count |
| **FTS5 / Text Search** | Store entries, recall by keyword, verify BM25 ranking order |
| **Vector Search** | Store with embeddings, recall by semantic similarity, verify cosine ordering |
| **Hybrid Search** | Store entries, recall with hybrid mode, verify RRF fusion ranking |
| **Namespace Isolation** | Store in namespace A and B, `recall_namespaced` sees only its own |
| **Session Scoping** | Store with session_id, recall with session filter, `purge_session` |
| **Category Filtering** | Store Core/Daily/Conversation, list with category filter |
| **Time Range Filtering** | Store entries at different timestamps, recall with since/until |
| **Importance Scoring** | Store with importance, verify retrieval respects weighting |
| **Export / GDPR** | Store entries, export with filters, verify completeness |
| **Purge Operations** | `purge_namespace`, `purge_session` — verify correct deletion counts |
| **Concurrent Access** | 5+ tokio tasks storing/recalling simultaneously through gRPC |
| **Ephemeral DB lifecycle** | Provision → use → Touch → let expire → verify reaped |
| **Access control** | Requests without valid grant are rejected |

### 3.3 Existing Tests to Extend

| Existing Test | Adaptation |
|---|---|
| `battle_tests.rs` | Parameterize with `GrpcMemory` variant (refactor `connection()` calls to trait methods) |
| `tests/integration/memory_comparison.rs` | Add gRPC to comparison matrix |
| `tests/integration/memory_loop_continuity.rs` | Run against gRPC backend |

### 3.4 Performance Baseline

Capture latency for key operations (direct SqliteMemory vs. GrpcMemory via loopback):
- `store()` — single entry
- `recall()` — FTS, vector, hybrid (100 / 1000 / 10000 entries)
- Concurrent 10-client recall storm

Expected overhead: <1ms per RPC on loopback. This is a sanity check, not optimization.

### Definition of Done — Phase 3

- Full test matrix passes for both SqliteMemory and GrpcMemory
- Zero behavior differences (same ranking, same filter results, same deletion counts)
- Performance baseline documented (latency per operation)
- Tests run in CI with `--features grpc-memory` (no Docker dependency — in-process server)

---

## Phase 4: Onboarding & Docs

**Goal**: End-to-end usability for both the gRPC server (root daemon) and gRPC client (child agents).

### Tasks

| # | Task |
|---|---|
| 4.1 | `dev/config.template.toml`: add `[grpc_server]` section with comments |
| 4.2 | Child agent config template: minimal `config.toml` with `backend = "grpc"` and `[memory.grpc]` section |
| 4.3 | `docker-compose.yml`: example service definition for child agent container connecting to root |
| 4.4 | `docs/`: memory backend comparison page (sqlite vs grpc vs qdrant) |
| 4.5 | `architecture/memory/README.md`: update with embedded gRPC diagram |
| 4.6 | Health check: `zeroclaw status` reports gRPC server status (listening, connected clients, child DBs) |
| 4.7 | Onboarding wizard: `--memory grpc` option with URL/agent-id prompts (for child agents) |

### Definition of Done — Phase 4

- Root daemon config documented: how to enable `[grpc_server]`
- Child agent can be provisioned with minimal config
- `zeroclaw status` shows gRPC server metrics
- Docs cover when to use gRPC vs SQLite

---

## Phase 5: Claude Code / Sandboxed Coding Agent

**Deferred — only begins after Phase 3 is green.**

This is the work documented in `devdocs/sandboxed-coding-agent.md` and `devdocs/implementation-plan.md`. With the embedded gRPC memory service proven, Docker-sandboxed Claude Code agents connect to the root's gRPC endpoint. Each agent gets an ephemeral DB via CatalogService, uses the Touch heartbeat to stay alive, and the reaper cleans up after sessions end.

The embedded approach makes this simpler:
- No separate `memory-server` container to orchestrate
- Child agent Docker image only needs: Claude Code CLI + ZeroClaw binary (with `grpc-memory` feature)
- Root daemon manages everything from one process

---

## Dependency Graph

```
Phase 0: Embedded gRPC server in ZeroClaw daemon
    │
    ├──► Phase 1: GrpcMemory client backend
    │       │
    │       ├──► Phase 2: Adapt factory/snapshot/hygiene for child agents
    │       │
    │       ▼
    │    Phase 3: Parity testing (blocks everything below)
    │       │
    │       ├──► Phase 4: Onboarding & docs
    │       │
    │       ▼
    │    Phase 5: Sandboxed coding agents
    │
    └──► Phase 0.5: Schema extraction (optional, independent)
```

> **Phase 0 and 1 are tightly coupled** — the server needs a client to test against, and vice versa. Develop them together. Phase 0.5 (schema extraction) is independently valuable but not blocking.

---

## Blind Spots & Mitigations

| Blind Spot | Mitigation |
|---|---|
| Snapshot.rs bypasses Memory trait (direct `brain.db`) | Root unaffected; child agents skip snapshot (ephemeral) — Phase 2.2 |
| Hygiene opens `brain.db` directly | Root unaffected; child agents skip hygiene — Phase 2.3 |
| KnowledgeGraph has own SQLite connection | Stays local. Child agents needing KG is a future concern. |
| AuditedMemory writes local `audit.db` | Stays local per-agent (fine — audit is inherently per-instance) |
| Embedding responsibility (who computes vectors) | `embedding_mode` in client config: "client" (default) or "server" — Phase 1.1 |
| Factory runs hygiene/snapshot before backend construction | Conditional skip for gRPC — Phase 2.3 |
| `battle_tests.rs` calls `SqliteMemory::connection()` directly | Refactor 3 places to trait-level assertions — Phase 3.3 |
| SQLite single-writer contention on root's `brain.db` | gRPC server serializes writes through `Arc<dyn Memory>` → `SqliteMemory` with WAL. Same contention as direct access — not worse. For truly concurrent writes, child agents use their own ephemeral DBs. |
| Root daemon crash takes down gRPC server | Same as today with gateway crash. Component supervisor restarts. Child agents reconnect (tonic has retry). |
| Child agent can't reach gRPC after network partition | `GrpcMemory::health_check()` fails → agent reports unhealthy. Design for graceful degradation in Phase 5. |

---

## What NOT to Do

- **Don't build a standalone `memory-server` binary** — the embedded approach is strictly simpler. If multi-host deployment is ever needed, extract then.
- **Don't build ConfigService** — root reads `config.toml` natively. No `_system.db`.
- **Don't add Cloudflare integration** — deferred. Not relevant to single-host Docker topology.
- **Don't try to make KnowledgeGraph remote** — low usage, independent SQLite, not blocking multi-agent.
- **Don't optimize gRPC latency before Phase 3** — correctness first, then measure.
- **Don't skip parity tests** — this is the gate. Everything downstream depends on it.
- **Don't extract schemas (Phase 0.5) as a prerequisite** — it's nice-to-have tech debt cleanup, not a correctness requirement for the embedded approach.

---

## Estimated Scope

| Phase | Size | LOC Estimate | Notes |
|---|---|---|---|
| Phase 0 (embedded gRPC server) | Medium | ~500–800 | `src/gateway/grpc.rs` + protos + daemon wiring. No separate binary. |
| Phase 0.5 (schema extraction) | Medium | ~500–700 | Optional. Shared crate + SQL files + refactor inline DDL. |
| Phase 1 (GrpcMemory client) | Medium | ~300–400 | `src/memory/grpc.rs` + config + factory wiring |
| Phase 2 (adapt adjacent systems) | Small | ~100–150 | Conditional skips in factory. Minimal code. |
| Phase 3 (parity testing) | Medium | ~600–800 | Test harness + parameterized tests. No Docker infra needed. |
| Phase 4 (onboarding + docs) | Small | ~200 | Config templates, wizard prompts, docs |
| Phase 5 (Claude Code) | Large | Separate plan | Depends on all above |

**Total for Phases 0–4**: ~1700–2350 LOC (down from ~4000–5200 in the standalone approach).

---

## Reference

- `devdocs/PLAN-sqlite-grpc-memory-service_3.md` — Original standalone gRPC service design (~1700 lines). Protobuf definitions and RPC method signatures remain authoritative.
- `devdocs/sandboxed-coding-agent.md` — Claude Code integration design.
- `devdocs/implementation-plan.md` — Sandboxed coding agent phased rollout.
- `src/daemon/mod.rs` — Daemon startup, component supervisors pattern.
- `src/gateway/mod.rs` — Existing HTTP gateway (axum). gRPC server follows same pattern.
- `src/memory/traits.rs` — Memory trait (15+ async methods).
- `src/memory/mod.rs` — Factory function `create_memory_with_storage_and_routes`.
- `src/config/schema.rs` — `MemoryConfig` struct (line ~4842).
