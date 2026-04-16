# ZeroClaw — Development Documentation

Internal design documents, architecture decisions, and implementation plans for features under active development.

> These documents are **working drafts** — not user-facing docs. They capture design intent, trade-off analysis, and implementation roadmaps for contributors.

---

## Active Documents

| Document | Status | Summary |
|----------|--------|---------|
| [gRPC Memory Integration Plan](PLAN-grpc-memory-integration.md) | Planning | Embedded gRPC memory daemon in ZeroClaw for multi-agent memory sharing |
| [Sandboxed Coding Agent](sandboxed-coding-agent.md) | Design | Docker-sandboxed Claude Code integration with PTY-based interactive supervision |
| [Implementation Plan](implementation-plan.md) | Planning | Phased rollout plan for the sandboxed coding agent feature |

## Reference Documents

| Document | Summary |
|----------|---------|
| [SQLite gRPC Memory Service (v3)](PLAN-sqlite-grpc-memory-service_3.md) | Original standalone gRPC service design — protobuf definitions and RPC signatures remain authoritative |

---

## Conventions

- Keep documents focused on **one feature or initiative** per file.
- Use status labels: `Design`, `Planning`, `In Progress`, `Done`, `Abandoned`.
- Link to source files using relative paths from repo root (e.g., `src/tools/claude_code.rs`).
- Update documents as implementation proceeds — these are living documents, not snapshots.
