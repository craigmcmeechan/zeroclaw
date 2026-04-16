# Sandboxed Coding Agent — Design Document

**Status**: Design  
**Author**: Craig  
**Created**: 2026-03-28  

---

## 1. Vision

Run a **Docker-sandboxed instance of ZeroClaw + Claude Code** that operates as an autonomous coding agent. ZeroClaw acts as the orchestrator — receiving high-level tasks from users via channels (Telegram, Discord, etc.), delegating coding work to Claude Code running inside the container, and supervising the session via PTY/WebSocket terminal streaming.

The goal is a two-tier agent architecture:

```
User (human)
  │
  │  Telegram / Discord / Gateway / etc.
  ▼
┌─────────────────────────────────────────┐
│  ZeroClaw — Orchestrator                 │
│  (daemon mode, runs on host or in its    │
│   own container)                         │
│                                          │
│  Responsibilities:                       │
│  • Receive tasks via channels            │
│  • Decompose into sub-tasks              │
│  • Delegate to Claude Code               │
│  • Monitor PTY output in real-time       │
│  • Answer Claude Code's questions        │
│    using its own provider + memory       │
│  • Intervene on errors/stalls            │
│  • Report results back via channel       │
│  • Track cost, enforce security policy   │
│  • Store learnings in memory             │
└──────────┬──────────────────────────────┘
           │
           │  PTY / WebSocket (your existing Rust library)
           ▼
┌─────────────────────────────────────────┐
│  Docker Container — Coding Sandbox       │
│                                          │
│  Contents:                               │
│  • Claude Code CLI (npm installed)       │
│  • Dev tooling (Node, Python, Rust, etc) │
│  • Git (for the target codebase)         │
│  • ZeroClaw config (custom workspace)    │
│                                          │
│  Mounted volumes:                        │
│  • /workspace → git repo (target code)   │
│  • /zeroclaw-data → ZeroClaw config/mem  │
│                                          │
│  Environment:                            │
│  • ANTHROPIC_API_KEY (or API URL)        │
│  • Claude Code OAuth session / API key   │
│                                          │
│  Claude Code preserves:                  │
│  • CLAUDE.md / AGENTS.md instructions    │
│  • .claude/ settings & permissions       │
│  • Hooks (pre/post commit, lint, etc.)   │
│  • Internal tool loop (edit→lint→fix)    │
│  • Git commit workflows                  │
└─────────────────────────────────────────┘
```

---

## 2. Motivation

### Why sandboxed Docker?

- **Isolation**: Coding agent has full shell/file access *inside the container* but cannot affect the host or other services.
- **Reproducibility**: Identical dev environment for every task — no "works on my machine" drift.
- **Multiple instances**: Can run several sandboxed agents concurrently (different repos, different providers) without conflict.
- **Custom tooling per project**: Each container image can include project-specific SDKs, compilers, and dependencies.

### Why PTY-based supervision (not just `claude --print`)?

The existing `ClaudeCodeTool` in `src/tools/claude_code.rs` uses `claude -p` (print mode) which is fire-and-forget. This works for simple tasks but has limitations:

| Capability | `--print` mode | PTY supervision |
|-----------|---------------|-----------------|
| CLAUDE.md / hooks / .claude settings | ✅ Preserved | ✅ Preserved |
| Internal lint→fix loops | ✅ Works | ✅ Works |
| Session continuity (multi-turn) | ⚠️ Via `--resume` (cold start each time) | ✅ Persistent session |
| Real-time observability | ❌ Only final output | ✅ Stream every step |
| Answering Claude's questions | ❌ Non-interactive | ✅ ZeroClaw can inject answers |
| Intervention on errors/stalls | ❌ Must wait for timeout | ✅ Detect and react immediately |
| Long-running multi-file refactors | ⚠️ Risk of timeout | ✅ No timeout constraint |

### Why ZeroClaw as orchestrator (not Claude Code alone)?

Claude Code is a powerful coding agent but lacks:

- **Multi-channel I/O**: No native Telegram/Discord/Slack integration.
- **Persistent memory**: No cross-session knowledge accumulation.
- **Cost tracking & budgets**: No spending controls.
- **Security policy**: No autonomy levels, approval gates, or audit logging.
- **Scheduling**: No cron/routine-based task automation.
- **Multi-agent coordination**: No delegation to other sub-agents.

ZeroClaw provides all of these. The combination is: ZeroClaw for orchestration, Claude Code for coding execution.

---

## 3. What Exists Today in ZeroClaw

### 3.1 `ClaudeCodeTool` — `src/tools/claude_code.rs`

A Tool implementation that delegates tasks to `claude -p` (non-interactive mode):

- Spawns `claude -p "<prompt>" --output-format json`
- Supports `--resume <session_id>` for session continuity
- Supports `--allowedTools` for tool restriction
- Supports `--append-system-prompt` for context injection
- Enforces workspace boundary on `working_directory`
- Environment: cleared + safe whitelist + configurable passthrough
- Timeout with `kill_on_drop(true)`
- Parses JSON output, extracts `result` and `session_id`

Config: `ClaudeCodeConfig` with:
- `timeout_secs` (default: 300)
- `max_output_bytes` (default: 1MB)
- `allowed_tools` (default: Read, Edit, Bash, Write, etc.)
- `system_prompt` (optional append)
- `env_passthrough` (extra env vars to forward)

### 3.2 `DelegateTool` — `src/tools/delegate.rs`

Multi-agent delegation tool supporting:

- Named sub-agent configs with independent provider/model
- Synchronous, background, and parallel execution modes
- Background results persisted to `workspace/delegate_results/`
- Depth-limited recursion (prevents infinite delegation chains)
- Cancellation token for cascade abort

### 3.3 `RuntimeAdapter` — `src/runtime/`

Three implementations:

| Runtime | Shell access | File access | How commands run |
|---------|-------------|-------------|-----------------|
| `NativeRuntime` | ✅ | ✅ | Direct `sh -c` / `cmd /C` |
| `DockerRuntime` | ✅ | ✅ (in container) | `docker exec <id> sh -c` |
| `WasmRuntime` | ❌ | ❌ | Stub — future WASI |

Config: `[runtime]` section with `adapter`, `workspace_dir`, `[runtime.docker]`.

### 3.4 Sandbox backends — `src/security/`

Docker, Firejail, Bubblewrap, Landlock, Seatbelt, Noop. Applied to native runtime's `build_shell_command()`.

### 3.5 Hands — `src/hands/`

Named agent instances with:
- Per-hand provider/model override
- Tool allowlists
- Rolling context (learnings persist across runs)
- Cron/interval scheduling

**Current status**: Data structures and persistence implemented. Not yet integrated into the orchestration loop.

### 3.6 Gateway WebSocket — `src/gateway/ws.rs`

Chat-oriented WebSocket at `/ws/chat`:
- Message types: `message`, `chunk`, `thinking`, `tool_call`, `tool_result`, `done`, `error`
- Session management with persistence
- Bearer token auth
- Text-only (no binary/terminal streaming)

### 3.7 Shell Tool — `src/tools/shell.rs`

Non-interactive command execution:
- `tokio::process::Command::output()` — fire-and-forget
- 60s default timeout
- 1MB output truncation
- No PTY, no stdin forwarding, no session persistence

---

## 4. Gaps Identified

### Gap 1: No PTY support in ZeroClaw

All shell execution is non-interactive (`Command::output()`). No pseudo-terminal allocation, no stdin streaming, no persistent shell sessions.

**Mitigated by**: Craig's existing Rust/tokio PTY library that streams terminal I/O over WebSocket.

### Gap 2: No terminal message types on gateway WebSocket

The gateway WebSocket only handles chat messages. No binary frames, no terminal I/O protocol, no xterm data streaming.

**Needed for**: Real-time observation of Claude Code's terminal output, and for injecting input.

### Gap 3: No interactive process management

No mechanism to spawn a long-lived process, attach to it, read incremental output, write input, and detach — across tool invocations.

**Needed for**: Maintaining a Claude Code session across multiple ZeroClaw agent turns.

### Gap 4: No output classifier for Claude Code terminal state

To supervise Claude Code, ZeroClaw needs to understand what Claude Code is doing based on raw terminal output (ANSI-escaped text). No parser for detecting states like "working", "asking a question", "done", "error".

---

## 5. Design Decisions

### 5.1 Integration point: Tool (not Channel, not RuntimeAdapter)

The PTY-based Claude Code session should be a **Tool** implementation, not a Channel or RuntimeAdapter:

- **Not a Channel** — channels connect ZeroClaw to *users*. Claude Code is a tool, not a user.
- **Not a RuntimeAdapter** — the runtime adapter controls *how the agent's own commands run*. Claude Code is a delegate, not the agent itself.
- **A Tool** — tools are "actions the agent can invoke." Starting, monitoring, and interacting with a Claude Code session is an agent action.

Specifically, a **multi-action Tool** with actions: `start`, `read`, `write`, `status`, `stop`.

### 5.2 Two tools, not one replacement

Keep the existing `ClaudeCodeTool` (`claude_code`) for simple fire-and-forget tasks. Add `ClaudeCodeSessionTool` (`claude_code_session`) for interactive PTY-supervised sessions. The orchestrator can choose which to use based on task complexity.

### 5.3 ZeroClaw is the brain, Claude Code is the hands

ZeroClaw receives tasks, decomposes them, provides context from memory, and makes strategic decisions. Claude Code executes coding work within its agent loop (edit, lint, fix, commit). When Claude Code asks a question, ZeroClaw's own LLM provider answers it — guided by task context and sub-agent guidelines.

### 5.4 Docker image composition

The sandbox container image should include:

```dockerfile
FROM ubuntu:22.04

# Dev tools (language-specific layers added per project)
RUN apt-get update && apt-get install -y \
    curl git vim build-essential \
    python3 python3-pip \
    nodejs npm \
    sudo

# Claude Code CLI
RUN npm install -g @anthropic-ai/claude-code

# Non-root user
RUN useradd -m -s /bin/bash developer && \
    echo "developer ALL=(ALL) NOPASSWD:ALL" > /etc/sudoers.d/developer

USER developer
WORKDIR /workspace

CMD ["/bin/bash"]
```

Volumes:
- `/workspace` → target git repo (read-write)
- `/home/developer/.claude` → Claude Code OAuth/session (read-only or read-write)
- `/zeroclaw-data` → ZeroClaw config + memory (if running ZeroClaw instance inside)

Environment:
- `ANTHROPIC_API_KEY` — for Claude Code API billing
- `ANTHROPIC_BASE_URL` — for custom API endpoints (e.g., proxy, different provider)

---

## 6. Open Questions

1. **Should ZeroClaw run inside the same container as Claude Code, or externally?**
   - Inside: simpler networking, shared filesystem. But couples the orchestrator to the sandbox.
   - Outside: better isolation, orchestrator survives sandbox failures. Needs network bridge for PTY WebSocket.
   - **Leaning**: Outside. ZeroClaw on host (or its own container), connects to sandbox via PTY/WebSocket.

2. **How to handle Claude Code's OAuth session in Docker?**
   - Mount `~/.claude/` from host as a volume?
   - Use `ANTHROPIC_API_KEY` for API-key billing instead of Max subscription?
   - Pre-authenticate and bake session into the image? (security concern)
   - **Leaning**: `ANTHROPIC_API_KEY` env var for simplicity. Mount `.claude/` if OAuth is preferred.

3. **How granular should the output classifier be?**
   - Simple: regex-based state detection (working/asking/done/error).
   - Medium: ANSI-strip + pattern matching with configurable patterns.
   - Advanced: Use an LLM call to classify ambiguous output.
   - **Leaning**: Start with regex, add LLM fallback later if needed.

4. **Should the PTY library be vendored into ZeroClaw or kept as an external crate?**
   - Vendored: simpler build, no external dependency management.
   - External crate: cleaner separation, reusable by other projects.
   - **Leaning**: External crate behind a feature flag (e.g., `pty-session`).

5. **How does this interact with the Hands system?**
   - A "coding-hand" could be defined that uses `claude_code_session` as its primary tool.
   - Hands already support per-hand provider/model overrides and tool allowlists.
   - But Hands aren't wired into the orchestration loop yet.
   - **Leaning**: Build the tool first, wire into Hands later when Hands integration is completed.
