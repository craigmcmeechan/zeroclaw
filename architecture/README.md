# ZeroClaw Architecture Documentation

> **Audience**: Developers extending ZeroClaw — adding providers, channels, tools, memory backends, observers, runtime adapters, or hardware peripherals.

This directory contains in-depth technical documentation for ZeroClaw's trait-driven modular architecture. It is designed to get you productive quickly, whether you are implementing a new LLM provider, wiring up a chat platform, or exposing a hardware board as agent-accessible tools.

---

## Quick Navigation

| Document | What It Covers |
|----------|---------------|
| [Architectural Overview](overview.md) | System design philosophy, module map, extension points, configuration cascade, security model, build system, auxiliary subsystems |
| [Data Flow](data-flow.md) | End-to-end message flow diagrams — CLI, gateway/webhook, WebSocket, tool execution loop, memory consolidation, cron/routine triggers |

### Trait Deep-Dives

Each subfolder documents one core trait: its purpose, full API surface, existing implementations, factory/registration mechanics, configuration, and a step-by-step extension guide.

| Trait | Subfolder | Source | What It Abstracts |
|-------|-----------|--------|-------------------|
| `Provider` | [providers/](providers/) | `src/providers/traits.rs` | LLM model interaction — chat, streaming, tool calling |
| `Channel` | [channels/](channels/) | `src/channels/traits.rs` | Bidirectional messaging platforms (Telegram, Discord, Slack, …) |
| `Tool` | [tools/](tools/) | `src/tools/traits.rs` | Agent-callable actions (shell, file ops, web, browser, …) |
| `Memory` | [memory/](memory/) | `src/memory/traits.rs` | Long-term knowledge storage, recall, and lifecycle |
| `Observer` | [observability/](observability/) | `src/observability/traits.rs` | Event/metric recording for logging, tracing, and metrics |
| `RuntimeAdapter` | [runtime/](runtime/) | `src/runtime/traits.rs` | Platform abstraction — shell, filesystem, resource limits |
| `Peripheral` | [peripherals/](peripherals/) | `src/peripherals/traits.rs` | Hardware boards exposed as agent tools (STM32, RPi GPIO, …) |

---

## Which Document Should I Read?

```
"I want to understand the overall system."
  └─→  Start with overview.md, then data-flow.md

"I want to add a new LLM provider (e.g., Mistral, Cohere)."
  └─→  providers/README.md  §Extension Guide

"I want to add a new chat channel (e.g., Mastodon, Teams)."
  └─→  channels/README.md  §Extension Guide

"I want to add a new tool the agent can call."
  └─→  tools/README.md  §Extension Guide

"I want to add a new memory backend (e.g., Redis, Postgres)."
  └─→  memory/README.md  §Extension Guide

"I want to add observability output (e.g., Datadog, Grafana)."
  └─→  observability/README.md  §Extension Guide

"I want to support a new execution environment (e.g., Wasm, Firecracker)."
  └─→  runtime/README.md  §Extension Guide

"I want to support a new hardware board."
  └─→  peripherals/README.md  §Extension Guide

"I want to understand how a message flows end-to-end."
  └─→  data-flow.md

"I want to understand security, sandboxing, or approval."
  └─→  overview.md  §Security Architecture
```

---

## Conventions Used in This Documentation

- **Trait method signatures** are shown in Rust syntax with `async_trait` expansion omitted for clarity.
- **Default implementations** are noted explicitly; methods without defaults are marked **required**.
- **File paths** are relative to the repository root (`src/providers/traits.rs`).
- **ASCII diagrams** are used instead of external image formats so they render in any terminal or editor.
- **"Factory function"** refers to the `pub fn/async fn` in each module's `mod.rs` that constructs a `Box<dyn Trait>` from configuration.

---

## Related Resources

| Resource | Location |
|----------|----------|
| Contribution Playbooks | `docs/contributing/change-playbooks.md` |
| PR Discipline & Privacy | `docs/contributing/pr-discipline.md` |
| Docs System Contract | `docs/contributing/docs-contract.md` |
| Agent Instructions | `AGENTS.md` (repo root) |
| Full API Reference | `cargo doc --open` |
| Configuration Template | `dev/config.template.toml` |
| JSON Schema (config) | `zeroclaw config export-schema` |
