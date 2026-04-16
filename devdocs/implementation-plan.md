# Sandboxed Coding Agent — Implementation Plan

**Status**: Planning  
**Depends on**: [sandboxed-coding-agent.md](sandboxed-coding-agent.md) (design document)  
**Created**: 2026-03-28  

---

## Phase Overview

| Phase | Name | Effort | Dependencies | Deliverable |
|-------|------|--------|--------------|-------------|
| 0 | Docker image & config | Small | None | Working container with Claude Code + dev tools |
| 1 | PTY library integration | Medium | Phase 0, external crate | Feature-gated PTY crate wired into ZeroClaw |
| 2 | `ClaudeCodeSessionTool` | Medium | Phase 1 | New multi-action Tool for interactive sessions |
| 3 | Output classifier | Small–Medium | Phase 2 | State detection for Claude Code terminal output |
| 4 | Orchestrator loop | Medium | Phases 2–3 | ZeroClaw agent can autonomously drive a Claude Code session |
| 5 | Polish & Hands integration | Small | Phase 4 | Config schema, Hand definition, docs |

---

## Phase 0 — Docker Image & Config

**Goal**: A ready-to-use Docker image that runs Claude Code inside a sandboxed dev environment, with ZeroClaw config mounted externally.

### Tasks

- [ ] **0.1** Create `dev/sandbox/Dockerfile.claude-code` — extends the existing sandbox image with Claude Code CLI
  ```dockerfile
  FROM zeroclaw-sandbox:latest  # or ubuntu:22.04 base

  # Claude Code CLI
  RUN npm install -g @anthropic-ai/claude-code

  # Optional: Rust toolchain (for ZeroClaw-on-ZeroClaw work)
  # RUN curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh -s -- -y

  USER developer
  WORKDIR /workspace
  CMD ["/bin/bash"]
  ```

- [ ] **0.2** Create `dev/sandbox/docker-compose.claude-code.yml` — compose file for spinning up the coding sandbox
  ```yaml
  services:
    coding-agent:
      build:
        context: .
        dockerfile: Dockerfile.claude-code
      container_name: zeroclaw-coding-agent
      volumes:
        - ${TARGET_REPO}:/workspace              # Git repo to work on
        - ${CLAUDE_CONFIG:-~/.claude}:/home/developer/.claude  # OAuth session
        - zeroclaw-coding-data:/zeroclaw-data     # ZeroClaw config/memory
      environment:
        - ANTHROPIC_API_KEY=${ANTHROPIC_API_KEY:-}
        - ANTHROPIC_BASE_URL=${ANTHROPIC_BASE_URL:-}
      stdin_open: true
      tty: true    # Allocate TTY for interactive use
      deploy:
        resources:
          limits:
            cpus: '4'
            memory: 2G
  volumes:
    zeroclaw-coding-data:
  ```

- [ ] **0.3** Create a ZeroClaw config template for the sandboxed instance — `dev/sandbox/config.claude-code.toml`
  ```toml
  workspace_dir = "/zeroclaw-data/workspace"
  default_provider = "anthropic"
  default_model = "claude-sonnet-4-20250514"

  [claude_code]
  timeout_secs = 600
  allowed_tools = ["Read", "Edit", "Bash", "Write", "MultiEdit"]
  env_passthrough = ["ANTHROPIC_API_KEY", "ANTHROPIC_BASE_URL"]
  ```

- [ ] **0.4** Manual validation: build image, mount a test repo, run `claude -p "list all files" --output-format json` inside the container.

### Definition of Done

- Container builds and runs without errors.
- Claude Code CLI is accessible and authenticated (via API key or mounted OAuth).
- A test `claude -p` invocation succeeds against a mounted repo.

---

## Phase 1 — PTY Library Integration

**Goal**: Wire Craig's existing Rust/tokio PTY WebSocket library into the ZeroClaw build behind a feature flag.

### Tasks

- [ ] **1.1** Evaluate integration approach:
  - **Option A**: Add as a workspace crate in `crates/pty-session/`
  - **Option B**: Publish as a standalone crate, add as a `Cargo.toml` dependency
  - **Option C**: Vendor directly into `src/tools/pty/`
  - **Decision**: TBD based on crate structure and reusability needs

- [ ] **1.2** Add feature flag to `Cargo.toml`:
  ```toml
  [features]
  pty-session = ["dep:pty-session"]  # or whatever the crate name is
  ```

- [ ] **1.3** Define the internal interface ZeroClaw needs from the PTY library:
  ```rust
  /// Minimal interface ZeroClaw needs from the PTY layer.
  pub trait PtySession: Send + Sync {
      /// Spawn a process in a PTY. Returns a session handle.
      async fn spawn(&self, command: &str, args: &[&str], env: &HashMap<String, String>) -> Result<SessionId>;

      /// Read available output since last read. Non-blocking. Returns empty if nothing new.
      async fn read(&self, session: &SessionId) -> Result<String>;

      /// Write input to the PTY's stdin.
      async fn write(&self, session: &SessionId, input: &str) -> Result<()>;

      /// Check if the spawned process is still running.
      async fn is_alive(&self, session: &SessionId) -> Result<bool>;

      /// Kill the process and clean up.
      async fn kill(&self, session: &SessionId) -> Result<()>;

      /// Resize the PTY (if needed for formatting).
      async fn resize(&self, session: &SessionId, cols: u16, rows: u16) -> Result<()>;
  }
  ```

- [ ] **1.4** Adapt or wrap the existing PTY library to implement this interface.

- [ ] **1.5** Write integration test: spawn `/bin/bash`, write `echo hello`, read output, confirm `hello` in output.

### Definition of Done

- PTY library compiles as part of ZeroClaw when `pty-session` feature is enabled.
- Can spawn a process, read/write, and kill it from Rust.
- Integration test passes in CI (Linux at minimum).

---

## Phase 2 — `ClaudeCodeSessionTool`

**Goal**: A new Tool that allows the ZeroClaw agent to start, monitor, interact with, and stop an interactive Claude Code session via PTY.

### Tasks

- [ ] **2.1** Create `src/tools/claude_code_session.rs` implementing the `Tool` trait:
  ```
  Tool name: "claude_code_session"
  Actions:
    start   — spawn Claude Code in container PTY, return session_id
    read    — get recent output from session (since last read)
    write   — send input/answer to Claude Code's stdin
    status  — return session state (working/asking/done/error/idle)
    stop    — kill the session
  ```

- [ ] **2.2** Define `ClaudeCodeSessionConfig` in `src/config/schema.rs`:
  ```rust
  pub struct ClaudeCodeSessionConfig {
      /// Docker container to exec into (or "local" for native PTY).
      pub container: Option<String>,
      /// Docker image to auto-start if container doesn't exist.
      pub image: Option<String>,
      /// Default working directory inside the container.
      pub working_directory: String,
      /// Read poll interval (ms) for the monitor loop.
      pub poll_interval_ms: u64,
      /// Max idle time before session is considered stale (secs).
      pub idle_timeout_secs: u64,
      /// Environment variables to set in the PTY session.
      pub env: HashMap<String, String>,
      /// Claude Code CLI arguments (e.g., model override).
      pub claude_args: Vec<String>,
  }
  ```

- [ ] **2.3** Session state management:
  ```rust
  pub struct SessionState {
      pub id: String,
      pub status: SessionStatus,
      pub started_at: chrono::DateTime<chrono::Utc>,
      pub last_activity: chrono::DateTime<chrono::Utc>,
      pub output_buffer: String,       // Accumulated since last read
      pub total_output: String,        // Full session log (capped)
  }

  pub enum SessionStatus {
      Starting,
      Idle,          // Claude Code waiting for input
      Working,       // Claude Code actively processing
      Asking,        // Claude Code asking a question (detected by classifier)
      Done,          // Claude Code completed the task
      Error(String), // Claude Code hit an error
      Killed,        // Session was terminated
  }
  ```

- [ ] **2.4** Session store — `HashMap<String, SessionState>` behind `Arc<RwLock<>>`. Active sessions persist in memory; session logs saved to `workspace/coding_sessions/`.

- [ ] **2.5** Register in tool factory (`src/tools/mod.rs`) behind `pty-session` feature flag.

- [ ] **2.6** Security integration:
  - Enforce `ToolOperation::Act` (same as existing `claude_code` tool)
  - Rate limiting applies to `start` action
  - Workspace boundary enforcement on `working_directory`
  - Container ID validation (must match configured container)

- [ ] **2.7** Unit tests for session lifecycle and state transitions.

### Tool Schema

```json
{
  "type": "object",
  "properties": {
    "action": {
      "type": "string",
      "enum": ["start", "read", "write", "status", "stop"],
      "description": "The session operation to perform"
    },
    "session_id": {
      "type": "string",
      "description": "Session ID (required for read/write/status/stop)"
    },
    "prompt": {
      "type": "string",
      "description": "Initial task prompt (for 'start' action)"
    },
    "input": {
      "type": "string",
      "description": "Text to send to Claude Code's stdin (for 'write' action)"
    },
    "working_directory": {
      "type": "string",
      "description": "Working directory for the session (for 'start' action)"
    }
  },
  "required": ["action"]
}
```

### Definition of Done

- Can start a Claude Code session via the tool.
- Can read streaming output and write input.
- Session state tracking works (start → working → done).
- Security policy enforced (workspace boundary, rate limit, autonomy level).

---

## Phase 3 — Output Classifier

**Goal**: Parse ANSI-stripped Claude Code terminal output to detect the current state of the session.

### Tasks

- [ ] **3.1** Create `src/tools/claude_code_classifier.rs`:
  - ANSI escape code stripper (regex: `\x1b\[[0-9;]*[a-zA-Z]`)
  - Pattern matchers for known Claude Code output states

- [ ] **3.2** Define detection patterns (initial set — will evolve):
  ```
  ASKING:
    - Line starts with "? " (interactive question)
    - Line contains "Do you want to" / "Should I" / "Would you like"
    - Prompt character "❯ " at start of line after question text

  WORKING:
    - Tool execution markers: "⏺ ", "● Read", "● Edit", "● Bash", "● Write"
    - Progress output (file paths, diff output, command output)
    - "Thinking..." or reasoning text

  DONE:
    - Final summary output (no more tool calls, text ends)
    - Exit code 0
    - Pattern: "I've completed" / "Done." / "All changes have been"

  ERROR:
    - Non-zero exit code
    - "Error:", "error[", "FAILED", "panic"
    - Claude Code crash/disconnection

  IDLE:
    - No output for N seconds after previous activity
    - Waiting for input (cursor blinking)
  ```

- [ ] **3.3** Implement as a state machine:
  ```rust
  pub struct OutputClassifier {
      current_state: SessionStatus,
      last_output_time: Instant,
      idle_threshold: Duration,
  }

  impl OutputClassifier {
      /// Feed new output, return updated state.
      pub fn classify(&mut self, raw_output: &str) -> SessionStatus { ... }
  }
  ```

- [ ] **3.4** Unit tests with captured Claude Code output samples (collect real output from manual sessions for test fixtures).

- [ ] **3.5** (Future) Optional LLM-based fallback classifier for ambiguous output — call ZeroClaw's own provider with "Is Claude Code asking me a question? Output: ..."

### Definition of Done

- Classifier correctly detects Working, Asking, Done, Error states on sample output.
- ANSI stripping works for common escape sequences.
- False positive rate on "Asking" state is low (avoids ZeroClaw injecting input when Claude Code is just narrating).

---

## Phase 4 — Orchestrator Loop

**Goal**: The ZeroClaw agent can autonomously receive a coding task, start a Claude Code session, monitor it, answer questions, and report results. This is the integration phase where everything comes together.

### Tasks

- [ ] **4.1** Define a **system prompt fragment** for the orchestrator when `claude_code_session` tool is available:
  ```
  You have access to the claude_code_session tool which lets you start and
  interact with an interactive Claude Code coding session. When given a
  coding task:

  1. Start a session with the task prompt.
  2. Periodically read output to monitor progress.
  3. If Claude Code asks a question, evaluate it against the task
     requirements and your memory, then write an answer.
  4. If Claude Code reports an error, analyze it and provide guidance.
  5. When the session completes, summarize the results and report
     to the user.
  6. Store key learnings (decisions made, patterns found) in memory.

  Do NOT micro-manage Claude Code. Let it work autonomously through its
  own edit→lint→fix loops. Only intervene when it explicitly asks a
  question or encounters an error it cannot resolve.
  ```

- [ ] **4.2** Implement a **monitor loop pattern** — the agent's tool-call loop needs to re-invoke `claude_code_session/read` periodically. Options:
  - **Option A**: Agent naturally loops (LLM decides to call `read` after seeing "Working" status) — relies on LLM behavior.
  - **Option B**: Tool returns a special `ToolResult` with a "check again in N seconds" hint — agent loop has built-in retry.
  - **Option C**: Background monitor thread pushes events into the agent's turn — needs changes to the agent loop.
  - **Leaning**: Option A initially (simplest), evolve to Option B if the LLM doesn't reliably re-poll.

- [ ] **4.3** Test end-to-end flow manually:
  1. Start ZeroClaw in daemon mode with a channel.
  2. Send a coding task via the channel.
  3. Verify ZeroClaw starts a Claude Code session.
  4. Verify ZeroClaw reads output and detects states correctly.
  5. Simulate Claude Code asking a question — verify ZeroClaw answers.
  6. Verify ZeroClaw reports completion via channel.

- [ ] **4.4** Handle edge cases:
  - Claude Code crashes mid-session → detect via `is_alive()`, report error.
  - Claude Code hangs (no output for too long) → idle timeout, kill and report.
  - User cancels task via channel → cascade kill to PTY session.
  - Multiple concurrent sessions → session store handles isolation.
  - Container doesn't exist or Docker is not available → graceful error.

- [ ] **4.5** Cost tracking integration — each Claude Code session should contribute to `CostTracker` (at minimum, track elapsed time; ideally, parse Claude Code's token usage from its output).

### Definition of Done

- Full round-trip: user message → ZeroClaw → Claude Code session → task completed → user notified.
- ZeroClaw autonomously answers at least one Claude Code question during a session.
- Error handling works for crash, timeout, and cancellation scenarios.

---

## Phase 5 — Polish & Hands Integration

**Goal**: Production-ready config, a Hand definition for the coding agent, and documentation.

### Tasks

- [ ] **5.1** Finalize `ClaudeCodeSessionConfig` with sensible defaults and add to `dev/config.template.toml`.

- [ ] **5.2** Create example Hand definition — `examples/hands/coding-agent.toml`:
  ```toml
  name = "coding-agent"
  description = "Autonomous coding agent powered by Claude Code"
  active = true

  prompt = """
  You are a coding orchestrator. When given a task, use the
  claude_code_session tool to delegate the actual coding work
  to Claude Code running in a sandboxed Docker container.
  Monitor progress, answer questions, and report results.
  """

  allowed_tools = [
    "claude_code_session",
    "memory_store",
    "memory_recall",
    "ask_user",
    "escalate"
  ]

  model = "anthropic/claude-sonnet-4-20250514"
  max_history = 50

  [schedule]
  kind = "manual"  # Triggered by delegation, not cron
  ```

- [ ] **5.3** Update `architecture/tools/README.md` with the new tool entry.

- [ ] **5.4** Write user-facing setup guide: `docs/coding-agent-setup.md`.

- [ ] **5.5** Add integration test to CI (feature-gated, requires Docker).

### Definition of Done

- Config documented in template.
- Example Hand definition works out of the box.
- Architecture and setup docs updated.
- CI integration test passes when Docker is available.

---

## Risk Register

| Risk | Impact | Mitigation |
|------|--------|------------|
| Claude Code output format changes between versions | Classifier breaks | Pin Claude Code version in Docker image; make patterns configurable |
| PTY library has platform-specific issues (Windows/macOS) | Build failures on some platforms | Feature-gated; Linux-first, test on macOS; Windows via WSL |
| LLM doesn't reliably re-poll `read` action | Agent stops monitoring mid-session | Implement Option B (check-again hint) in Phase 4.2 |
| Claude Code API key costs accumulate in unsupervised sessions | Budget overrun | Wire into CostTracker; enforce `max_cost_per_day_cents` |
| Claude Code session hangs with no output | Zombie session consumes resources | Idle timeout + kill in Phase 4.4 |
| ANSI escape codes vary across terminal emulators | Classifier misses patterns | Strip all ANSI before classification; test with real captured output |
