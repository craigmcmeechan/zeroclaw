# ZeroClaw — Architectural Overview

## 1. Design Philosophy

ZeroClaw is a **Rust-first autonomous agent runtime** optimized for six pillars: **performance, efficiency, stability, extensibility, sustainability, and security**. Every major subsystem is abstracted behind a trait, and every concrete implementation is registered through a factory function driven by configuration. This design makes it possible to:

- Swap LLM providers, messaging platforms, or storage backends without touching orchestration code.
- Add new capabilities (a tool, a channel, a hardware board) by implementing a single trait and adding one registration line.
- Run the same agent binary across radically different environments (bare-metal Linux, Docker container, future Wasm edge) by switching the `RuntimeAdapter`.

The codebase uses **`async_trait`** for all I/O-bound traits, **`anyhow::Result`** for error propagation, and **`Arc<dyn Trait>`** for shared concurrent access to stateful subsystems (memory, observer, provider).

---

## 2. High-Level Component Diagram

```
┌──────────────────────────────────────────────────────────────────────┐
│                        CLI / Daemon / Gateway                        │
│  src/main.rs ─ Clap command routing                                  │
│  src/daemon/  ─ Long-running mode (channels + heartbeat + cron)      │
│  src/gateway/ ─ Axum HTTP/WS server (webhooks, REST API, pairing)    │
└──────────────┬───────────────────────────────────────┬───────────────┘
               │                                       │
               ▼                                       ▼
┌──────────────────────────┐             ┌──────────────────────────┐
│     Agent Orchestrator   │             │     Channel Listeners    │
│  src/agent/agent.rs      │◄────────────│  src/channels/           │
│  src/agent/loop_.rs      │  inbound    │  Telegram, Discord,      │
│  src/agent/dispatcher.rs │  messages   │  Slack, Signal, Email,   │
└──────┬───────────────────┘             │  IRC, Matrix, …40+      │
       │                                 └──────────────────────────┘
       │  requests
       ▼
┌──────────────────────────┐
│     LLM Provider         │
│  src/providers/           │
│  OpenAI, Anthropic,       │
│  Ollama, Azure, Bedrock,  │
│  OpenRouter, …20+         │
│  + ReliableProvider       │
│    (retry/fallback)       │
│  + RouterProvider         │
│    (model routing)        │
└──────┬───────────────────┘
       │  tool calls
       ▼
┌──────────────────────────┐      ┌──────────────────────────┐
│     Tool Execution       │      │     Security Policy      │
│  src/tools/               │◄────│  src/security/            │
│  Shell, File, Glob,       │ gate│  AutonomyLevel, Sandbox,  │
│  Browser, WebFetch,       │     │  Approval, E-Stop, OTP,   │
│  Memory ops, Cron,        │     │  PairingGuard, AuditLog   │
│  Notion, …100+            │     └──────────────────────────┘
└──────┬───────────────────┘
       │  read/write                ┌──────────────────────────┐
       ▼                            │     Observability        │
┌──────────────────────────┐        │  src/observability/       │
│     Memory Backend       │        │  Log, Verbose, Noop,      │
│  src/memory/              │        │  OpenTelemetry,           │
│  Sqlite (vec+FTS5),       │        │  Prometheus               │
│  Markdown, Lucid,         │        └──────────────────────────┘
│  Qdrant, None             │
└──────────────────────────┘        ┌──────────────────────────┐
                                    │     Runtime Adapter      │
┌──────────────────────────┐        │  src/runtime/             │
│     Peripherals          │        │  Native, Docker, Wasm    │
│  src/peripherals/         │        └──────────────────────────┘
│  Nucleo F401RE (STM32),   │
│  Arduino, RPi GPIO        │
└──────────────────────────┘
```

---

## 3. Module Map

Every public module exported from `src/lib.rs`, grouped by function:

### Core Orchestration

| Module | Path | Purpose |
|--------|------|---------|
| `agent` | `src/agent/` | Orchestration loop — `Agent` struct, `turn_streamed()`, history, prompt building, tool dispatching, loop detection |
| `commands` | `src/commands/` | Shared CLI command enums re-exported from `lib.rs` |
| `config` | `src/config/` | Configuration schema (`Config` struct, 6000+ line `schema.rs`), loading, merging, env overrides |
| `gateway` | `src/gateway/` | Axum HTTP/WS server — REST API, WebSocket chat, pairing, rate limiting, SSE, static files, TLS |
| `daemon` | `src/daemon/` | Long-running mode combining gateway + channels + heartbeat + cron with graceful shutdown |

### Extension Traits (the seven pillars)

| Module | Path | Trait | Purpose |
|--------|------|-------|---------|
| `providers` | `src/providers/` | `Provider` | LLM interaction — chat, streaming, tool calling |
| `channels` | `src/channels/` | `Channel` | Bidirectional messaging platforms |
| `tools` | `src/tools/` | `Tool` | Agent-callable actions |
| `memory` | `src/memory/` | `Memory` | Long-term knowledge storage and recall |
| `observability` | `src/observability/` | `Observer` | Event/metric recording |
| `runtime` | `src/runtime/` | `RuntimeAdapter` | Platform abstraction |
| `peripherals` | `src/peripherals/` | `Peripheral` | Hardware boards as agent tools |

### Security & Trust

| Module | Path | Purpose |
|--------|------|---------|
| `security` | `src/security/` | SecurityPolicy, Sandbox trait, PairingGuard, SecretStore, AuditLogger, E-Stop, OTP, WebAuthn, prompt guard, leak detector, workspace boundaries |
| `approval` | `src/approval/` | Interactive tool-call approval workflow for supervised mode |
| `trust` | `src/trust/` | Trust model and permission boundary evaluation |
| `auth` | `src/auth/` | OAuth login/refresh/logout flows |
| `verifiable_intent` | `src/verifiable_intent/` | SD-JWT layered credentials for constraint-based commerce gating |

### Automation & Scheduling

| Module | Path | Purpose |
|--------|------|---------|
| `cron` | `src/cron/` | Cron job scheduler with SQL-backed store, timezone-aware scheduling |
| `hooks` | `src/hooks/` | Lifecycle event hooks (before/after tool execution, messages) |
| `routines` | `src/routines/` | Event-pattern automation rules loaded from `routines.toml` |
| `sop` | `src/sop/` | Standard Operating Procedures engine — multi-step workflow execution |
| `hands` | `src/hands/` | Named, isolated agent instances with rolling context |

### Capabilities & Integrations

| Module | Path | Purpose |
|--------|------|---------|
| `skills` | `src/skills/` | Community/user-defined capability bundles (tools + prompts) |
| `integrations` | `src/integrations/` | Third-party service registry (status, categories) |
| `rag` | `src/rag/` | RAG pipeline for hardware datasheets (markdown, text, optional PDF) |
| `multimodal` | `src/multimodal.rs` | Reserved — multi-modal processing (not yet implemented) |
| `nodes` | `src/nodes/` | Experimental multi-node agent coordination transport |
| `tunnel` | `src/tunnel/` | Tunnel providers (Cloudflare, ngrok, Tailscale, OpenVPN, Pinggy, custom) |

### Operational

| Module | Path | Purpose |
|--------|------|---------|
| `health` | `src/health/` | Component health registry (running/degraded/error, uptime, restarts) |
| `heartbeat` | `src/heartbeat/` | Periodic liveness — writes `HEARTBEAT.md` with agent stats |
| `cost` | `src/cost/` | Token/cost tracking with per-model accounting and budget enforcement |
| `doctor` | `src/doctor/` | Diagnostic probes — model reachability, trace inspection |
| `service` | `src/service/` | OS service management (systemd/launchd install/start/stop/status) |
| `migration` | `src/migration.rs` | Database/storage migration logic |
| `onboard` | `src/onboard/` | First-run onboarding wizard |
| `i18n` | `src/i18n.rs` | Internationalization (locale detection, translation keys) |
| `identity` | `src/identity.rs` | Identity management (planned) |
| `cli_input` | `src/cli_input.rs` | Terminal input handling for interactive mode |
| `util` | `src/util.rs` | Shared utility functions |
| `hardware` | `src/hardware/` | USB discovery and device introspection |
| `plugins` | `src/plugins/` | Wasm plugin runtime (feature-gated: `plugins-wasm`) |

---

## 4. Extension Point Summary

Every trait follows the same pattern: **trait definition → factory function → config-driven selection**.

| Trait | Trait File | Factory Location | Registration Pattern | Implementation Count |
|-------|-----------|-----------------|---------------------|---------------------|
| `Provider` | `src/providers/traits.rs` | `src/providers/mod.rs` | `match` on provider name string + credential resolution | 20+ (plus 50+ OpenAI-compatible) |
| `Channel` | `src/channels/traits.rs` | `src/channels/mod.rs` | Config-driven explicit instantiation per enabled channel | 40+ |
| `Tool` | `src/tools/traits.rs` | `src/tools/mod.rs` | Conditional `Vec` assembly based on security/config | 100+ |
| `Memory` | `src/memory/traits.rs` | `src/memory/mod.rs` | `match` on `config.memory.backend` string | 5 |
| `Observer` | `src/observability/traits.rs` | `src/observability/mod.rs` | `match` on `config.observability.backend` string | 5 |
| `RuntimeAdapter` | `src/runtime/traits.rs` | `src/runtime/mod.rs` | `match` on `config.runtime.kind` string | 3 |
| `Peripheral` | `src/peripherals/traits.rs` | `src/peripherals/mod.rs` | Config-driven list iteration | 3+ |

**To add a new implementation**, the steps are always:

1. Create `src/{module}/{your_impl}.rs` implementing the trait.
2. Add `mod your_impl;` to `src/{module}/mod.rs`.
3. Add a match arm / config branch in the factory function.
4. Add any required config fields to `src/config/schema.rs`.
5. Write tests.

See each trait's subfolder for the detailed step-by-step guide.

---

## 5. Configuration Cascade

Configuration resolves in this precedence order (highest first):

```
1. Environment variables     ZEROCLAW_API_KEY, ZEROCLAW_WORKSPACE, …
2. CLI flags                 --provider, --model, --temperature, --config-dir
3. config.toml               [default_provider], [gateway], [memory], …
4. Compiled defaults         Hardcoded in schema.rs via #[serde(default)]
```

### Key Environment Variables

| Variable | Purpose |
|----------|---------|
| `ZEROCLAW_API_KEY` | Default API key (overrides `config.api_key`) |
| `ZEROCLAW_WORKSPACE` | Workspace root directory |
| `ZEROCLAW_GATEWAY_TIMEOUT_SECS` | Gateway request timeout override |
| `ZEROCLAW_EXTRA_HEADERS` | Extra HTTP headers (CSV format) |
| `ZEROCLAW_LOCALE` / `LANG` / `LC_ALL` | Locale detection for i18n |

### Config File Location

```
~/.zeroclaw/config.toml            # Primary config
~/.zeroclaw/active_workspace.toml  # Workspace pointer
$ZEROCLAW_WORKSPACE/config.toml    # Workspace-local config (merged)
```

The full config schema can be exported as JSON Schema: `zeroclaw config export-schema`.

---

## 6. Security Architecture

ZeroClaw has a layered security model that constrains agent behavior at every level.

### 6.1 Autonomy Levels

```
ReadOnly       Agent can observe but never execute side effects.
Supervised     Agent can act, but medium+ risk operations require user approval.
Full           Agent operates autonomously within policy bounds.
```

The autonomy level is set in `config.toml` under `[autonomy]` and enforced by `SecurityPolicy` throughout the tool execution loop.

### 6.2 Command Risk Classification

| Risk Level | Examples | Behavior |
|------------|----------|----------|
| **Low** | `cat`, `ls`, `grep`, `find`, `echo` | Always allowed |
| **Medium** | `npm run test`, `cargo build`, `git commit` | Requires approval in Supervised mode |
| **High** | `rm -rf /`, `passwd`, `sudo`, `chmod 777` | Blocked when `block_high_risk_commands = true` |

### 6.3 Sandbox Backends

Shell commands execute inside a sandbox. The backend is auto-detected or configured explicitly:

| Backend | Platform | Isolation Level |
|---------|----------|----------------|
| Docker | Linux, macOS (Desktop), Windows (WSL) | Full process + network isolation |
| Firejail | Linux | seccomp + AppArmor (no root needed) |
| Bubblewrap | Linux | User-namespaces (Flatpak's sandbox) |
| Landlock | Linux 5.13+ | LSM rules (no container overhead) |
| Seatbelt | macOS | MACF rules |
| Noop | Any (fallback) | No isolation — runs in agent process |

All sandbox backends implement the `Sandbox` trait in `src/security/traits.rs`.

### 6.4 Pairing & Authentication

The gateway supports a multi-tier authentication model:

1. **No pairing** (default for local-only): Any client can call APIs.
2. **Pairing code flow**: Operator generates an 8-char code (`zeroclaw gateway get-paircode`), client submits it via `POST /pair`, receives a long-lived bearer token.
3. **Bearer token auth**: All subsequent API/WS requests include `Authorization: Bearer <token>`.
4. **WebAuthn/FIDO2** (optional): Hardware key authentication.

### 6.5 Additional Security Layers

| Layer | Module | Purpose |
|-------|--------|---------|
| Approval gates | `src/approval/` | Interactive pre-execution approval in supervised mode |
| Action tracker | `SecurityPolicy.tracker` | Sliding-window rate limit (max actions per hour) |
| Budget enforcement | `src/cost/` | `max_cost_per_day_cents` — hard stop on spending |
| Credential scrubbing | `src/agent/agent.rs` | Redacts API keys/passwords from tool output before sending to LLM |
| Prompt guard | `src/security/prompt_guard.rs` | Prompt injection defense |
| Leak detector | `src/security/leak_detector.rs` | Detects credential emission in agent output |
| Workspace boundary | `src/security/workspace_boundary.rs` | Restricts file access to workspace directory |
| Domain matcher | `src/security/domain_matcher.rs` | URL allowlist/blocklist for web tools |
| E-Stop | `src/security/estop.rs` | Emergency stop (kill-all, network-kill, domain-block, tool-freeze) |
| Audit logger | `src/security/audit.rs` | Security event logging to disk |

---

## 7. Build System & Features

ZeroClaw uses Cargo features for conditional compilation of optional subsystems:

### Key Feature Flags

| Feature | What It Enables |
|---------|----------------|
| `rag-pdf` | PDF ingestion in the RAG pipeline |
| `voice-wake` | Wake-word detection for voice interfaces |
| `plugins-wasm` | Wasm plugin runtime (`src/plugins/`) |
| `whatsapp-web` | WhatsApp Web channel (wa-rs ecosystem) |
| `webauthn` | WebAuthn/FIDO2 authentication |
| `observability-prometheus` | Prometheus metrics observer |
| `observability-otel` | OpenTelemetry tracing observer |
| `browser-native` | Native browser automation (fantoccini) |
| `sandbox-landlock` | Landlock LSM sandbox backend |
| `sandbox-bubblewrap` | Bubblewrap sandbox backend |
| `ci-all` | Meta-feature enabling everything for CI |

### Build Commands

```bash
cargo build                          # Debug build (default features)
cargo build --release                # Release build
cargo build --features ci-all        # Build with all features
cargo check --all-targets            # Fast type-check
cargo clippy --all-targets -- -D warnings  # Lint
cargo fmt --all -- --check           # Format check
cargo test                           # Run tests
```

### Pre-PR Validation

```bash
./dev/ci.sh all    # Full CI: lint + test + security audit
```

---

## 8. Key Dependencies

| Crate | Version | Role |
|-------|---------|------|
| `tokio` | 1.50 | Async runtime (multi-thread, IO, timers, signals) |
| `axum` | 0.8 | HTTP/WebSocket server for the gateway |
| `reqwest` | 0.12 | HTTP client for LLM API calls |
| `serde` / `serde_json` | 1.x | Serialization throughout |
| `rusqlite` | 0.37 | SQLite for memory, cron, sessions (bundled) |
| `async-trait` | latest | Async trait method support |
| `chacha20poly1305` | 0.10 | AEAD encryption (secret store) |
| `ring` | 0.17 | Cryptographic primitives |
| `rustls` | 0.23 | TLS for gateway and outbound connections |
| `tracing` | latest | Structured logging |
| `prometheus` | 0.14 | Metrics (feature-gated) |
| `opentelemetry` | 0.31 | Distributed tracing (feature-gated) |
| `clap` | latest | CLI argument parsing |
| `toml` | 1.0 | Config file parsing |
| `schemars` | 1.2 | JSON Schema generation for config |
| `chrono` / `cron` | latest | Time handling and cron expressions |
| `tokio-serial` | 5 | Serial port for hardware peripherals |
| `probe-rs` | 0.31 | STM32/Nucleo debug probe (feature-gated) |
| `rppal` | 0.22 | Raspberry Pi GPIO (feature-gated, Linux-only) |
| `fantoccini` | latest | Browser automation (feature-gated) |
| `matrix-sdk` | 0.16 | Matrix E2EE channel (feature-gated) |

---

## 9. Auxiliary Subsystems

These modules are not trait-driven extension points but provide important runtime capabilities.

### 9.1 Hands — Named Agent Instances

**Path**: `src/hands/`

Hands are isolated agent instances with their own rolling context. Each hand loads from a TOML definition in the workspace `hands/` directory and maintains a JSON context file. Used for multi-tenant or multi-purpose agent workflows within the same ZeroClaw runtime.

**Key types**: `Hand`, `HandContext`, `HandRun`, `HandRunStatus`

### 9.2 Hooks — Event-Triggered Automation

**Path**: `src/hooks/`

Lifecycle hooks that fire before/after key operations (tool execution, message processing). Ships with built-in hooks for webhook audit logging and command logging. Extensible via the `HookHandler` trait.

**Key types**: `HookHandler`, `HookResult`, `HookRunner`

### 9.3 Routines — Event-Pattern Automation

**Path**: `src/routines/`

Lightweight automation rules loaded from `routines.toml`. Routes events (webhooks, cron, channel messages) through configurable matchers (exact, glob, regex) and fires actions (SOP triggers, shell commands, messages). Per-routine cooldown prevents rapid re-triggering.

**Key types**: `Routine`, `RoutineAction`, `EventPattern`, `MatchStrategy`, `RoutinesEngine`

### 9.4 SOPs — Standard Operating Procedures

**Path**: `src/sop/`

Multi-step workflow execution engine with four modes: Supervised, Auto, StepByStep, PriorityBased, and Deterministic. Loads SOP manifests from `workspace/sops/`. Includes step condition evaluation, audit logging, and metrics.

**Key types**: `Sop`, `SopRun`, `SopStep`, `SopExecutionMode`, `SopEngine`

### 9.5 Skills — Community Capabilities

**Path**: `src/skills/`

Skills bundle tool definitions, prompts, and automation into installable packages at `~/.zeroclaw/workspace/skills/<name>/SKILL.md`. Supports discovery from ClawhHub registry, OpenClaw public repo, and local creation.

**Key types**: `Skill`, `SkillTool`

### 9.6 Cost Tracking

**Path**: `src/cost/`

Tracks token usage and costs across model invocations. Per-model, per-period statistics with budget enforcement (`max_cost_per_day_cents`). Integrates transparently with the provider wrapper.

**Key types**: `CostTracker`, `CostRecord`, `CostSummary`, `BudgetCheck`

### 9.7 Tunnel Providers

**Path**: `src/tunnel/`

Agnostic tunnel abstraction wrapping multiple providers to expose the local gateway port publicly. The gateway calls `start()`/`stop()` on the active tunnel. Implementations wrap external binaries.

**Implementations**: Cloudflare, ngrok, Tailscale, OpenVPN, Pinggy, Custom, None

### 9.8 Verifiable Intent

**Path**: `src/verifiable_intent/`

Rust-native SD-JWT layered credential system for constraint-based commerce gating. Full lifecycle: credential issuance (L2/L3), chain verification, constraint evaluation, binding integrity.

### 9.9 Cron Scheduler

**Path**: `src/cron/`

Declarative cron job scheduler with SQL-backed store. Standard cron expressions, timezone-aware scheduling, shell and SOP-trigger job types. Validates commands against security policy.

**Key types**: `CronJob`, `Schedule`, `CronRun`, `Scheduler`

### 9.10 Health & Heartbeat

**Path**: `src/health/`, `src/heartbeat/`

Health registry tracks component status (running/degraded/error), uptime, and restart counts. Heartbeat engine periodically writes `HEARTBEAT.md` with liveness, uptime, and memory/cron stats for external monitoring.

---

## 10. Error Handling Strategy

ZeroClaw uses **`anyhow::Result<T>`** throughout — there is no centralized error enum. Errors propagate via context chaining:

```rust
mem.load_embeddings()
    .context("Failed to initialize embeddings")?;
```

This approach favors developer ergonomics and rich error messages over exhaustive pattern matching. For subsystem boundaries (traits), errors are always `anyhow::Result`, letting implementations use whatever internal error types they need while presenting a uniform interface to callers.

---

## 11. Concurrency Model

- **Tokio multi-threaded runtime** — all I/O is async.
- **`Arc<dyn Trait>`** — shared ownership for Memory, Observer, and Provider across concurrent tasks.
- **`tokio::sync::mpsc`** — channels connecting Channel listeners to the Agent via `ChannelMessage`.
- **`SessionQueue`** — serializes concurrent turns for the same session (prevents overlapping LLM calls).
- **Tool parallelism** — the agent can execute independent tool calls in parallel within a single turn, controlled by the dispatcher.

---

*Next: [Data Flow →](data-flow.md)*
