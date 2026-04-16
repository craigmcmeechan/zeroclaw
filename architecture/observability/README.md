# Observability Trait — Architecture Deep-Dive

> **Source**: `src/observability/traits.rs`
> **Factory**: `src/observability/mod.rs`
> **Parent doc**: [Architecture Overview](../overview.md)

---

## Purpose

The `Observer` trait provides a synchronous, fire-and-forget event/metric recording interface. Every significant system event — LLM requests, tool invocations, agent lifecycle, errors — is pushed through registered observers. This enables logging, metrics collection, tracing, and alerting without coupling business logic to any specific monitoring backend.

Multiple observers can be active simultaneously via `MultiObserver`, which fans out each event to all registered backends.

**When to implement**: You want to add a new monitoring backend (e.g., Datadog, New Relic, custom dashboards, file-based audit trails).

---

## Trait Definition

```rust
pub trait Observer: Send + Sync + 'static {
    // ── Core ──────────────────────────────────────────────────

    /// Backend name (e.g., "log", "otel", "prometheus").
    fn name(&self) -> &str;

    /// Record a structured event.
    fn record_event(&self, event: &ObserverEvent);

    /// Record a numeric metric.
    fn record_metric(&self, metric: &ObserverMetric);

    // ── Lifecycle ─────────────────────────────────────────────

    /// Flush buffered data. Default: no-op.
    fn flush(&self) {}

    // ── Runtime reflection ────────────────────────────────────

    /// Downcast support for runtime type inspection.
    fn as_any(&self) -> &dyn std::any::Any;
}
```

**Key design choice**: This trait is **not** `async_trait`. Recording events must never block the hot path. Implementations that need I/O (e.g., HTTP export) should buffer internally and flush asynchronously.

### Method Summary

| Method | Required? | Notes |
|--------|-----------|-------|
| `name()` | **Yes** | Backend identifier |
| `record_event()` | **Yes** | All structured events |
| `record_metric()` | **Yes** | All numeric metrics |
| `flush()` | No | Default no-op; override for buffered backends |
| `as_any()` | **Yes** | Enables runtime downcasting |

---

## Event Types — `ObserverEvent`

The `ObserverEvent` enum captures all observable system events:

```rust
pub enum ObserverEvent {
    // ── Agent lifecycle ───────────────────────────────────────
    AgentStart { session_id: String },
    AgentEnd { session_id: String },
    TurnComplete { session_id: String, turn: usize },

    // ── LLM interactions ──────────────────────────────────────
    LlmRequest { provider: String, model: String, tokens: Option<usize> },
    LlmResponse { provider: String, model: String, latency_ms: u64, tokens_used: Option<usize> },
    CacheHit { key: String },
    CacheMiss { key: String },

    // ── Tool execution ────────────────────────────────────────
    ToolCallStart { tool: String, session_id: Option<String> },
    ToolCall { tool: String, success: bool, latency_ms: u64 },

    // ── Channel events ────────────────────────────────────────
    ChannelMessage { channel: String, direction: String },

    // ── Background tasks ──────────────────────────────────────
    HeartbeatTick { timestamp: String },

    // ── Hands (deployment/recovery) ───────────────────────────
    HandStarted { hand: String },
    HandCompleted { hand: String, success: bool },
    HandFailed { hand: String, error: String },
    DeploymentStarted { target: String },
    DeploymentCompleted { target: String, success: bool },
    DeploymentFailed { target: String, error: String },
    RecoveryCompleted { strategy: String },

    // ── Errors ────────────────────────────────────────────────
    Error { message: String, context: Option<String> },
}
```

This covers **18 distinct event variants** across 6 categories.

---

## Metric Types — `ObserverMetric`

```rust
pub enum ObserverMetric {
    LlmLatency { provider: String, model: String, ms: u64 },
    ToolLatency { tool: String, ms: u64 },
    TokensUsed { provider: String, model: String, count: usize },
    MemoryCount { backend: String, count: usize },
    TurnDuration { session_id: String, ms: u64 },
    QueueDepth { channel: String, depth: usize },
    ErrorCount { context: String, count: usize },
    CacheHitRate { hits: usize, misses: usize },
    ActiveSessions { count: usize },
    CustomMetric { name: String, value: f64, labels: Vec<(String, String)> },
}
```

**10 metric variants** covering latency, throughput, resource usage, and custom extensions.

---

## Existing Implementations

| Implementation | File | Feature Gate | Key Characteristics |
|----------------|------|--------------|-------------------|
| `LogObserver` | `mod.rs` | None | Logs events at `info`/`debug` via `tracing`. Default backend. Zero overhead beyond logging. |
| `VerboseObserver` | `mod.rs` | None | Logs everything at `debug` level with full detail. For development. |
| `NoopObserver` | `mod.rs` | None | Silently discards all events. Minimal overhead. |
| `OtelObserver` | `otel.rs` | `observability-otel` | OpenTelemetry integration. Exports traces and metrics via OTLP. |
| `PrometheusObserver` | `prometheus.rs` | `observability-prometheus` | Exposes Prometheus `/metrics` endpoint. Counters, histograms, gauges. |

### `MultiObserver` — Fan-Out Combiner

`MultiObserver` (in `multi.rs`) wraps `Vec<Arc<dyn Observer>>` and dispatches each event/metric to all registered backends:

```
record_event(ToolCall)
        │
        ▼
   MultiObserver
        │
  ┌─────┼─────────────────┐
  ▼     ▼                  ▼
 Log   OTel          Prometheus
```

This is the default usage pattern — the agent constructs a `MultiObserver` containing all configured backends.

### Supporting Modules

| File | Purpose |
|------|---------|
| `dora.rs` | Runtime diagnostics and observability aware restart/detection |
| `multi.rs` | `MultiObserver` fan-out combiner |
| `runtime_trace.rs` | Runtime-level trace instrumentation |

---

## Factory / Registration

**Location**: `src/observability/mod.rs`

Observers are created from config and collected into a `MultiObserver`:

```rust
pub fn create_observers(config: &ObservabilityConfig) -> Vec<Arc<dyn Observer>> {
    let mut observers: Vec<Arc<dyn Observer>> = Vec::new();

    // Log observer is always included unless explicitly disabled
    if config.log_enabled != Some(false) {
        observers.push(Arc::new(LogObserver));
    }

    #[cfg(feature = "observability-otel")]
    if config.otel.is_some() {
        observers.push(Arc::new(OtelObserver::new(&config.otel.unwrap())));
    }

    #[cfg(feature = "observability-prometheus")]
    if config.prometheus.is_some() {
        observers.push(Arc::new(PrometheusObserver::new(&config.prometheus.unwrap())));
    }

    observers
}
```

---

## Configuration

```toml
[observability]
log_enabled = true              # LogObserver (default: true)
verbose = false                 # Use VerboseObserver instead of LogObserver

# OpenTelemetry (requires feature: observability-otel)
[observability.otel]
endpoint = "http://localhost:4317"    # OTLP gRPC endpoint
service_name = "zeroclaw"
sample_rate = 1.0

# Prometheus (requires feature: observability-prometheus)
[observability.prometheus]
bind = "0.0.0.0:9090"                # Metrics endpoint
path = "/metrics"
```

---

## Extension Guide — Adding a New Observer

### Step 1: Create the implementation file

Create `src/observability/my_observer.rs`:

```rust
use crate::observability::traits::*;
use std::any::Any;

pub struct MyObserver {
    client: MyMetricsClient,
    buffer: std::sync::Mutex<Vec<BufferedEvent>>,
}

impl MyObserver {
    pub fn new(config: &MyObserverConfig) -> Self {
        Self {
            client: MyMetricsClient::connect(&config.endpoint),
            buffer: std::sync::Mutex::new(Vec::new()),
        }
    }
}

impl Observer for MyObserver {
    fn name(&self) -> &str {
        "my_observer"
    }

    fn record_event(&self, event: &ObserverEvent) {
        // IMPORTANT: This must NOT block. Buffer internally.
        let mut buf = self.buffer.lock().unwrap();
        buf.push(event.into());
        // Optionally flush when buffer is full
        if buf.len() >= 100 {
            // async flush via spawned task
        }
    }

    fn record_metric(&self, metric: &ObserverMetric) {
        // Convert to your metrics format and buffer
        match metric {
            ObserverMetric::LlmLatency { provider, model, ms } => {
                // Record histogram
            }
            ObserverMetric::TokensUsed { count, .. } => {
                // Record counter
            }
            _ => {}
        }
    }

    fn flush(&self) {
        // Drain buffer and send to backend
        let mut buf = self.buffer.lock().unwrap();
        let events: Vec<_> = buf.drain(..).collect();
        // self.client.send_batch(events);
    }

    fn as_any(&self) -> &dyn Any {
        self
    }
}
```

### Step 2: Register and add to factory

In `src/observability/mod.rs`:

```rust
mod my_observer;
pub use my_observer::MyObserver;

// In create_observers():
if config.my_observer.is_some() {
    observers.push(Arc::new(MyObserver::new(&config.my_observer.unwrap())));
}
```

### Step 3: Add feature gate (optional)

If your observer adds heavy dependencies, gate it behind a Cargo feature:

```toml
# In Cargo.toml
[features]
observability-mybackend = ["dep:my-metrics-lib"]
```

Wrap the registration with `#[cfg(feature = "observability-mybackend")]`.

---

## Common Patterns & Gotchas

1. **Never block in `record_event()` / `record_metric()`**: These are called inline during the agent's hot path. Any I/O must be buffered and flushed asynchronously. Use `Mutex<Vec<_>>` for buffering and flush via `tokio::spawn`.

2. **`as_any()` is required for downcasting**: The runtime uses this to check if a specific observer type is present (e.g., to access the Prometheus metrics endpoint directly).

3. **Event types are exhaustive — use `_` pattern**: When matching `ObserverEvent` variants, always include a catch-all arm. New variants may be added.

4. **`flush()` is called on shutdown**: The agent calls `flush()` on all observers before exiting. Ensure your buffered data is persisted.

5. **Feature-gated observers must compile cleanly when disabled**: Ensure `#[cfg(feature = "...")]` is applied consistently on the module declaration, use statements, and factory registration.

6. **Metric labels should be low-cardinality**: Avoid putting high-cardinality values (session IDs, user IDs) as metric labels—this causes cardinality explosion in Prometheus/OTLP.

---

*[← Memory](../memory/) | [Runtime →](../runtime/)*
