# Memory Trait — Architecture Deep-Dive

> **Source**: `src/memory/traits.rs`
> **Factory**: `src/memory/mod.rs`
> **Parent doc**: [Architecture Overview](../overview.md)

---

## Purpose

The `Memory` trait abstracts long-term knowledge storage, retrieval, and lifecycle management. The agent uses memory to persist facts, preferences, conversation summaries, and contextual information across sessions. During each turn, the agent recalls relevant memories to inject as context before the LLM call, and after each turn, it extracts and stores new information.

Memory is what makes the agent a persistent entity rather than a stateless function call.

**When to implement**: You want to add a new storage backend (e.g., Redis, PostgreSQL, Pinecone, Weaviate, a custom vector DB).

---

## Trait Definition

```rust
#[async_trait]
pub trait Memory: Send + Sync {
    // ── Identification ────────────────────────────────────────

    /// Backend name (e.g., "sqlite", "markdown", "qdrant").
    fn name(&self) -> &str;

    // ── Core CRUD ─────────────────────────────────────────────

    /// REQUIRED. Store a memory entry.
    async fn store(
        &self,
        key: &str,
        content: &str,
        category: MemoryCategory,
        session_id: Option<&str>,
    ) -> anyhow::Result<()>;

    /// REQUIRED. Semantic recall — search memories by query.
    async fn recall(
        &self,
        query: &str,
        limit: usize,
        session_id: Option<&str>,
        since: Option<&str>,
        until: Option<&str>,
    ) -> anyhow::Result<Vec<MemoryEntry>>;

    /// REQUIRED. Get a specific memory by key.
    async fn get(&self, key: &str) -> anyhow::Result<Option<MemoryEntry>>;

    /// REQUIRED. List memories, optionally filtered by category/session.
    async fn list(
        &self,
        category: Option<&MemoryCategory>,
        session_id: Option<&str>,
    ) -> anyhow::Result<Vec<MemoryEntry>>;

    /// REQUIRED. Delete a memory by key. Returns true if found & deleted.
    async fn forget(&self, key: &str) -> anyhow::Result<bool>;

    // ── Bulk operations ───────────────────────────────────────

    /// Delete all memories in a namespace. Default: not supported.
    async fn purge_namespace(&self, _namespace: &str) -> anyhow::Result<usize> {
        anyhow::bail!("purge_namespace not supported by this memory backend")
    }

    /// Delete all memories for a session. Default: not supported.
    async fn purge_session(&self, _session_id: &str) -> anyhow::Result<usize> {
        anyhow::bail!("purge_session not supported by this memory backend")
    }

    // ── Stats & Health ────────────────────────────────────────

    /// REQUIRED. Total number of stored memories.
    async fn count(&self) -> anyhow::Result<usize>;

    /// REQUIRED. Check if the backend is healthy.
    async fn health_check(&self) -> bool;

    // ── Advanced operations ───────────────────────────────────

    /// Store procedural (multi-turn) conversation messages.
    /// Default: no-op.
    async fn store_procedural(
        &self,
        _messages: &[ProceduralMessage],
        _session_id: Option<&str>,
    ) -> anyhow::Result<()> { Ok(()) }

    /// Recall within a specific namespace.
    /// Default: filters recall() results by namespace.
    async fn recall_namespaced(
        &self,
        namespace: &str,
        query: &str,
        limit: usize,
        session_id: Option<&str>,
        since: Option<&str>,
        until: Option<&str>,
    ) -> anyhow::Result<Vec<MemoryEntry>>;

    /// Export memories matching a filter (GDPR data portability).
    /// Default: filters list() results.
    async fn export(
        &self,
        filter: &ExportFilter,
    ) -> anyhow::Result<Vec<MemoryEntry>>;

    /// Store with extended metadata (namespace, importance).
    /// Default: delegates to store().
    async fn store_with_metadata(
        &self,
        key: &str,
        content: &str,
        category: MemoryCategory,
        session_id: Option<&str>,
        _namespace: Option<&str>,
        _importance: Option<f64>,
    ) -> anyhow::Result<()>;
}
```

### Required vs. Default Methods

| Method | Required? | Notes |
|--------|-----------|-------|
| `name()` | **Yes** | Backend identifier |
| `store()` | **Yes** | Primary write path |
| `recall()` | **Yes** | Semantic search — the most critical method |
| `get()` | **Yes** | Exact key lookup |
| `list()` | **Yes** | Filtered listing |
| `forget()` | **Yes** | Delete by key |
| `count()` | **Yes** | Total entry count |
| `health_check()` | **Yes** | Liveness check |
| `purge_namespace()` | No | Default: returns error (not supported) |
| `purge_session()` | No | Default: returns error (not supported) |
| `store_procedural()` | No | Default: no-op |
| `recall_namespaced()` | No* | Has default that filters recall() results |
| `export()` | No* | Has default that filters list() results |
| `store_with_metadata()` | No* | Has default that delegates to store() |

*These have defaults but benefit from native implementations for performance.

---

## Associated Types

### `MemoryEntry`

A single stored memory:

```rust
pub struct MemoryEntry {
    pub id: String,
    pub key: String,                        // Unique identifier
    pub content: String,                    // The actual memory content
    pub category: MemoryCategory,
    pub timestamp: String,                  // ISO 8601
    pub session_id: Option<String>,        // Which session created this
    pub score: Option<f64>,                // Relevance score from recall
    pub namespace: Option<String>,         // Logical grouping
    pub importance: Option<f64>,           // Priority weight (0.0-1.0)
    pub superseded_by: Option<String>,     // If updated by a newer entry
}
```

### `MemoryCategory`

Categories control how memories are treated:

```rust
pub enum MemoryCategory {
    Core,              // Evergreen facts — never time-decayed
    Daily,             // Per-day summaries — decayed over weeks
    Conversation,      // Per-session context — decayed fastest
    Custom(String),    // User-defined categories
}
```

Serializes as snake_case strings: `"core"`, `"daily"`, `"conversation"`, `"my_category"`.

### `ProceduralMessage`

Stored conversation fragments:

```rust
pub struct ProceduralMessage {
    pub role: String,              // "user", "assistant", "system"
    pub content: String,
    pub name: Option<String>,      // Optional speaker name
}
```

### `ExportFilter`

Filter criteria for GDPR data export:

```rust
pub struct ExportFilter {
    pub namespace: Option<String>,
    pub session_id: Option<String>,
    pub category: Option<MemoryCategory>,
    pub since: Option<String>,     // ISO 8601 lower bound
    pub until: Option<String>,     // ISO 8601 upper bound
}
```

---

## Existing Implementations

| Implementation | File | Key Characteristics |
|----------------|------|-------------------|
| `SqliteMemory` | `sqlite.rs` | **Primary backend**. Dual-index: vector embeddings (cosine similarity) + FTS5 (keyword search). Embedding cache. Configurable search weights. |
| `MarkdownMemory` | `markdown.rs` | File-based storage in markdown files. Simple, human-readable, no embeddings. Good for debugging. |
| `LucidMemory` | `lucid.rs` | Hybrid backend combining structured storage with semantic search. |
| `QdrantMemory` | `qdrant.rs` | Vector database backend via Qdrant. Pure vector search, best for large-scale deployments. |
| `NoneMemory` | `none.rs` | No-op backend. All operations succeed silently. Used when memory is disabled. |

### SqliteMemory — The Default Backend

SqliteMemory is the most feature-complete backend and the default choice:

```
┌─────────────────────────────────────────────────────┐
│  SqliteMemory                                       │
│                                                     │
│  ┌─────────────────┐  ┌─────────────────────────┐   │
│  │ Vector Index     │  │ FTS5 Keyword Index      │   │
│  │ (embeddings)     │  │ (SQLite full-text)      │   │
│  │                  │  │                          │   │
│  │ cosine_sim(      │  │ MATCH query              │   │
│  │   query_embed,   │  │ rank by BM25             │   │
│  │   stored_embed   │  │                          │   │
│  │ )                │  │                          │   │
│  └────────┬─────────┘  └──────────┬───────────────┘   │
│           │                        │                   │
│           └──────────┬─────────────┘                   │
│                      ▼                                 │
│  ┌─────────────────────────────────────────────────┐   │
│  │ Merge & Rank                                    │   │
│  │ score = (vector_weight * vec_score)             │   │
│  │       + (keyword_weight * kw_score)             │   │
│  │       + time_decay_factor                       │   │
│  │       + importance_boost                        │   │
│  └─────────────────────────────────────────────────┘   │
│                                                     │
│  Embedding cache: in-memory LRU for recent queries    │
└─────────────────────────────────────────────────────┘
```

---

## Supporting Modules

The memory subsystem includes several supporting modules beyond the backends:

| File | Purpose |
|------|---------|
| `embeddings.rs` | Embedding generation via configured provider (OpenAI, local, etc.) |
| `vector.rs` | Vector operations (cosine similarity, normalization) |
| `chunker.rs` | Text chunking for large documents before embedding |
| `retrieval.rs` | High-level retrieval orchestration (combines multiple search strategies) |
| `consolidation.rs` | Consolidate/summarize related memories to stay within limits |
| `decay.rs` | Time-based decay functions for memory scoring |
| `importance.rs` | Importance scoring heuristics |
| `hygiene.rs` | Memory hygiene — dedup, supersession, cleanup |
| `conflict.rs` | Conflict resolution when memories contradict |
| `snapshot.rs` | Memory snapshot/restore for backup |
| `policy.rs` | Memory retention policies |
| `knowledge_graph.rs` | Graph-based memory relationships |
| `response_cache.rs` | Cache recent LLM responses to avoid re-computation |
| `audit.rs` | Memory access audit logging |
| `backend.rs` | Backend abstraction utilities |
| `battle_tests.rs` | Comprehensive backend conformance tests |
| `cli.rs` | CLI commands for memory management |

---

## Factory / Registration

**Location**: `src/memory/mod.rs`

```rust
pub async fn create_memory(config: &MemoryConfig) -> anyhow::Result<Arc<dyn Memory>> {
    match config.backend.as_str() {
        "sqlite" => {
            let mem = SqliteMemory::new(&config.sqlite)?;
            mem.initialize().await?;
            Ok(Arc::new(mem))
        }
        "markdown" => {
            Ok(Arc::new(MarkdownMemory::new(&config.markdown_dir)?))
        }
        "lucid" => {
            Ok(Arc::new(LucidMemory::new(&config.lucid)?))
        }
        "qdrant" => {
            Ok(Arc::new(QdrantMemory::new(&config.qdrant)?))
        }
        "none" => {
            Ok(Arc::new(NoneMemory))
        }
        _ => anyhow::bail!("Unknown memory backend: {}", config.backend),
    }
}
```

Note: Memory is wrapped in `Arc<dyn Memory>` because it's shared across the agent, tools (memory_store, memory_recall, etc.), and the gateway.

---

## Configuration

```toml
[memory]
backend = "sqlite"          # "sqlite", "markdown", "lucid", "qdrant", "none"

[memory.sqlite]
path = "~/.zeroclaw/workspace/memory.db"
vector_weight = 0.7         # Weight for vector similarity in hybrid search
keyword_weight = 0.3        # Weight for FTS5 keyword matching
cache_size = 1000           # Embedding cache entries

[memory.markdown]
dir = "~/.zeroclaw/workspace/memory/"

[memory.qdrant]
url = "http://localhost:6333"
collection = "zeroclaw_memory"

# Embedding config (used by vector-capable backends)
[memory.embeddings]
provider = "openai"             # or "ollama", "local"
model = "text-embedding-3-small"
dimensions = 1536
```

---

## Extension Guide — Adding a New Memory Backend

### Step 1: Create the implementation file

Create `src/memory/my_backend.rs`:

```rust
use async_trait::async_trait;
use crate::memory::traits::*;

pub struct MyBackendMemory {
    // Your storage client
    client: MyDbClient,
}

impl MyBackendMemory {
    pub async fn new(config: &MyBackendConfig) -> anyhow::Result<Self> {
        let client = MyDbClient::connect(&config.url).await?;
        Ok(Self { client })
    }
}

#[async_trait]
impl Memory for MyBackendMemory {
    fn name(&self) -> &str {
        "mybackend"
    }

    async fn store(
        &self,
        key: &str,
        content: &str,
        category: MemoryCategory,
        session_id: Option<&str>,
    ) -> anyhow::Result<()> {
        // Insert into your storage
        // Generate embedding if your backend supports vector search
        // Store category, session_id, timestamp
        todo!()
    }

    async fn recall(
        &self,
        query: &str,
        limit: usize,
        session_id: Option<&str>,
        since: Option<&str>,
        until: Option<&str>,
    ) -> anyhow::Result<Vec<MemoryEntry>> {
        // This is the critical method — semantic search
        // Generate embedding for query
        // Search your vector index
        // Optionally combine with keyword search
        // Apply time filters (since/until)
        // Apply session filter
        // Return ranked results
        todo!()
    }

    async fn get(&self, key: &str) -> anyhow::Result<Option<MemoryEntry>> {
        // Exact key lookup
        todo!()
    }

    async fn list(
        &self,
        category: Option<&MemoryCategory>,
        session_id: Option<&str>,
    ) -> anyhow::Result<Vec<MemoryEntry>> {
        // List with optional filters
        todo!()
    }

    async fn forget(&self, key: &str) -> anyhow::Result<bool> {
        // Delete by key
        todo!()
    }

    async fn count(&self) -> anyhow::Result<usize> {
        // COUNT(*) or equivalent
        todo!()
    }

    async fn health_check(&self) -> bool {
        // Ping your storage backend
        self.client.ping().await.is_ok()
    }

    // Recommended: Override these for better performance
    async fn purge_namespace(&self, namespace: &str) -> anyhow::Result<usize> {
        // DELETE WHERE namespace = ?
        todo!()
    }

    async fn purge_session(&self, session_id: &str) -> anyhow::Result<usize> {
        // DELETE WHERE session_id = ?
        todo!()
    }

    async fn store_with_metadata(
        &self,
        key: &str,
        content: &str,
        category: MemoryCategory,
        session_id: Option<&str>,
        namespace: Option<&str>,
        importance: Option<f64>,
    ) -> anyhow::Result<()> {
        // Store with full metadata (better than default delegation)
        todo!()
    }
}
```

### Step 2: Register the module

In `src/memory/mod.rs`:

```rust
mod my_backend;
pub use my_backend::MyBackendMemory;
```

### Step 3: Add to factory

```rust
"mybackend" => {
    let mem = MyBackendMemory::new(&config.mybackend).await?;
    Ok(Arc::new(mem))
}
```

### Step 4: Add config

In `src/config/schema.rs`:

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct MyBackendConfig {
    pub url: String,
    pub collection: Option<String>,
}
```

### Step 5: Run conformance tests

Use the `battle_tests.rs` test suite to verify your backend:

```bash
cargo test --lib memory::battle_tests -- --test-threads=1
```

---

## Testing Your Memory Backend

The `battle_tests.rs` module provides a conformance test suite that exercises all trait methods. Key scenarios:

1. **Store → Get** — `store("key", "content", Core, None)` then `get("key")` returns the entry.
2. **Store → Recall** — `store("user likes rust", ...)` then `recall("programming language", 5, ...)` returns the entry.
3. **Store → Forget** — `forget("key")` returns `true`, subsequent `get("key")` returns `None`.
4. **List with filters** — `list(Some(Core), None)` returns only Core entries.
5. **Count** — Matches number of stored entries.
6. **Time filtering** — `recall(query, limit, None, Some("2025-01-01"), Some("2025-12-31"))`.
7. **Session isolation** — Entries stored with session_id = "A" are not returned for session_id = "B".
8. **Namespace isolation** — `recall_namespaced("ns1", ...)` returns only "ns1" entries.
9. **Health check** — Returns true when backend is connected, false when down.
10. **Export** — `export(ExportFilter { ... })` returns matching entries.

---

## Common Patterns & Gotchas

1. **`recall()` is the hot path**: This method is called every turn to inject context. It must be fast (sub-100ms ideally). Use indexes, caching, and limit result sizes.

2. **Embedding generation is expensive**: If your backend uses vector search, batch embedding generation where possible. Use the embedding cache (`src/memory/embeddings.rs`) to avoid re-embedding the same queries.

3. **`MemoryCategory::Core` should never decay**: Core memories are facts the agent should always remember. Don't apply time decay to Core entries in your `recall()` ranking.

4. **`superseded_by` for contradiction resolution**: When a new memory contradicts an old one (e.g., "user prefers Python" supersedes "user prefers Java"), set `superseded_by` on the old entry. The recall logic should deprioritize superseded entries.

5. **Thread safety**: Memory is shared via `Arc<dyn Memory>` across multiple concurrent tasks. Your implementation must handle concurrent reads and writes safely.

6. **GDPR compliance**: The `export()` and `purge_*()` methods exist for data portability and right-to-erasure. Implement them properly if your backend stores personal data.

7. **Migration from defaults**: Users may switch backends. Consider supporting import from the default SQLite format.

---

*[← Tools](../tools/) | [Observability →](../observability/)*
