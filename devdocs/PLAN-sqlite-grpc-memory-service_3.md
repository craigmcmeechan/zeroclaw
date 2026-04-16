# SQLite gRPC Memory Service — Minimal Docker Image Plan

## Mission

A single, tiny Docker image that exposes SQLite databases over gRPC so multiple AI agents (powered by [ZeroClaw](https://github.com/zeroclaw-labs/zeroclaw)) can concurrently read/write shared memories using **vector search** (`sqlite-vec`) and **BM25 full-text search** (FTS5). The image also serves as the **system configuration store** for the entire ZeroClaw deployment — identities, souls, skills, tools, providers, agent instances, shared workspaces, and tunnel config — and bundles **cloudflared** for zero-config Cloudflare Tunnel access.

---

## 1. Language Choice: Rust

| Criterion | Rust | Go | Python |
|---|---|---|---|
| Final binary size | ~5–8 MB static | ~10–15 MB static | 100 MB+ runtime |
| Docker image (scratch) | **~8–12 MB** | ~15–20 MB | ~150 MB+ |
| gRPC ecosystem | `tonic` (mature, async) | `grpc-go` (excellent) | `grpcio` (heavy C deps) |
| SQLite binding | `rusqlite` (direct libsqlite3) | `go-sqlite3` (CGO) | `sqlite3` (C ext) |
| Extension loading | Compile sqlite-vec statically in | CGO complicates static builds | pip install, runtime load |
| Concurrency model | tokio async, fine-grained locking | goroutines, but CGO limits | GIL, multiprocess needed |

**Verdict:** Rust with `tonic` + `rusqlite` produces the smallest static binary, links sqlite-vec and FTS5 at compile time, and gives us async concurrency without CGO hassles.

---

## 2. Core Dependencies

| Component | Crate / Library | Purpose |
|---|---|---|
| gRPC server | `tonic` + `prost` | Async gRPC with protobuf codegen |
| Async runtime | `tokio` | Async I/O, task scheduling, process management |
| SQLite | `rusqlite` (bundled feature) | Embeds libsqlite3 with FTS5 enabled |
| Vector search | `sqlite-vec` (C source, compiled in) | KNN vector search via `vec0` virtual tables |
| FTS5 / BM25 | Built into SQLite | `CREATE VIRTUAL TABLE ... USING fts5(...)` |
| Serialization | `prost` | Protobuf message types |
| Connection pool | `r2d2` or manual `Mutex<Connection>` | Safe multi-client access to SQLite |
| Tunnel | `cloudflared` (static Go binary, bundled) | Cloudflare Tunnel sidecar, managed as child process |
| Config serialization | `serde` + `serde_json` / `toml` | Parse ZeroClaw config.toml ↔ _system.db |

---

## 3. Architecture

```
┌──────────────────────────────────────────────────────────────────────┐
│                        Docker Container                              │
│                   (FROM scratch + cloudflared)                        │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │                 Single Static Rust Binary                      │  │
│  │                                                                │  │
│  │  ┌──────────┐  ┌──────────────┐  ┌─────────────┐  ┌────────┐ │  │
│  │  │  tonic   │  │ CatalogSvc   │  │ MemorySvc   │  │ Config │ │  │
│  │  │  gRPC    │──│ (discover,   │──│ (vec, fts,  │──│ Svc    │ │  │
│  │  │  server  │  │  provision)  │  │  hybrid)    │  │        │ │  │
│  │  └──────────┘  └──────┬───────┘  └──────┬──────┘  └───┬────┘ │  │
│  │                       │                 │              │      │  │
│  │             ┌─────────▼─────────────────▼──────────────▼───┐  │  │
│  │             │              DB Manager                      │  │  │
│  │             │       HashMap<name, RwLock<Connection>>       │  │  │
│  │             └─────────┬────────────┬───────────────────────┘  │  │
│  │                       │            │                          │  │
│  │    ┌──────────────────┼────────────┼──────────────────┐       │  │
│  │    ▼                  ▼            ▼                  ▼       │  │
│  │ ┌──────────┐  ┌────────────┐ ┌──────────┐   ┌──────────────┐│  │
│  │ │_system.db│  │_catalog.db │ │ shared   │   │ rt_agent-x_  ││  │
│  │ │(zeroclaw │  │(registry)  │ │ *.db     │   │ (ephemeral)  ││  │
│  │ │ config)  │  │            │ │          │   │              ││  │
│  │ └──────────┘  └────────────┘ └──────────┘   └──────────────┘│  │
│  │                                                              │  │
│  │  ┌──────────────────────────────────────────────────────┐    │  │
│  │  │  Background Tasks (tokio)                            │    │  │
│  │  │  • Reaper: evict expired ephemeral DBs               │    │  │
│  │  │  • Tunnel Manager: spawn/monitor cloudflared child    │    │  │
│  │  └──────────────────────────────────────────────────────┘    │  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │  /usr/bin/cloudflared  (static Go binary, ~35 MB)             │  │
│  │  Managed as child process by Tunnel Manager                   │  │
│  │  Config read from _system.db → written to /tmp/cloudflared.yml│  │
│  └────────────────────────────────────────────────────────────────┘  │
│                                                                      │
│  Volume mount: /data                                                 │
└──────────────────────────────────────────────────────────────────────┘

        ▲  gRPC (port 50051)               ▲  Cloudflare Tunnel
        │                                   │  (public hostname → localhost:50051)
        │                                   │
   ┌────┴─────┐  ┌──────────┐  ┌──────────┐    ┌─────────────────────┐
   │  Agent 1 │  │ Agent 2  │  │ Agent N  │    │ Remote agents via   │
   │  (local) │  │  (local) │  │  (local) │    │ Cloudflare edge     │
   └──────────┘  └──────────┘  └──────────┘    └─────────────────────┘
```

---

## 4. Catalog Database (`_catalog.db`)

The server maintains a special internal database at `/data/_catalog.db` that acts as a **registry of all managed databases**. It boots automatically on first run and is never exposed for raw SQL from clients.

### Why a catalog?

An agent spinning up doesn't inherently know what databases exist, which ones it needs, or whether a runtime scratch DB already exists for it. The catalog solves three problems:

1. **Discovery** — "What embedding stores are available?" → query by role/tag
2. **Provisioning** — "I need a scratch DB for this run" → create ephemeral DB, register it, get a handle back
3. **Lifecycle** — "This agent session ended" → garbage-collect its ephemeral DBs

### Catalog schema

```sql
-- ── Always created at server boot ─────────────────────────

-- Master registry of every database the server manages
CREATE TABLE databases (
    db_name     TEXT PRIMARY KEY,          -- logical name (maps to /data/{db_name}.db)
    purpose     TEXT NOT NULL,             -- human-readable description
    role        TEXT NOT NULL,             -- enum-like: see below
    lifecycle   TEXT NOT NULL DEFAULT 'persistent',  -- 'persistent' | 'ephemeral'
    owner_agent TEXT,                      -- NULL = shared, otherwise agent ID
    capabilities TEXT NOT NULL DEFAULT '[]', -- JSON array: ["vec","fts","raw"]
    schema_version INTEGER NOT NULL DEFAULT 1,
    created_at  TEXT NOT NULL DEFAULT (datetime('now')),
    last_accessed_at TEXT,
    expires_at  TEXT,                      -- NULL = never; set for ephemeral DBs
    metadata    TEXT DEFAULT '{}'          -- arbitrary JSON (model dims, tokenizer, etc.)
);

-- Which agents are authorised to access which databases
CREATE TABLE access_grants (
    db_name     TEXT NOT NULL REFERENCES databases(db_name) ON DELETE CASCADE,
    agent_id    TEXT NOT NULL,
    permission  TEXT NOT NULL DEFAULT 'read',  -- 'read' | 'write' | 'admin'
    granted_at  TEXT NOT NULL DEFAULT (datetime('now')),
    PRIMARY KEY (db_name, agent_id)
);

-- Audit log of database lifecycle events
CREATE TABLE catalog_events (
    id          INTEGER PRIMARY KEY,
    db_name     TEXT NOT NULL,
    event_type  TEXT NOT NULL,             -- 'created' | 'accessed' | 'expired' | 'dropped'
    agent_id    TEXT,
    detail      TEXT,                      -- JSON context
    timestamp   TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Index for fast role-based lookups
CREATE INDEX idx_databases_role ON databases(role);
CREATE INDEX idx_databases_lifecycle ON databases(lifecycle, expires_at);
CREATE INDEX idx_access_grants_agent ON access_grants(agent_id);
```

### Database roles (extensible enum)

| Role | Meaning | Typical lifecycle |
|---|---|---|
| `embedding_store` | Long-lived vector memory (RAG, semantic search) | persistent, shared |
| `conversation_history` | Shared conversation logs across agents | persistent, shared |
| `knowledge_base` | Curated facts, documents, reference material | persistent, shared |
| `agent_runtime` | Scratch space for a single agent session | ephemeral, owned |
| `agent_state` | Persistent per-agent state (preferences, learned config) | persistent, owned |
| `task_queue` | Shared task/job queue between agents | persistent, shared |
| `cache` | Throwaway cache (web results, API responses) | ephemeral, shared |
| `custom` | Anything else — purpose field describes it | either |

### Lifecycle model

```
Agent connects
    │
    ▼
┌─────────────────────────────────────────┐
│  Discover(role="embedding_store")       │──→ Returns existing shared DBs
│  Discover(role="conversation_history")  │──→ Returns existing shared DBs
└─────────────────────────────────────────┘
    │
    ▼
┌─────────────────────────────────────────┐
│  Provision(                             │
│    role="agent_runtime",                │
│    lifecycle="ephemeral",               │──→ Creates new DB, registers in catalog
│    owner="agent-7b3f",                  │    Sets expires_at = now + TTL
│    ttl=3600                             │
│  )                                      │
└─────────────────────────────────────────┘
    │
    ▼
  Agent works against all three DBs
    │
    ▼
┌─────────────────────────────────────────┐
│  Agent disconnects / session ends       │
│                                         │
│  Background reaper:                     │
│    SELECT db_name FROM databases        │
│    WHERE lifecycle='ephemeral'          │──→ Drops DB file, removes catalog entry
│    AND expires_at < datetime('now')     │
└─────────────────────────────────────────┘
```

### Agent startup example (what an agent's init flow looks like)

```
1.  agent → Discover(roles=["embedding_store", "conversation_history"])
    server → { databases: [
        { db_name: "embeddings_1536", role: "embedding_store", capabilities: ["vec"] },
        { db_name: "shared_chat_log", role: "conversation_history", capabilities: ["fts","raw"] }
    ]}

2.  agent → Provision(
        role: "agent_runtime",
        purpose: "Scratch space for task-planning agent session",
        lifecycle: "ephemeral",
        owner_agent: "planner-a1b2",
        ttl_seconds: 7200,
        capabilities: ["raw"]
    )
    server → { db_name: "rt_planner-a1b2_1711540800", expires_at: "..." }

3.  agent → GrantAccess(db_name: "embeddings_1536", agent_id: "planner-a1b2", permission: "read")
    server → OK

4.  Agent now holds handles to three databases and works against all of them.
```

### Catalog is the source of truth for the DB manager

The in-memory `HashMap<String, RwLock<Connection>>` that manages open connections is populated **from the catalog on boot** and updated whenever `Provision` or `DropDatabase` is called. The catalog also drives:

- **Startup recovery** — on restart, scan `/data/*.db`, reconcile with catalog, re-register any orphans
- **Reaper task** — a tokio interval task (e.g. every 60s) that evicts expired ephemeral DBs
- **Access control** — every RPC checks `access_grants` before opening a connection (Phase 5+ auth layer)

---

## 5. System Configuration Database (`_system.db`)

The server maintains a second internal database at `/data/_system.db` that stores **all ZeroClaw deployment configuration**. This replaces scattering config across TOML files, env vars, and markdown files — everything is queryable, versionable, and accessible to every agent via gRPC.

### Why store config in SQLite?

ZeroClaw's `config.toml` is designed for a single-node CLI. When you're running multiple containerised agent instances that share infrastructure, you need a config store that is concurrent-safe, queryable, and centrally managed. `_system.db` is that store. Agents read config over gRPC; operators can still import/export TOML for human editing.

### Schema

```sql
-- ═══════════════════════════════════════════════════════════
--  IDENTITIES & SOULS — who the agents are
-- ═══════════════════════════════════════════════════════════

CREATE TABLE identities (
    id          TEXT PRIMARY KEY,        -- e.g. "default", "research-agent"
    format      TEXT NOT NULL DEFAULT 'openclaw',  -- 'openclaw' | 'aieos'
    name        TEXT NOT NULL,
    description TEXT,
    -- OpenClaw-format markdown fields (stored as text)
    soul_md     TEXT,          -- SOUL.md content: core personality & principles
    agents_md   TEXT,          -- AGENTS.md: session conventions, init rules
    identity_md TEXT,          -- IDENTITY.md: user preferences & role
    user_md     TEXT,          -- USER.md: user context & info
    memory_md   TEXT,          -- MEMORY.md: curated long-term facts
    tools_md    TEXT,          -- TOOLS.md: tool usage guidelines
    heartbeat_md TEXT,         -- HEARTBEAT.md: heartbeat instructions
    bootstrap_md TEXT,         -- BOOTSTRAP.md: bootstrap instructions
    -- AIEOS-format (alternative)
    aieos_json  TEXT,          -- full AIEOS JSON structure (if format='aieos')
    -- Metadata
    is_default  BOOLEAN NOT NULL DEFAULT 0,
    created_at  TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  PROVIDERS — LLM backends
-- ═══════════════════════════════════════════════════════════

CREATE TABLE providers (
    id              TEXT PRIMARY KEY,    -- e.g. "anthropic-main", "ollama-local"
    provider_type   TEXT NOT NULL,       -- 'anthropic' | 'openai' | 'openrouter' | 'ollama' | 'gemini' | ...
    api_key_encrypted TEXT,              -- encrypted at rest (empty for local models)
    api_url         TEXT,                -- custom endpoint URL (NULL = default)
    default_model   TEXT,
    default_temperature REAL DEFAULT 0.7,
    is_default      BOOLEAN NOT NULL DEFAULT 0,
    -- Reliability / failover
    max_retries     INTEGER DEFAULT 3,
    timeout_seconds INTEGER DEFAULT 120,
    failover_to     TEXT,                -- provider ID to fail over to
    -- Auth profile support (OAuth / subscription)
    auth_kind       TEXT,                -- 'api_key' | 'oauth' | 'device_code' | 'setup_token'
    auth_profile    TEXT,                -- profile name for multi-account
    -- Cost tracking
    daily_limit_usd  REAL,
    monthly_limit_usd REAL,
    metadata_json   TEXT DEFAULT '{}',
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Model routing rules
CREATE TABLE model_routes (
    id              INTEGER PRIMARY KEY,
    provider_id     TEXT NOT NULL REFERENCES providers(id) ON DELETE CASCADE,
    pattern         TEXT NOT NULL,       -- query classification pattern
    model           TEXT NOT NULL,       -- model to route to
    priority        INTEGER DEFAULT 0,
    enabled         BOOLEAN DEFAULT 1
);

-- ═══════════════════════════════════════════════════════════
--  SKILLS — bundled, community, and workspace skills
-- ═══════════════════════════════════════════════════════════

CREATE TABLE skills (
    id              TEXT PRIMARY KEY,    -- skill name / slug
    source          TEXT NOT NULL,       -- 'bundled' | 'community' | 'workspace' | 'git'
    source_url      TEXT,                -- git URL or path
    skill_md        TEXT,                -- SKILL.md content
    skill_toml      TEXT,                -- SKILL.toml content (if toml-based)
    enabled         BOOLEAN NOT NULL DEFAULT 1,
    audit_status    TEXT DEFAULT 'pending',  -- 'pending' | 'passed' | 'failed'
    audit_report    TEXT,
    installed_at    TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  TOOLS — tool configurations and permissions
-- ═══════════════════════════════════════════════════════════

CREATE TABLE tools (
    id              TEXT PRIMARY KEY,    -- e.g. 'shell', 'browser', 'jira', 'notion'
    category        TEXT NOT NULL,       -- 'core' | 'web' | 'integration' | 'mcp' | 'hardware'
    enabled         BOOLEAN NOT NULL DEFAULT 1,
    config_json     TEXT DEFAULT '{}',   -- tool-specific config (allowed_domains, tokens, etc.)
    -- MCP-specific
    mcp_server_url  TEXT,
    mcp_server_name TEXT,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  AGENT INSTANCES — Docker container / process tracking
-- ═══════════════════════════════════════════════════════════

CREATE TABLE agent_instances (
    id              TEXT PRIMARY KEY,    -- unique instance ID
    identity_id     TEXT REFERENCES identities(id),
    provider_id     TEXT REFERENCES providers(id),
    container_id    TEXT,                -- Docker container ID (if containerised)
    container_image TEXT,                -- Docker image name:tag
    hostname        TEXT,
    pid             INTEGER,
    -- Runtime config
    runtime_kind    TEXT DEFAULT 'native',  -- 'native' | 'docker'
    autonomy_level  TEXT DEFAULT 'supervised',  -- 'readonly' | 'supervised' | 'full'
    workspace_path  TEXT,
    -- Status
    status          TEXT DEFAULT 'stopped',    -- 'running' | 'stopped' | 'error'
    last_heartbeat  TEXT,
    started_at      TEXT,
    -- Resource limits
    max_actions_per_hour INTEGER,
    max_cost_per_day_usd REAL,
    -- Assigned databases (JSON array of db_names from _catalog.db)
    assigned_dbs    TEXT DEFAULT '[]',
    metadata_json   TEXT DEFAULT '{}',
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  CHANNELS — messaging channel configurations
-- ═══════════════════════════════════════════════════════════

CREATE TABLE channels (
    id              TEXT PRIMARY KEY,    -- e.g. 'telegram-main', 'discord-dev'
    channel_type    TEXT NOT NULL,       -- 'telegram' | 'discord' | 'slack' | 'whatsapp' | 'matrix' | ...
    enabled         BOOLEAN NOT NULL DEFAULT 1,
    config_json     TEXT NOT NULL,       -- channel-specific config (tokens, IDs, etc.)
    allowed_users   TEXT DEFAULT '[]',   -- JSON array of allowed user IDs
    pairing_required BOOLEAN DEFAULT 1,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  SHARED FOLDERS & WORKSPACES
-- ═══════════════════════════════════════════════════════════

CREATE TABLE shared_folders (
    id              TEXT PRIMARY KEY,
    path            TEXT NOT NULL UNIQUE, -- host path or volume mount
    purpose         TEXT,                 -- human-readable description
    mount_mode      TEXT DEFAULT 'rw',    -- 'ro' | 'rw'
    -- Which agents can access
    access_mode     TEXT DEFAULT 'shared', -- 'shared' | 'restricted'
    allowed_agents  TEXT DEFAULT '[]',     -- JSON array of agent instance IDs (if restricted)
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE TABLE workspaces (
    id              TEXT PRIMARY KEY,
    name            TEXT NOT NULL,
    path            TEXT NOT NULL,        -- workspace root directory
    identity_id     TEXT REFERENCES identities(id),
    is_active       BOOLEAN DEFAULT 0,
    -- Sandbox config
    workspace_only  BOOLEAN DEFAULT 1,
    allowed_commands TEXT DEFAULT '["git","npm","cargo","ls","cat","grep"]',
    forbidden_paths TEXT DEFAULT '["/etc","/root","~/.ssh","~/.gnupg","~/.aws"]',
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  CLOUDFLARED / TUNNEL CONFIGURATION
-- ═══════════════════════════════════════════════════════════

CREATE TABLE tunnel_config (
    id              TEXT PRIMARY KEY DEFAULT 'default',
    tunnel_kind     TEXT NOT NULL DEFAULT 'cloudflare',  -- 'cloudflare' | 'tailscale' | 'ngrok' | 'custom' | 'none'
    enabled         BOOLEAN NOT NULL DEFAULT 0,
    -- Cloudflare-specific
    cf_tunnel_token TEXT,                -- cloudflared tunnel token (encrypted at rest)
    cf_tunnel_id    TEXT,                -- tunnel UUID
    cf_account_id   TEXT,
    cf_credentials_json TEXT,            -- credentials-file content (encrypted)
    -- Ingress rules (JSON array)
    cf_ingress_rules TEXT DEFAULT '[
        {"hostname": "", "service": "grpc://localhost:50051"},
        {"service": "http_status:404"}
    ]',
    -- General tunnel fields
    custom_start_command TEXT,           -- for kind='custom'
    custom_url_pattern   TEXT,
    -- Process management
    auto_start      BOOLEAN DEFAULT 1,  -- start cloudflared on container boot
    restart_on_fail BOOLEAN DEFAULT 1,
    health_check_interval_seconds INTEGER DEFAULT 30,
    -- Status (written by tunnel manager, not user)
    tunnel_status   TEXT DEFAULT 'stopped',  -- 'running' | 'stopped' | 'error'
    tunnel_url      TEXT,                    -- assigned URL (populated at runtime)
    tunnel_pid      INTEGER,
    last_error      TEXT,
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  AUTONOMY & SECURITY POLICIES
-- ═══════════════════════════════════════════════════════════

CREATE TABLE security_policies (
    id              TEXT PRIMARY KEY DEFAULT 'default',
    autonomy_level  TEXT DEFAULT 'supervised',
    max_actions_per_hour INTEGER DEFAULT 100,
    max_cost_per_day_usd REAL DEFAULT 10.0,
    monthly_cost_limit_usd REAL DEFAULT 100.0,
    warn_at_percent INTEGER DEFAULT 80,
    sandbox_enabled BOOLEAN DEFAULT 1,
    -- Secrets
    secrets_encrypt BOOLEAN DEFAULT 1,
    secrets_key_path TEXT DEFAULT '/data/.secret_key',
    updated_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  CRON / SCHEDULED TASKS
-- ═══════════════════════════════════════════════════════════

CREATE TABLE scheduled_tasks (
    id              TEXT PRIMARY KEY,
    cron_expr       TEXT,                -- cron expression (NULL for one-shot)
    command         TEXT NOT NULL,
    timezone        TEXT DEFAULT 'UTC',
    agent_id        TEXT,                -- which agent runs it (NULL = default)
    enabled         BOOLEAN DEFAULT 1,
    last_run_at     TEXT,
    next_run_at     TEXT,
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  CONFIG VERSIONING — audit trail for config changes
-- ═══════════════════════════════════════════════════════════

CREATE TABLE config_changelog (
    id              INTEGER PRIMARY KEY,
    table_name      TEXT NOT NULL,
    record_id       TEXT NOT NULL,
    field_name      TEXT,
    old_value       TEXT,
    new_value       TEXT,
    changed_by      TEXT,               -- agent ID or 'operator'
    timestamp       TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes
CREATE INDEX idx_agent_instances_status ON agent_instances(status);
CREATE INDEX idx_channels_type ON channels(channel_type);
CREATE INDEX idx_config_changelog_table ON config_changelog(table_name, record_id);
```

### Mapping to ZeroClaw's `config.toml`

| config.toml section | `_system.db` table | Notes |
|---|---|---|
| Top-level (`api_key`, `default_provider`, `default_model`) | `providers` (row with `is_default=1`) | Agent reads default provider on boot |
| `[identity]` | `identities` | SOUL.md, AGENTS.md, etc. stored as TEXT columns |
| `[memory]` | Handled by `_catalog.db` | Memory DBs are registered databases, not config |
| `[channels_config.*]` | `channels` | One row per channel, `config_json` holds channel-specific fields |
| `[gateway]` | `security_policies` + `tunnel_config` | Gateway bind/pairing maps to security; tunnel maps to cloudflared |
| `[autonomy]` | `security_policies` + `workspaces` | Autonomy level, allowed commands, forbidden paths |
| `[tunnel]` | `tunnel_config` | Full cloudflared config including credentials and ingress rules |
| `[runtime]` | `agent_instances` | Each instance knows its `runtime_kind` |
| `[cost]` | `providers` + `security_policies` | Per-provider and global cost limits |
| `[skills]` | `skills` | Each skill is a row; enable/disable, audit status |
| `[browser]`, `[heartbeat]`, etc. | `tools` | Tool-specific config stored as JSON in `config_json` |
| `[hardware]`, `[peripherals]` | `tools` | Category `'hardware'`, config_json holds serial/probe settings |
| `[secrets]` | `security_policies` | Encryption toggle and key path |

### Import / export for humans

```
# Import existing config.toml into _system.db
memory-server config import --file ~/.zeroclaw/config.toml

# Export _system.db back to config.toml for zeroclaw CLI compatibility
memory-server config export --format toml > config.toml

# Export as JSON (for programmatic use)
memory-server config export --format json
```

---

## 6. Cloudflared Integration

### Why bundle cloudflared?

ZeroClaw already supports Cloudflare Tunnels. By bundling cloudflared directly into this container, agents get **zero-config remote access** — the gRPC port is exposed through Cloudflare's edge network without opening firewall ports, configuring NAT, or running a separate tunnel container.

### How it works

1. **On boot**, the Tunnel Manager reads `tunnel_config` from `_system.db`
2. If `enabled=true` and `tunnel_kind='cloudflare'`, it writes a temporary config YAML to `/tmp/cloudflared.yml` derived from the DB fields
3. It spawns `cloudflared tunnel run` as a child process via `tokio::process::Command`
4. Stdout/stderr are captured and streamed to the tracing log
5. The Tunnel Manager monitors the process health (restart on crash, exponential backoff)
6. The assigned tunnel URL is written back to `tunnel_config.tunnel_url` for agents to discover

### Dockerfile impact

cloudflared is a static Go binary (~35 MB). This makes the image larger than the pure scratch approach but still tiny for what it provides. We use a **distroless** base instead of scratch to get the CA cert bundle needed by cloudflared's TLS:

```dockerfile
# ── Stage 1: Build Rust binary ──────────────────────────────
FROM rust:1.83-alpine AS builder
RUN apk add --no-cache musl-dev perl make
WORKDIR /app
COPY . .
RUN RUSTFLAGS="-C target-feature=+crt-static" \
    cargo build --release --target x86_64-unknown-linux-musl

# ── Stage 2: Fetch cloudflared ──────────────────────────────
FROM alpine:3.21 AS cloudflared-fetcher
ARG CLOUDFLARED_VERSION=2025.2.1
ARG TARGETARCH=amd64
RUN wget -O /cloudflared \
    "https://github.com/cloudflare/cloudflared/releases/download/${CLOUDFLARED_VERSION}/cloudflared-linux-${TARGETARCH}" \
    && chmod +x /cloudflared

# ── Stage 3: Runtime ────────────────────────────────────────
FROM gcr.io/distroless/static-debian12:nonroot

COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/memory-server /memory-server
COPY --from=cloudflared-fetcher /cloudflared /usr/bin/cloudflared

EXPOSE 50051
VOLUME ["/data"]

ENTRYPOINT ["/memory-server"]
```

**Estimated final image size: ~47 MB** (Rust binary ~12 MB + cloudflared ~35 MB + distroless base ~2 MB).

### Tunnel configuration via gRPC

Agents and operators configure the tunnel through the `ConfigService` RPCs:

```
# Set tunnel token (from Cloudflare dashboard)
config.SetField(table="tunnel_config", id="default", field="cf_tunnel_token", value="eyJ...")

# Enable the tunnel
config.SetField(table="tunnel_config", id="default", field="enabled", value="true")

# Server automatically spawns cloudflared, populates tunnel_url
config.GetTunnelStatus() → { status: "running", url: "https://my-agent.cfargotunnel.com" }
```

---

## 7. Cloudflare Management Database (`_cloudflare.db`)

The server maintains a third internal database at `/data/_cloudflare.db` that acts as a **local cache and management layer for the Cloudflare API**. This gives agents and operators the ability to manage DNS records, tunnels, and zones through the same gRPC interface — with local state that survives API outages, enables diff-based syncing, and provides an audit trail.

### Why a separate database?

Cloudflare state is fundamentally different from both the catalog (internal DB lifecycle) and system config (deployment settings). DNS records and tunnel configs are **external resources owned by Cloudflare** that we mirror locally. Keeping them separate means you can blow away `_cloudflare.db` and re-sync from the API without affecting anything else. It also keeps the `_system.db` focused on zeroclaw config rather than mixing in third-party API state.

### Schema

```sql
-- ═══════════════════════════════════════════════════════════
--  CLOUDFLARE ACCOUNT & AUTH
-- ═══════════════════════════════════════════════════════════

CREATE TABLE cf_accounts (
    account_id      TEXT PRIMARY KEY,       -- Cloudflare account ID
    name            TEXT,
    api_token_encrypted TEXT NOT NULL,       -- encrypted Bearer token
    email           TEXT,                    -- optional, for Global API Key auth
    api_key_encrypted TEXT,                  -- optional, Global API Key (encrypted)
    is_default      BOOLEAN DEFAULT 0,
    verified_at     TEXT,                    -- last successful API call timestamp
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  ZONES
-- ═══════════════════════════════════════════════════════════

CREATE TABLE cf_zones (
    zone_id         TEXT PRIMARY KEY,       -- Cloudflare zone ID
    account_id      TEXT NOT NULL REFERENCES cf_accounts(account_id),
    name            TEXT NOT NULL,           -- e.g. "example.com"
    status          TEXT,                    -- 'active' | 'pending' | 'initializing' | 'moved' | 'deleted'
    paused          BOOLEAN DEFAULT 0,
    plan_name       TEXT,                    -- 'free' | 'pro' | 'business' | 'enterprise'
    name_servers    TEXT,                    -- JSON array of assigned nameservers
    original_name_servers TEXT,              -- JSON array of original registrar nameservers
    -- Sync metadata
    last_synced_at  TEXT,                    -- last full sync from Cloudflare API
    etag            TEXT,                    -- API response ETag for conditional requests
    created_at      TEXT NOT NULL DEFAULT (datetime('now')),
    cf_created_on   TEXT,                    -- Cloudflare's created_on timestamp
    cf_modified_on  TEXT
);

-- ═══════════════════════════════════════════════════════════
--  DNS RECORDS (local mirror of Cloudflare DNS)
-- ═══════════════════════════════════════════════════════════

CREATE TABLE cf_dns_records (
    record_id       TEXT PRIMARY KEY,       -- Cloudflare DNS record ID
    zone_id         TEXT NOT NULL REFERENCES cf_zones(zone_id) ON DELETE CASCADE,
    -- Core record fields (mirror Cloudflare API response)
    record_type     TEXT NOT NULL,           -- 'A' | 'AAAA' | 'CNAME' | 'MX' | 'TXT' | 'SRV' | 'NS' | 'CAA' | ...
    name            TEXT NOT NULL,           -- full record name, e.g. "app.example.com"
    content         TEXT NOT NULL,           -- record value (IP, hostname, text, etc.)
    ttl             INTEGER DEFAULT 1,       -- 1 = auto
    proxied         BOOLEAN DEFAULT 0,       -- Cloudflare orange-cloud proxy
    proxiable       BOOLEAN DEFAULT 1,
    priority        INTEGER,                 -- for MX, SRV records
    -- Extended fields
    comment         TEXT,
    tags            TEXT DEFAULT '[]',       -- JSON array of "key:value" tags
    settings_json   TEXT DEFAULT '{}',       -- ipv4_only, ipv6_only, etc.
    -- Management metadata
    managed_by      TEXT,                    -- NULL = manual, 'tunnel' = auto-created by tunnel, 'agent' = created by agent
    agent_id        TEXT,                    -- which agent created/owns this record (if managed_by='agent')
    purpose         TEXT,                    -- human-readable note: "gRPC tunnel endpoint", "webhook for discord", etc.
    -- Sync state
    sync_status     TEXT DEFAULT 'synced',   -- 'synced' | 'pending_create' | 'pending_update' | 'pending_delete' | 'conflict'
    local_modified  BOOLEAN DEFAULT 0,       -- dirty flag: modified locally, not yet pushed
    cf_created_on   TEXT,
    cf_modified_on  TEXT,
    last_synced_at  TEXT,
    local_updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  TUNNELS
-- ═══════════════════════════════════════════════════════════

CREATE TABLE cf_tunnels (
    tunnel_id       TEXT PRIMARY KEY,       -- Cloudflare tunnel UUID
    account_id      TEXT NOT NULL REFERENCES cf_accounts(account_id),
    name            TEXT NOT NULL,           -- tunnel name
    status          TEXT,                    -- 'active' | 'inactive' | 'degraded'
    -- Credentials
    tunnel_token_encrypted TEXT,             -- encrypted tunnel run token
    credentials_json_encrypted TEXT,         -- encrypted credentials-file content
    -- Connection info (populated at runtime)
    connections_count INTEGER DEFAULT 0,     -- healthy connections (max 4 per replica)
    connections_json TEXT DEFAULT '[]',      -- JSON array of connection details
    -- Config
    config_src      TEXT DEFAULT 'local',    -- 'local' | 'cloudflare' (remote-managed)
    warp_routing    BOOLEAN DEFAULT 0,       -- private network routing enabled
    protocol        TEXT DEFAULT 'auto',     -- 'auto' | 'quic' | 'http2'
    -- Metadata
    remote_config   TEXT,                    -- JSON: last known remote config from API
    cf_created_at   TEXT,
    last_synced_at  TEXT,
    local_updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  TUNNEL INGRESS RULES
-- ═══════════════════════════════════════════════════════════

CREATE TABLE cf_tunnel_ingress (
    id              INTEGER PRIMARY KEY,
    tunnel_id       TEXT NOT NULL REFERENCES cf_tunnels(tunnel_id) ON DELETE CASCADE,
    priority        INTEGER NOT NULL,        -- order of evaluation (0 = first, catch-all = last)
    hostname        TEXT,                     -- NULL = catch-all rule
    path_regex      TEXT,                     -- optional path pattern
    service         TEXT NOT NULL,            -- e.g. "grpc://localhost:50051", "http_status:404"
    -- Origin request overrides
    origin_config_json TEXT DEFAULT '{}',     -- connectTimeout, noTLSVerify, httpHostHeader, etc.
    -- Management
    managed_by      TEXT,                     -- NULL = manual, 'system' = auto-created for gRPC
    purpose         TEXT,                     -- "gRPC memory service", "zeroclaw gateway webhook"
    created_at      TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ═══════════════════════════════════════════════════════════
--  TUNNEL ROUTES (private network CIDR routes)
-- ═══════════════════════════════════════════════════════════

CREATE TABLE cf_tunnel_routes (
    route_id        TEXT PRIMARY KEY,        -- Cloudflare route ID
    tunnel_id       TEXT NOT NULL REFERENCES cf_tunnels(tunnel_id) ON DELETE CASCADE,
    network         TEXT NOT NULL,            -- CIDR, e.g. "10.0.0.0/8"
    comment         TEXT,
    virtual_network_id TEXT,                  -- optional: Cloudflare virtual network
    cf_created_at   TEXT,
    last_synced_at  TEXT
);

-- ═══════════════════════════════════════════════════════════
--  SYNC LOG — audit trail for all Cloudflare API interactions
-- ═══════════════════════════════════════════════════════════

CREATE TABLE cf_sync_log (
    id              INTEGER PRIMARY KEY,
    entity_type     TEXT NOT NULL,           -- 'dns_record' | 'tunnel' | 'zone' | 'ingress' | 'route'
    entity_id       TEXT NOT NULL,
    action          TEXT NOT NULL,           -- 'create' | 'update' | 'delete' | 'sync_pull' | 'sync_push'
    direction       TEXT NOT NULL,           -- 'local_to_cf' | 'cf_to_local'
    request_json    TEXT,                    -- what we sent (redacted)
    response_json   TEXT,                    -- what we got back (redacted)
    status          TEXT NOT NULL,           -- 'success' | 'error' | 'pending'
    error_message   TEXT,
    triggered_by    TEXT,                    -- 'agent:<id>' | 'operator' | 'sync_task' | 'system'
    timestamp       TEXT NOT NULL DEFAULT (datetime('now'))
);

-- Indexes
CREATE INDEX idx_dns_records_zone ON cf_dns_records(zone_id);
CREATE INDEX idx_dns_records_type ON cf_dns_records(record_type);
CREATE INDEX idx_dns_records_sync ON cf_dns_records(sync_status) WHERE sync_status != 'synced';
CREATE INDEX idx_dns_records_managed ON cf_dns_records(managed_by) WHERE managed_by IS NOT NULL;
CREATE INDEX idx_tunnel_ingress_tunnel ON cf_tunnel_ingress(tunnel_id, priority);
CREATE INDEX idx_sync_log_entity ON cf_sync_log(entity_type, entity_id);
CREATE INDEX idx_sync_log_time ON cf_sync_log(timestamp);
```

### Sync strategy: local-first with push/pull

The server operates as a **local-first cache** of Cloudflare state. This means:

1. **Pull** — on boot and on a configurable interval (e.g. every 5 minutes), the Cloudflare Sync Task fetches zones, DNS records, and tunnel state from the API and upserts into `_cloudflare.db`. Conflicts are flagged with `sync_status = 'conflict'`.

2. **Push** — when an agent or operator creates/updates/deletes a record via gRPC, the change is written locally first (`sync_status = 'pending_create'`), then pushed to the Cloudflare API asynchronously. On success, status flips to `'synced'`. On failure, it stays pending with the error logged.

3. **Offline resilience** — if the Cloudflare API is unreachable, local changes queue up. The sync task retries on the next interval. Agents can always read cached DNS records even during an API outage.

```
Agent/Operator                    _cloudflare.db              Cloudflare API
      │                                │                            │
      │  CreateDnsRecord(...)          │                            │
      ├──────────────────────────────►│                            │
      │                   INSERT (sync_status='pending_create')    │
      │                                │                            │
      │                                │  POST /dns_records         │
      │                                ├───────────────────────────►│
      │                                │                            │
      │                                │  200 OK + record_id        │
      │                                │◄───────────────────────────┤
      │                   UPDATE sync_status='synced'              │
      │◄──────────────────────────────│                            │
      │  { record_id, sync_status }    │                            │
```

### What agents can do via gRPC

```
# DNS management
cf.ListZones()
cf.ListDnsRecords(zone_id, filters...)
cf.CreateDnsRecord(zone_id, type="A", name="agent-api.example.com", content="...", proxied=true)
cf.UpdateDnsRecord(record_id, content="new-ip")
cf.DeleteDnsRecord(record_id)
cf.SyncDns(zone_id)          # force pull from Cloudflare API

# Tunnel management
cf.ListTunnels(account_id)
cf.GetTunnel(tunnel_id)
cf.CreateTunnel(account_id, name="agent-tunnel")
cf.DeleteTunnel(tunnel_id)
cf.SetIngressRules(tunnel_id, rules=[...])
cf.GetTunnelConnections(tunnel_id)  # live connection health

# Full sync
cf.SyncAll()                  # pull all zones + records + tunnels
cf.GetSyncLog(entity_type, entity_id)  # audit trail
```

---

## 8. Database Migrations System

With four internal databases (`_catalog.db`, `_system.db`, `_cloudflare.db`, plus user data DBs), we need a robust migration system from day one to handle schema evolution without data loss.

### Design

Each database has its own migration sequence, stored as numbered SQL files. The server tracks which migrations have been applied in a `_migrations` table inside each database.

### Migration file layout

```
memory-server/
├── migrations/
│   ├── catalog/
│   │   ├── 001_initial_schema.sql
│   │   ├── 002_add_capabilities_index.sql
│   │   └── ...
│   ├── system/
│   │   ├── 001_initial_schema.sql
│   │   ├── 002_add_scheduled_tasks.sql
│   │   └── ...
│   ├── cloudflare/
│   │   ├── 001_initial_schema.sql
│   │   ├── 002_add_tunnel_routes.sql
│   │   └── ...
│   └── templates/
│       ├── 001_memories_vec_fts.sql      # default schema for new embedding_store DBs
│       └── 002_conversation_history.sql  # default schema for conversation_history DBs
```

### Migration tracking table (created in every database)

```sql
-- Auto-created by the migration runner on first boot
CREATE TABLE IF NOT EXISTS _migrations (
    version     INTEGER PRIMARY KEY,     -- migration number (001, 002, ...)
    name        TEXT NOT NULL,           -- filename without number prefix
    checksum    TEXT NOT NULL,           -- SHA256 of the migration SQL
    applied_at  TEXT NOT NULL DEFAULT (datetime('now')),
    duration_ms INTEGER                  -- how long the migration took
);
```

### Migration file format

Each `.sql` file is a plain SQL script. Wrap multi-statement migrations in a transaction:

```sql
-- migrations/catalog/002_add_capabilities_index.sql
-- Description: Add index on capabilities for faster discovery queries
-- Requires: 001

BEGIN;

CREATE INDEX IF NOT EXISTS idx_databases_capabilities
ON databases(capabilities);

ALTER TABLE databases ADD COLUMN tags TEXT DEFAULT '[]';

COMMIT;
```

### Migration runner logic (Rust pseudo-code)

```rust
fn run_migrations(conn: &Connection, db_name: &str, migrations_dir: &Path) -> Result<()> {
    // 1. Ensure _migrations table exists
    conn.execute_batch("CREATE TABLE IF NOT EXISTS _migrations (...)")?;

    // 2. Get current version
    let current: i64 = conn.query_row(
        "SELECT COALESCE(MAX(version), 0) FROM _migrations", [], |r| r.get(0)
    )?;

    // 3. Read migration files, sorted by version number
    let pending = read_migration_files(migrations_dir)?
        .into_iter()
        .filter(|m| m.version > current);

    // 4. Apply each pending migration
    for migration in pending {
        let start = Instant::now();

        // Verify checksum hasn't changed for already-applied migrations
        let checksum = sha256(&migration.sql);

        conn.execute_batch(&migration.sql)?;

        conn.execute(
            "INSERT INTO _migrations (version, name, checksum, duration_ms) VALUES (?1, ?2, ?3, ?4)",
            params![migration.version, migration.name, checksum, start.elapsed().as_millis()],
        )?;

        info!("Applied migration {}/{}: {}", db_name, migration.version, migration.name);
    }

    Ok(())
}
```

### Boot sequence with migrations

```
Server starts
    │
    ├─► Open/create _catalog.db  → run_migrations("catalog/")
    ├─► Open/create _system.db   → run_migrations("system/")
    ├─► Open/create _cloudflare.db → run_migrations("cloudflare/")
    │
    ├─► Scan /data/*.db for user databases
    │   └─► For each: check _migrations table, apply any pending template migrations
    │
    ├─► Start gRPC server
    ├─► Start Reaper task
    ├─► Start Cloudflare Sync task
    └─► Start Tunnel Manager (if enabled)
```

### CLI commands for migrations

```bash
# Check migration status for all databases
memory-server migrate status

# Apply pending migrations (happens automatically on boot too)
memory-server migrate up

# Generate a new migration file
memory-server migrate new --db catalog --name "add_quota_limits"
# → creates migrations/catalog/003_add_quota_limits.sql

# Dry-run: show what would be applied without executing
memory-server migrate plan

# Verify checksums (detect tampered migrations)
memory-server migrate verify
```

### Embedding migrations in the binary

For the `scratch` Docker image, migration SQL files are embedded into the binary at compile time using Rust's `include_str!` or the `rust-embed` crate. No filesystem reads needed at runtime:

```rust
#[derive(RustEmbed)]
#[folder = "migrations/"]
struct Migrations;

// Access: Migrations::get("catalog/001_initial_schema.sql")
```

---

## 9. SQLite Concurrency Strategy

SQLite is single-writer. To serve multiple agents:

1. **WAL mode** — enables concurrent readers alongside one writer
2. **Connection handling** — one `RwLock<Connection>` per database file:
   - Reads acquire a shared (read) lock → many readers in parallel
   - Writes acquire an exclusive (write) lock → serialized, but fast
3. **Busy timeout** — set `PRAGMA busy_timeout = 5000` so writes queue rather than fail
4. **Per-DB isolation** — each database file gets its own lock, so agents writing to different DBs never contend

---

## 10. Protobuf Service Definition (Draft)

```protobuf
syntax = "proto3";
package memory.v1;

import "google/protobuf/timestamp.proto";

// ══════════════════════════════════════════════════════════
//  Catalog Service — database registry & lifecycle
// ══════════════════════════════════════════════════════════
service CatalogService {
  // Discovery: find databases by role, capability, or owner
  rpc Discover(DiscoverRequest)               returns (DiscoverResponse);

  // Provisioning: create and register a new database
  rpc Provision(ProvisionRequest)             returns (ProvisionResponse);

  // Drop a database (ephemeral or persistent)
  rpc DropDatabase(DropDatabaseRequest)       returns (DropDatabaseResponse);

  // Access control
  rpc GrantAccess(GrantAccessRequest)         returns (GrantAccessResponse);
  rpc RevokeAccess(RevokeAccessRequest)       returns (RevokeAccessResponse);
  rpc WhoHasAccess(WhoHasAccessRequest)       returns (WhoHasAccessResponse);

  // Inspect a specific database's catalog entry
  rpc Describe(DescribeRequest)               returns (DescribeResponse);

  // List everything (admin view)
  rpc ListAll(ListAllRequest)                 returns (ListAllResponse);

  // Update metadata (purpose, tags, TTL extension)
  rpc UpdateEntry(UpdateEntryRequest)         returns (UpdateEntryResponse);

  // Heartbeat: agents call this to extend TTL on ephemeral DBs
  rpc Touch(TouchRequest)                     returns (TouchResponse);
}

// ── Catalog messages ──────────────────────────────────────

message DiscoverRequest {
  repeated string roles        = 1;  // filter by role(s)
  repeated string capabilities = 2;  // filter: must have ALL of these
  string owner_agent           = 3;  // optional: only this agent's DBs
  bool include_ephemeral       = 4;  // default false: skip runtime DBs
}

message DiscoverResponse {
  repeated DatabaseInfo databases = 1;
}

message ProvisionRequest {
  string purpose         = 1;  // human-readable description
  string role            = 2;  // e.g. "agent_runtime", "embedding_store"
  string lifecycle       = 3;  // "persistent" | "ephemeral"
  string owner_agent     = 4;  // who owns this (empty = shared)
  int32  ttl_seconds     = 5;  // for ephemeral: auto-expire after N seconds (0 = no expiry)
  repeated string capabilities = 6;  // ["vec", "fts", "raw"]
  string metadata_json   = 7;  // arbitrary JSON (embedding dims, model name, etc.)
  string db_name_hint    = 8;  // optional: preferred name (server may suffix to avoid collision)
}

message ProvisionResponse {
  DatabaseInfo database = 1;
}

message DropDatabaseRequest {
  string db_name  = 1;
  bool   force    = 2;   // drop even if agents currently have access
}

message DropDatabaseResponse {
  bool deleted = 1;
}

message GrantAccessRequest {
  string db_name    = 1;
  string agent_id   = 2;
  string permission = 3;  // "read" | "write" | "admin"
}
message GrantAccessResponse {}

message RevokeAccessRequest {
  string db_name  = 1;
  string agent_id = 2;
}
message RevokeAccessResponse {}

message WhoHasAccessRequest {
  string db_name = 1;
}
message WhoHasAccessResponse {
  repeated AccessGrant grants = 1;
}

message AccessGrant {
  string agent_id   = 1;
  string permission = 2;
  string granted_at = 3;
}

message DescribeRequest {
  string db_name = 1;
}
message DescribeResponse {
  DatabaseInfo database = 1;
  repeated AccessGrant grants = 2;
  int64 file_size_bytes = 3;
  int64 page_count      = 4;
}

message ListAllRequest {}
message ListAllResponse {
  repeated DatabaseInfo databases = 1;
}

message UpdateEntryRequest {
  string db_name       = 1;
  string purpose       = 2;  // empty = no change
  string metadata_json = 3;  // empty = no change
  int32  extend_ttl    = 4;  // seconds to add to expires_at
}
message UpdateEntryResponse {
  DatabaseInfo database = 1;
}

message TouchRequest {
  string db_name   = 1;
  string agent_id  = 2;
  int32  extend_by = 3;  // seconds to add (0 = use original TTL)
}
message TouchResponse {
  string new_expires_at = 1;
}

message DatabaseInfo {
  string db_name          = 1;
  string purpose          = 2;
  string role             = 3;
  string lifecycle        = 4;
  string owner_agent      = 5;
  repeated string capabilities = 6;
  int32  schema_version   = 7;
  string created_at       = 8;
  string last_accessed_at = 9;
  string expires_at       = 10;
  string metadata_json    = 11;
}


// ══════════════════════════════════════════════════════════
//  Memory Service — data operations on managed databases
// ══════════════════════════════════════════════════════════
service MemoryService {
  // Raw SQL (for schema setup, complex queries)
  rpc Execute(ExecuteRequest)                 returns (ExecuteResponse);
  rpc Query(QueryRequest)                     returns (QueryResponse);

  // ── Vector search ──────────────────────────────────────
  rpc VectorSearch(VectorSearchRequest)       returns (VectorSearchResponse);
  rpc VectorUpsert(VectorUpsertRequest)       returns (VectorUpsertResponse);

  // ── Full-text search (BM25) ────────────────────────────
  rpc TextSearch(TextSearchRequest)           returns (TextSearchResponse);
  rpc TextIndex(TextIndexRequest)             returns (TextIndexResponse);

  // ── Hybrid (vector + text combined) ────────────────────
  rpc HybridSearch(HybridSearchRequest)       returns (HybridSearchResponse);

  // ── Streaming for bulk ops ─────────────────────────────
  rpc BulkUpsert(stream BulkUpsertChunk)      returns (BulkUpsertResponse);
}

// ── Memory Service messages ───────────────────────────────

message ExecuteRequest {
  string db_name = 1;
  string sql     = 2;
  repeated Value params = 3;
}

message VectorSearchRequest {
  string db_name       = 1;
  string table         = 2;
  string column        = 3;      // vec0 column name
  repeated float query = 4;      // query embedding
  int32 k              = 5;      // top-K neighbours
  string where_clause  = 6;      // optional SQL filter
}

message TextSearchRequest {
  string db_name    = 1;
  string table      = 2;         // FTS5 table name
  string query      = 3;         // FTS5 MATCH expression
  int32 limit       = 4;
  repeated double column_weights = 5;  // bm25 column weights
}

message HybridSearchRequest {
  string db_name           = 1;
  string fts_table         = 2;
  string vec_table         = 3;
  string text_query        = 4;
  repeated float embedding = 5;
  int32 k                  = 6;
  float text_weight        = 7;  // blend ratio
  float vec_weight         = 8;
}

// Value is a tagged union for query parameters
message Value {
  oneof kind {
    int64  int_value    = 1;
    double float_value  = 2;
    string string_value = 3;
    bytes  blob_value   = 4;
  }
}
```

---

## 11. Database Schema Patterns

### Vector memory table (sqlite-vec)

```sql
-- Metadata in a regular table
CREATE TABLE memories (
    id         INTEGER PRIMARY KEY,
    agent_id   TEXT NOT NULL,
    content    TEXT NOT NULL,
    metadata   TEXT,                -- JSON blob
    created_at TEXT DEFAULT (datetime('now')),
    updated_at TEXT DEFAULT (datetime('now'))
);

-- Vector index as a virtual table
CREATE VIRTUAL TABLE vec_memories USING vec0(
    embedding float[1536]          -- match your model's dimension
);
```

### Full-text search table (FTS5 + BM25)

```sql
-- External-content FTS5 pointing at the memories table
CREATE VIRTUAL TABLE fts_memories USING fts5(
    content,
    metadata,
    content=memories,
    content_rowid=id,
    tokenize='porter unicode61'
);

-- Keep FTS in sync via triggers
CREATE TRIGGER memories_ai AFTER INSERT ON memories BEGIN
    INSERT INTO fts_memories(rowid, content, metadata)
    VALUES (new.id, new.content, new.metadata);
END;

CREATE TRIGGER memories_ad AFTER DELETE ON memories BEGIN
    INSERT INTO fts_memories(fts_memories, rowid, content, metadata)
    VALUES ('delete', old.id, old.content, old.metadata);
END;

CREATE TRIGGER memories_au AFTER UPDATE ON memories BEGIN
    INSERT INTO fts_memories(fts_memories, rowid, content, metadata)
    VALUES ('delete', old.id, old.content, old.metadata);
    INSERT INTO fts_memories(rowid, content, metadata)
    VALUES (new.id, new.content, new.metadata);
END;
```

### Hybrid search query (combine both)

```sql
-- Reciprocal Rank Fusion of vector + BM25
WITH vec_hits AS (
    SELECT rowid, distance,
           ROW_NUMBER() OVER (ORDER BY distance) AS vec_rank
    FROM vec_memories
    WHERE embedding MATCH ?1
    ORDER BY distance LIMIT ?2
),
text_hits AS (
    SELECT rowid, bm25(fts_memories) AS score,
           ROW_NUMBER() OVER (ORDER BY bm25(fts_memories)) AS text_rank
    FROM fts_memories
    WHERE fts_memories MATCH ?3
    ORDER BY bm25(fts_memories) LIMIT ?2
),
fused AS (
    SELECT COALESCE(v.rowid, t.rowid) AS id,
           COALESCE(1.0 / (60 + v.vec_rank), 0) * ?4 +
           COALESCE(1.0 / (60 + t.text_rank), 0) * ?5 AS rrf_score
    FROM vec_hits v
    FULL OUTER JOIN text_hits t ON v.rowid = t.rowid
)
SELECT m.*, f.rrf_score
FROM fused f
JOIN memories m ON m.id = f.id
ORDER BY f.rrf_score DESC
LIMIT ?2;
```

---

## 12. Dockerfile (Multi-stage, Minimal)

```dockerfile
# ── Stage 1: Build ─────────────────────────────────────────
FROM rust:1.83-alpine AS builder

RUN apk add --no-cache musl-dev perl make

WORKDIR /app
COPY . .

# Static build targeting musl for scratch compatibility
RUN RUSTFLAGS="-C target-feature=+crt-static" \
    cargo build --release --target x86_64-unknown-linux-musl

# ── Stage 2: Runtime (FROM scratch = ~0 bytes base) ───────
FROM scratch

# Copy the single static binary
COPY --from=builder /app/target/x86_64-unknown-linux-musl/release/memory-server /memory-server

# Copy CA certs if TLS is needed later
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

EXPOSE 50051
VOLUME ["/data"]

ENTRYPOINT ["/memory-server"]
```

**Estimated final image size: 8–15 MB** (single static binary + CA certs).

---

## 13. Cargo.toml Skeleton

```toml
[package]
name    = "memory-server"
version = "0.1.0"
edition = "2024"

[dependencies]
tonic    = "0.12"
prost    = "0.13"
tokio    = { version = "1", features = ["rt-multi-thread", "macros", "process", "signal"] }
rusqlite = { version = "0.32", features = ["bundled", "vtab", "load_extension"] }
serde    = { version = "1", features = ["derive"] }
serde_json = "1"
toml     = "0.8"
rust-embed = "8"                           # Compile migrations into binary
sha2     = "0.10"                          # Migration checksum verification
reqwest  = { version = "0.12", default-features = false, features = ["rustls-tls", "json"] }  # Cloudflare API client
tracing  = "0.1"
tracing-subscriber = "0.3"

[build-dependencies]
tonic-build = "0.12"
cc       = "1"                             # Compile sqlite-vec C source

[profile.release]
opt-level    = "z"      # optimise for size
lto          = true     # link-time optimisation
codegen-units = 1
panic        = "abort"
strip        = true
```

The `bundled` feature compiles SQLite from source with FTS5 enabled.
`sqlite-vec` is compiled as a C file and linked via a `build.rs` script.

---

## 14. Build Script for sqlite-vec (`build.rs`)

```rust
fn main() {
    // 1. Compile all protobuf definitions
    tonic_build::compile_protos("proto/catalog.proto")
        .expect("Failed to compile catalog protos");
    tonic_build::compile_protos("proto/memory.proto")
        .expect("Failed to compile memory protos");
    tonic_build::compile_protos("proto/config.proto")
        .expect("Failed to compile config protos");
    tonic_build::compile_protos("proto/cloudflare.proto")
        .expect("Failed to compile cloudflare protos");

    // 2. Compile sqlite-vec as a static C library
    cc::Build::new()
        .file("vendor/sqlite-vec/sqlite-vec.c")
        .flag("-DSQLITE_CORE")       // link into core, not loadable ext
        .opt_level(2)
        .compile("sqlite_vec");
}
```

---

## 15. Project Directory Layout

```
memory-server/
├── Cargo.toml
├── Cargo.lock
├── build.rs
├── Dockerfile
├── proto/
│   ├── catalog.proto              # CatalogService definition
│   ├── memory.proto               # MemoryService definition
│   ├── config.proto               # ConfigService definition (_system.db)
│   └── cloudflare.proto           # CloudflareService definition (_cloudflare.db)
├── migrations/
│   ├── catalog/
│   │   └── 001_initial_schema.sql
│   ├── system/
│   │   └── 001_initial_schema.sql
│   ├── cloudflare/
│   │   └── 001_initial_schema.sql
│   └── templates/                 # applied to user DBs on Provision()
│       ├── 001_memories_vec_fts.sql
│       └── 002_conversation_history.sql
├── vendor/
│   └── sqlite-vec/
│       ├── sqlite-vec.c
│       └── sqlite-vec.h
└── src/
    ├── main.rs                    # tokio entrypoint, gRPC server boot, migration runner
    ├── migrations.rs              # Embed + run migrations per-DB on boot
    ├── catalog/
    │   ├── mod.rs                 # CatalogService gRPC impl
    │   ├── registry.rs            # CRUD against _catalog.db tables
    │   ├── reaper.rs              # Background task: evict expired ephemeral DBs
    │   └── access.rs              # Grant/revoke/check access_grants
    ├── config/
    │   ├── mod.rs                 # ConfigService gRPC impl
    │   ├── identities.rs          # CRUD: identities/souls
    │   ├── providers.rs           # CRUD: LLM providers + model routes
    │   ├── skills.rs              # CRUD: skills registry
    │   ├── tools.rs               # CRUD: tool configs
    │   ├── agents.rs              # CRUD: agent instances
    │   ├── channels.rs            # CRUD: messaging channels
    │   ├── workspaces.rs          # CRUD: shared folders + workspaces
    │   ├── security.rs            # CRUD: security policies
    │   └── import_export.rs       # TOML ↔ _system.db conversion
    ├── cloudflare/
    │   ├── mod.rs                 # CloudflareService gRPC impl
    │   ├── api_client.rs          # Cloudflare REST API client (reqwest/hyper)
    │   ├── dns.rs                 # DNS record CRUD + local cache
    │   ├── tunnels.rs             # Tunnel CRUD + ingress rules
    │   ├── zones.rs               # Zone management
    │   ├── sync.rs                # Background sync task (pull/push)
    │   └── tunnel_manager.rs      # Spawn/monitor cloudflared child process
    ├── service.rs                 # MemoryService gRPC impl
    ├── db/
    │   ├── mod.rs                 # DB manager (open, pool, WAL setup)
    │   ├── manager.rs             # HashMap<name, RwLock<Connection>> + catalog sync
    │   ├── vector.rs              # vec0 helpers (upsert, search)
    │   ├── fts.rs                 # FTS5 helpers (index, search, bm25)
    │   └── hybrid.rs              # RRF / hybrid search logic
    └── proto/                     # auto-generated by tonic-build
        ├── catalog.v1.rs
        ├── memory.v1.rs
        ├── config.v1.rs
        └── cloudflare.v1.rs
```

---

## 16. Implementation Phases

### Phase 1 — Migrations + Catalog + Skeleton (Day 1–3)
- Scaffold Cargo project, protobuf definitions for all four services
- Implement migration runner with `rust-embed` for compiled-in SQL files
- Write `001_initial_schema.sql` for `_catalog.db`, `_system.db`, `_cloudflare.db`
- Boot all three internal DBs on startup, run migrations
- Implement `CatalogService`: `Provision`, `Discover`, `Describe`, `ListAll`, `DropDatabase`
- DB manager loads open connections from catalog on boot
- WAL mode, busy timeout, `RwLock` connection management
- Implement `MemoryService`: `Execute`, `Query` (raw SQL against any registered DB)
- Docker build producing a working `distroless` image (with cloudflared binary)

### Phase 2 — System Config (Day 4–5)
- Implement `ConfigService`: full CRUD for identities, providers, skills, tools, agents, channels, workspaces, security policies
- TOML import/export (`config import` / `config export`) for zeroclaw CLI compatibility
- Config changelog audit trail
- Tunnel config reads from `_system.db` → `tunnel_config` table

### Phase 3 — Lifecycle & Access (Day 6)
- Ephemeral DB reaper background task (tokio interval, every 60s)
- `Touch` RPC for agents to extend ephemeral TTLs (heartbeat pattern)
- `GrantAccess`, `RevokeAccess`, `WhoHasAccess` RPCs
- Access check middleware: every `MemoryService` RPC verifies the caller has a grant
- Startup recovery: scan `/data/*.db`, reconcile orphans with catalog
- Audit logging to `catalog_events`

### Phase 4 — Vector Search (Day 7–8)
- `VectorUpsert` — inserts into both `memories` and `vec_memories`
- `VectorSearch` — KNN query with optional WHERE filter
- Catalog auto-tags `capabilities: ["vec"]` when vec tables are created
- Integration test: upsert 1000 embeddings, search top-10

### Phase 5 — Full-Text Search (Day 9)
- `TextIndex` — creates FTS5 virtual table + sync triggers
- `TextSearch` — MATCH query with BM25 ranking + column weights
- Catalog auto-tags `capabilities: ["fts"]`
- Integration test: index documents, verify BM25 ordering

### Phase 6 — Hybrid Search (Day 10)
- `HybridSearch` — Reciprocal Rank Fusion of vector + BM25
- Configurable blend weights via request fields

### Phase 7 — Cloudflare Management (Day 11–13)
- Implement `CloudflareService`: DNS record CRUD, zone listing, tunnel management
- Cloudflare REST API client (`reqwest` with retry + rate limiting)
- Local-first sync: pull on boot, push on mutation, background sync interval
- Tunnel Manager: spawn/monitor cloudflared child process
- Auto-create CNAME records when tunnel ingress rules are added
- `cf_sync_log` audit trail for all API interactions
- Integration test: create tunnel + DNS records + ingress rules end-to-end

### Phase 8 — Multi-Agent Hardening (Day 14–15)
- Load test: 10 concurrent gRPC clients, each discovering + provisioning DBs
- Streaming `BulkUpsert` for batch memory ingestion
- Health check / reflection RPCs
- Observability: `tracing` logs, optional Prometheus metrics
- Verify reaper correctly cleans up after agent disconnects
- Migration `verify` command: checksum validation

### Phase 9 — Polish & Ship (Day 16–18)
- CI pipeline (GitHub Actions: build, test, push image)
- `docker-compose.yml` example with volume mount
- `memory-server migrate` CLI commands (status, up, new, plan, verify)
- README with client examples (Python, Rust, Go) showing full discover → provision → search → DNS management flow
- Publish image to registry

---

## 17. Key Design Decisions

| Decision | Rationale |
|---|---|
| **`_catalog.db` as internal registry** | Single source of truth; survives restarts; queryable with SQL |
| **`_system.db` for zeroclaw config** | Replaces scattered TOML/env/markdown with concurrent-safe, queryable, versionable store |
| **`_cloudflare.db` as API cache** | Local-first Cloudflare state; survives API outages; enables offline queuing + audit trail |
| **Separate DB per concern** | Blast radius: deleting `_cloudflare.db` re-syncs from API; doesn't touch config or catalog |
| **Compiled-in migrations** | `rust-embed` bakes SQL into the binary; no filesystem dependency; works in scratch/distroless |
| **Migration runner per-DB** | Each internal DB evolves independently; user DBs get template migrations on provision |
| **Four gRPC services** | CatalogService, MemoryService, ConfigService, CloudflareService — clean separation of concerns |
| **Ephemeral DBs with TTL + reaper** | Agents get scratch space without manual cleanup; prevents disk bloat |
| **Touch/heartbeat for TTL extension** | Long-running agents keep their DBs alive; crashed agents auto-expire |
| **`access_grants` table** | Foundation for auth; even without mTLS, agents declare identity |
| **Local-first Cloudflare sync** | Write locally, push async; agents read cached DNS even during API outage |
| **cloudflared bundled in image** | Zero-config tunnel; agents get remote access without separate container or firewall changes |
| **distroless base (not scratch)** | cloudflared needs CA certs + libc; distroless provides both at ~2 MB |
| **Rust + musl for memory-server** | Our binary is still fully static; cloudflared is the only non-static component |
| **sqlite-vec compiled in** | No `.so` loading at runtime; smaller, simpler |
| **FTS5 bundled** | `rusqlite`'s `bundled` feature enables FTS5 by default |
| **WAL mode** | Concurrent reads + serialized writes = good enough for agent workloads |
| **RwLock per DB** | Simple, correct; avoids heavyweight connection pools |
| **Hybrid via RRF** | Standard fusion method; no ML model needed |
| **gRPC not REST** | Streaming, strong typing, efficient binary protocol for agent-to-agent comms |

---

## 18. Risk Register

| Risk | Mitigation |
|---|---|
| Catalog DB corruption bricks the whole server | Backup `_catalog.db` on every write; startup integrity check with `PRAGMA integrity_check` |
| Reaper deletes a DB an agent is actively using | Check open connection count before drop; `force` flag required to override |
| Orphan DB files on disk not in catalog | Startup reconciliation scans `/data/*.db` and registers unknowns as `role=unknown` |
| Runaway ephemeral DB creation fills disk | Configurable max-ephemeral-count and max-total-disk limits; `Provision` rejects when exceeded |
| Agent impersonation (no auth in v1) | `agent_id` is self-declared; Phase 8+ adds mTLS or token verification |
| SQLite write contention under heavy agent load | WAL + busy_timeout; consider sharding across DB files |
| sqlite-vec not yet 1.0 | Pin a known-good commit; vendor the source |
| Large embedding dimensions hurt perf | sqlite-vec supports int8 quantisation; use it for dims > 1024 |
| FTS5 index bloat on frequent updates | Schedule `INSERT INTO fts_memories(fts_memories) VALUES('optimize')` |
| gRPC reflection exposes schema | Disable in prod or gate behind auth |
| No auth by default | Layer mTLS or a token interceptor in Phase 8+ |
| Cloudflare API rate limiting (1200 req/5min) | Client-side rate limiter; batch API where possible; sync interval tunable |
| Cloudflare API token leaked from `_cloudflare.db` | Tokens encrypted at rest; DB file permissions 0600; secrets key in `/data/.secret_key` |
| Local DNS cache diverges from Cloudflare reality | `sync_status` flags + periodic pull; `SyncAll` RPC for manual reconciliation; conflicts surfaced to operator |
| cloudflared crashes in container | Tunnel Manager auto-restarts with exponential backoff; health check writes status to `_system.db` |
| cloudflared binary bloats image size (~35 MB) | Acceptable trade-off for zero-config tunnel; optional `--no-cloudflared` build flag for minimal image |
| Migration SQL has a bug, corrupts DB | Migrations run in transactions; checksum verification prevents silent edits; `migrate verify` command |
| Migration applied out of order across replicas | Version-locked sequential application; migration runner refuses gaps |
| _system.db grows large with many config changes | `config_changelog` has configurable retention; old entries auto-pruned |

---

## 19. Future Extensions (Out of Scope for v1)

- **mTLS / token auth** for production deployments (agent_id becomes verified identity)
- **Read replicas** via SQLite replication (Litestream or LiteFS)
- **Embedding generation** as an optional sidecar (calls Ollama / OpenAI)
- **Change notifications** via gRPC server-streaming (agent subscribes to memory updates)
- **Catalog webhooks** — notify external systems when databases are created/expired
- **Database templates** — `Provision(template="rag_store")` auto-creates schema + vec + fts tables via template migrations
- **Quota management** — per-agent disk/DB limits enforced at the catalog level
- **Catalog replication** — sync `_catalog.db` across multiple server instances for HA
- **Cloudflare Workers integration** — deploy lightweight agents at the edge via Workers + D1
- **Cloudflare Access policies** — auto-create Zero Trust policies for tunnel-exposed services
- **DNSSEC management** — enable/disable DNSSEC per zone via `CloudflareService`
- **Cloudflare Load Balancing** — multi-tunnel failover with health checks
- **Migration rollback** — `migrate down` with reversible migration pairs (up/down SQL)
- **Admin UI** — lightweight web dashboard reading from all internal DBs (list databases, view access, DNS records, trigger sync, manage config)
- **WASM build** for running in-browser or on edge (without cloudflared)
