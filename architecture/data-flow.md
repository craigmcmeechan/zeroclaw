# ZeroClaw — Data Flow

This document traces messages through ZeroClaw end-to-end across the three primary entry points (CLI, Gateway REST/webhook, WebSocket) and details the internal loops for tool execution, memory consolidation, and scheduled automation.

---

## 1. CLI Interactive Flow

The simplest path — a user types a message in the terminal and gets a response.

```
┌──────────┐
│  stdin   │
│  (user)  │
└────┬─────┘
     │ text input
     ▼
┌──────────────────────────────────────────────────────────┐
│  main.rs → Commands::Agent                               │
│  Parses --provider, --model, --temperature, --peripheral │
└────┬─────────────────────────────────────────────────────┘
     │
     ▼
┌──────────────────────────────────────────────────────────┐
│  agent::loop_::run()                                     │
│                                                          │
│  1. Initialize Observer, Runtime, SecurityPolicy         │
│  2. Create Memory backend (sqlite/markdown/…)            │
│  3. Build Tool registry (security-filtered)              │
│  4. Create Provider (with credential resolution)         │
│  5. Build Agent via AgentBuilder                         │
│  6. Enter REPL loop:                                     │
│     ┌──────────────────────────────────────────┐         │
│     │  Read user input from stdin              │         │
│     │  Call agent.turn_streamed(message)        │         │
│     │  Drain TurnEvent stream → print to stdout│         │
│     │  Loop until EOF or "exit"                │         │
│     └──────────────────────────────────────────┘         │
└──────────────────────────────────────────────────────────┘
```

**Wire-up order matters**: Observer and Runtime are created first (they are immutable infrastructure), then Memory (needs storage path from Runtime), then Tools (need SecurityPolicy), then Provider (needs config + credentials).

---

## 2. Gateway REST / Webhook Flow

External services (Telegram, Slack webhooks, custom integrations) send HTTP requests to the gateway.

```
┌────────────────┐
│  HTTP Client   │
│  (webhook,     │
│   curl, app)   │
└───────┬────────┘
        │ POST /api/message  (or channel-specific webhook)
        │ Headers: Authorization: Bearer <token>
        ▼
┌──────────────────────────────────────────────────────────┐
│  Gateway (Axum)                                          │
│                                                          │
│  1. Rate limiter check (sliding window per IP)           │
│  2. Bearer token verification (PairingGuard)             │
│  3. Idempotency key dedup (if X-Idempotency-Key present)│
│  4. Request body size limit (64 KB)                      │
│  5. Request timeout (30s or ZEROCLAW_GATEWAY_TIMEOUT_SECS)│
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  Session Queue                                           │
│  Serializes concurrent turns for the same session_id.    │
│  If a turn is already in progress, the new request waits.│
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  agent::loop_::process_message()                         │
│                                                          │
│  Same wire-up as run(), but:                             │
│  - Single-turn execution (no REPL)                       │
│  - ApprovalManager::for_non_interactive()                │
│  - Streams TurnEvents, collects final response           │
│  - Optional session_id for memory persistence            │
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  HTTP Response                                           │
│  200 OK { "response": "...", "session_id": "..." }       │
└──────────────────────────────────────────────────────────┘
```

---

## 3. WebSocket Chat Flow

Real-time bidirectional streaming via WebSocket.

```
┌────────────────┐
│  WS Client     │
│  (web UI,      │
│   mobile app)  │
└───────┬────────┘
        │ GET /ws/chat?session_id=ID&token=BEARER
        │ Upgrade: websocket
        ▼
┌──────────────────────────────────────────────────────────┐
│  Gateway (Axum) → ws.rs handler                          │
│                                                          │
│  1. Token auth (query param, header, or Sec-WS-Protocol) │
│  2. Session load/resume from backend (SQLite/Redis)      │
│  3. Send session_start event to client:                  │
│     {"type":"session_start","session_id":"…",            │
│      "resumed":true,"message_count":42}                  │
└───────┬──────────────────────────────────────────────────┘
        │
        │ ◄── bidirectional ──►
        │
   ┌────┴──────────────────────────────────────────────┐
   │  Message Loop                                     │
   │                                                   │
   │  Client → Server:                                 │
   │    {"type":"message","content":"Hello"}            │
   │                                                   │
   │  Server processes via agent.turn_streamed():      │
   │                                                   │
   │  Server → Client (streamed):                      │
   │    {"type":"chunk","content":"Hi "}                │
   │    {"type":"chunk","content":"there! "}            │
   │    {"type":"tool_call","name":"shell",             │
   │     "args":{"command":"ls"}}                       │
   │    {"type":"tool_result","name":"shell",           │
   │     "output":"file1.txt\nfile2.txt"}               │
   │    {"type":"chunk","content":"I found 2 files."}   │
   │    {"type":"done","full_response":"Hi there! …"}   │
   │                                                   │
   │  Repeat until client disconnects                  │
   └───────────────────────────────────────────────────┘
```

**Key difference from REST**: Streaming. Each `TurnEvent` is emitted as a separate WebSocket frame in real time — text chunks, tool calls, tool results, and the final done event.

---

## 4. Agent Turn — Core Orchestration Loop

This is the heart of ZeroClaw. Every entry point (CLI, REST, WS) funnels into `Agent::turn_streamed()`.

```
┌──────────────────────────────────────────────────────────────┐
│  Agent::turn_streamed(user_message)                          │
│                                                              │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 1. SYSTEM PROMPT CONSTRUCTION                          │  │
│  │    - Agent personality injection                       │  │
│  │    - Security policy summary                           │  │
│  │    - Available tools description (if XML dispatcher)   │  │
│  │    - Workspace context                                 │  │
│  │    Built via PromptSection trait chain                  │  │
│  └────────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 2. HISTORY & MEMORY                                    │  │
│  │    - Load session history (auto-resume)                │  │
│  │    - Trim to token budget (history_pruner)             │  │
│  │    - Hybrid memory recall:                             │  │
│  │      • Vector similarity search                        │  │
│  │      • FTS5 keyword search                             │  │
│  │      • Time-decay weighting                            │  │
│  │    - Inject relevant memories as context               │  │
│  └────────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 3. LLM API CALL                                        │  │
│  │    - Build ChatRequest (messages + tools + options)     │  │
│  │    - Provider::chat() or Provider::stream_chat()        │  │
│  │    - Observer records LlmRequest event                  │  │
│  │    - Emit TurnEvent::Chunk for each text delta          │  │
│  │    - Emit TurnEvent::Thinking for reasoning output      │  │
│  └────────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 4. PARSE TOOL CALLS (Dispatcher)                       │  │
│  │                                                        │  │
│  │    XmlToolDispatcher:                                  │  │
│  │      Parse <tool_call>{"name":"…","arguments":{…}}     │  │
│  │      </tool_call> from response text                   │  │
│  │                                                        │  │
│  │    NativeToolDispatcher:                               │  │
│  │      Read structured tool_calls from ChatResponse      │  │
│  │                                                        │  │
│  │    If no tool calls → skip to step 7                   │  │
│  └────────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 5. TOOL EXECUTION LOOP                                 │  │
│  │    (repeats until no more tool calls or max_iterations) │  │
│  │                                                        │  │
│  │    For each tool call:                                 │  │
│  │    ┌─────────────────────────────────────────────────┐ │  │
│  │    │ a. Lookup tool in registry by name              │ │  │
│  │    │ b. Validate arguments against JSON Schema       │ │  │
│  │    │ c. SecurityPolicy check:                        │ │  │
│  │    │    - Is tool allowed?                           │ │  │
│  │    │    - Command risk classification (Low/Med/High) │ │  │
│  │    │    - Action rate limit check                    │ │  │
│  │    │ d. Approval gate (if supervised + medium risk): │ │  │
│  │    │    - Prompt user / check auto-approve list      │ │  │
│  │    │    - Wait for ApprovalResponse                  │ │  │
│  │    │ e. Execute tool (parallel or sequential)        │ │  │
│  │    │    - In sandbox if shell command                │ │  │
│  │    │ f. Scrub credentials from output                │ │  │
│  │    │ g. Observer records ToolCall event               │ │  │
│  │    │ h. Cost tracking (token usage)                  │ │  │
│  │    │ i. Emit TurnEvent::ToolCall + ToolResult        │ │  │
│  │    └─────────────────────────────────────────────────┘ │  │
│  │                                                        │  │
│  │    Format results via dispatcher                       │  │
│  │    Append to conversation history                      │  │
│  │    Send results back to LLM (goto step 3)             │  │
│  │                                                        │  │
│  │    Loop detection:                                     │  │
│  │    - If same tool called with same args repeatedly     │  │
│  │    - Auto-compaction of history if context too long    │  │
│  └────────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 6. FINAL RESPONSE                                      │  │
│  │    - LLM returns text without tool calls               │  │
│  │    - Emit final TurnEvent::Chunk (is_final = true)     │  │
│  │    - Observer records TurnComplete event                │  │
│  └────────────────────────────────────────────────────────┘  │
│                         │                                    │
│                         ▼                                    │
│  ┌────────────────────────────────────────────────────────┐  │
│  │ 7. POST-TURN BOOKKEEPING                               │  │
│  │    - Append to session history                         │  │
│  │    - Memory autosave (async fire-and-forget):          │  │
│  │      Extract facts → store to long-term memory         │  │
│  │    - Observer records AgentEnd with duration/cost      │  │
│  └────────────────────────────────────────────────────────┘  │
│                                                              │
│  Return: final response string                               │
└──────────────────────────────────────────────────────────────┘
```

---

## 5. Tool Execution Detail

Zooming into the tool execution sandbox path:

```
Tool call parsed: { name: "shell", args: { command: "cargo test" } }
                │
                ▼
┌──────────────────────────────────────────────────────────┐
│  Tool Registry Lookup                                    │
│  Find Box<dyn Tool> by name in tools Vec                 │
│  If not found → return error to LLM                      │
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  SecurityPolicy Gate                                     │
│                                                          │
│  workspace_only?  → validate paths stay in workspace     │
│  allowed_commands? → check against whitelist             │
│  forbidden_paths?  → reject if args reference them       │
│  rate_limit?       → check tracker (actions/hour)        │
│                                                          │
│  Classify command risk:                                  │
│    "cargo test" → Medium                                 │
│                                                          │
│  If autonomy == Supervised && risk >= Medium:            │
│    → Enter approval workflow                             │
│  If block_high_risk && risk == High:                     │
│    → Reject, return error to LLM                         │
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  Approval Workflow (if triggered)                        │
│                                                          │
│  Check auto-approve list (config)                        │
│  Check session "Always" allowlist (runtime cache)        │
│  If not pre-approved:                                    │
│    → Display approval prompt to user                     │
│    → Wait for: Approve / Deny / Always-Approve           │
│  Log decision to ApprovalLogEntry                        │
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  RuntimeAdapter::build_shell_command()                   │
│                                                          │
│  NativeRuntime: direct tokio::process::Command           │
│  DockerRuntime: wraps in `docker exec` with mounts       │
│  WasmRuntime: (stub — not yet implemented)               │
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  Sandbox Execution                                       │
│                                                          │
│  Docker:     docker run --rm --network=none …            │
│  Firejail:   firejail --quiet --private=… command        │
│  Bubblewrap: bwrap --ro-bind … --dev /dev command        │
│  Landlock:   landlock_restrict_self() + exec             │
│  Seatbelt:   sandbox-exec -f profile command             │
│  Noop:       direct process spawn (no isolation)         │
│                                                          │
│  Capture stdout + stderr                                 │
│  Enforce timeout                                         │
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  Post-Execution                                          │
│                                                          │
│  Credential scrubbing (regex-based redaction)            │
│  Truncate output if exceeds limit                        │
│  Build ToolResult { success, output, error }             │
│  Emit TurnEvent::ToolResult                              │
│  Observer.record_event(ToolCall { tool, duration, … })   │
│  CostTracker.record(tokens_used)                         │
└──────────────────────────────────────────────────────────┘
```

---

## 6. Channel Listener Flow

Channels run concurrently as tokio tasks, feeding messages into the agent via an mpsc channel.

```
┌──────────────────────────────────────────────────────────┐
│  Daemon Mode Startup (daemon/mod.rs)                     │
│                                                          │
│  For each enabled channel in config:                     │
│    1. Construct Box<dyn Channel> via factory              │
│    2. Create mpsc::channel<ChannelMessage>(buffer)        │
│    3. Spawn tokio task: channel.listen(tx)               │
│                                                          │
│  Also spawn: gateway server, heartbeat engine, cron      │
└───────┬──────────────────────────────────────────────────┘
        │
        ▼
┌────────────────────────────┐   ┌────────────────────────────┐
│  Telegram::listen(tx)      │   │  Discord::listen(tx)       │
│  - Long-polling /getUpdates│   │  - Gateway connection       │
│  - Parse Update → Channel  │   │  - on_message event         │
│    Message                 │   │  - Parse → ChannelMessage   │
│  - tx.send(msg)            │   │  - tx.send(msg)             │
└────────────┬───────────────┘   └──────────────┬─────────────┘
             │                                   │
             └──────────┬────────────────────────┘
                        │  mpsc::Receiver<ChannelMessage>
                        ▼
┌──────────────────────────────────────────────────────────┐
│  Message Router                                          │
│                                                          │
│  For each received ChannelMessage:                       │
│    1. Resolve session_id (from sender + channel + thread)│
│    2. Call process_message() with message content         │
│    3. Collect response                                   │
│    4. Build SendMessage { content, recipient, thread_ts } │
│    5. channel.send(send_message)                         │
│                                                          │
│  Supports:                                               │
│    - Typing indicators (start_typing / stop_typing)      │
│    - Draft updates (send_draft / update_draft / finalize)│
│    - Threaded replies (thread_ts propagation)            │
│    - Reactions (add_reaction on tool progress)           │
│    - Interruption scoping (per-thread isolation)         │
└──────────────────────────────────────────────────────────┘
```

---

## 7. Memory Consolidation Flow

Memory operations happen at two points: during a turn (recall) and after a turn (store).

```
┌──────────────────────────────────────────────────────────┐
│  RECALL (during turn, step 2)                            │
│                                                          │
│  Input: user message text, session_id, time window       │
│                                                          │
│  SqliteMemory (dual-index):                              │
│  ┌────────────────────┐  ┌────────────────────────────┐  │
│  │ Vector Search      │  │ FTS5 Keyword Search        │  │
│  │ (embedding cosine) │  │ (SQLite full-text)         │  │
│  └────────┬───────────┘  └──────────┬─────────────────┘  │
│           │                          │                    │
│           └──────────┬───────────────┘                    │
│                      ▼                                    │
│  ┌────────────────────────────────────────────────────┐   │
│  │ Merge & Rank                                       │   │
│  │ - Weighted combination (vector_weight, kw_weight)  │   │
│  │ - Time decay                                       │   │
│  │ - Importance boost                                 │   │
│  │ - Dedup by key                                     │   │
│  │ - Limit to top N results                           │   │
│  └────────────────────────────────────────────────────┘   │
│                      │                                    │
│                      ▼                                    │
│  Inject as context messages before user message           │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  STORE (after turn, step 7 — async fire-and-forget)      │
│                                                          │
│  Extract facts from conversation:                        │
│    - User preferences, corrections                       │
│    - Learned information                                 │
│    - Task outcomes                                       │
│                                                          │
│  Categorize:                                             │
│    Core         → Evergreen facts (not time-decayed)     │
│    Daily        → Per-day summaries                      │
│    Conversation → Per-session context                    │
│                                                          │
│  Store via Memory::store_with_metadata()                 │
│    - Generate embedding for vector index                 │
│    - Insert into FTS5 for keyword index                  │
│    - Set namespace, importance, session_id               │
│                                                          │
│  Supersession:                                           │
│    If new fact contradicts old → set superseded_by       │
└──────────────────────────────────────────────────────────┘
```

---

## 8. Cron & Routine Trigger Flow

Scheduled and event-driven automation.

```
┌──────────────────────────────────────────────────────────┐
│  Cron Scheduler (cron/scheduler.rs)                      │
│                                                          │
│  Runs as background tokio task in daemon mode.           │
│  Every 60s:                                              │
│    1. Load all CronJobs from SQL store                   │
│    2. For each job, check if next_run <= now             │
│    3. Validate command against SecurityPolicy             │
│    4. Execute:                                           │
│       - Shell type → RuntimeAdapter::build_shell_command │
│       - SOP type → SopEngine::run(sop_name)              │
│       - Message type → process_message(content)          │
│    5. Record CronRun (success/failure, duration)         │
│    6. Compute next_run from cron expression              │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  Routines Engine (routines/engine.rs)                    │
│                                                          │
│  Loaded from routines.toml at startup.                   │
│                                                          │
│  On every event (webhook, cron tick, channel message):   │
│    1. Match event against each Routine's EventPattern    │
│       - Exact, Glob, or Regex match strategies           │
│    2. Check per-routine cooldown timer                   │
│    3. Fire RoutineAction:                                │
│       - TriggerSop(sop_name)                             │
│       - ShellCommand(cmd)                                │
│       - SendMessage(channel, content)                    │
│    4. Update cooldown timer                              │
└──────────────────────────────────────────────────────────┘

┌──────────────────────────────────────────────────────────┐
│  Hooks Runner (hooks/runner.rs)                          │
│                                                          │
│  Intercepts lifecycle events:                            │
│    - BeforeToolExecution(tool, args)                     │
│    - AfterToolExecution(tool, result)                    │
│    - OnMessage(channel_message)                          │
│                                                          │
│  For each registered HookHandler:                        │
│    handler.handle(event) → HookResult                    │
│    - Continue: allow the operation to proceed            │
│    - Abort: prevent the operation                        │
│    - Modify: alter the args/result                       │
│                                                          │
│  Built-in hooks:                                         │
│    - WebhookAuditHook: logs all webhook events           │
│    - CommandLoggerHook: logs shell commands               │
└──────────────────────────────────────────────────────────┘
```

---

## 9. Gateway Initialization Sequence

How the gateway assembles its state at startup:

```
zeroclaw gateway start
        │
        ▼
┌──────────────────────────────────────────────────────────┐
│  1. Load Config                                          │
│     config.toml → merge env overrides → validate         │
│                                                          │
│  2. Initialize subsystems:                               │
│     Observer   ← config.observability.backend            │
│     Runtime    ← config.runtime.kind                     │
│     Memory     ← config.memory.backend                   │
│     Provider   ← config.default_provider + credentials   │
│     Tools      ← build_tools(config, security, runtime)  │
│                                                          │
│  3. Build AppState (Arc-wrapped for Axum sharing):       │
│     - Config, Memory, Provider (all Arc<…>)              │
│     - PairingGuard (require_pairing from config)         │
│     - RateLimiter (limit_per_window from config)         │
│     - IdempotencyStore (TTL from config)                 │
│     - SessionQueue (concurrency serializer)               │
│     - SessionBackend (SQLite/Redis)                      │
│     - Event broadcast channel (mpsc unbounded)           │
│                                                          │
│  4. Build Axum Router:                                   │
│     POST /pair              → pairing handler            │
│     GET  /api/status        → status (Bearer auth)       │
│     GET  /api/config        → config (masked, Bearer)    │
│     PUT  /api/config        → update config (Bearer)     │
│     GET  /api/tools         → list tools (Bearer)        │
│     GET  /api/cron          → list cron jobs (Bearer)    │
│     POST /api/cron          → create cron job (Bearer)   │
│     GET  /api/memory        → search memory (Bearer)     │
│     POST /api/memory        → store memory (Bearer)      │
│     GET  /ws/chat           → WebSocket upgrade          │
│     GET  /*                 → static files               │
│                                                          │
│  5. Apply middleware layers:                             │
│     - Rate limiting                                      │
│     - Request body limit (64 KB)                         │
│     - Request timeout                                    │
│     - CORS (if configured)                               │
│     - TLS/mTLS (if configured)                           │
│                                                          │
│  6. Bind to gateway.host:gateway.port                    │
│  7. Accept connections                                   │
└──────────────────────────────────────────────────────────┘
```

---

## 10. Dispatcher Variants

The agent supports two tool-calling protocols, selected by provider capabilities:

### XML Dispatcher (default for providers without native tool calling)

```
System prompt includes tool descriptions as XML-tagged text:

  <tools>
    <tool name="shell" …>
      <description>Execute a shell command</description>
      <parameters>{"command": {"type": "string"}}</parameters>
    </tool>
  </tools>

LLM responds with embedded XML:
  "Let me check. <tool_call>{"name":"shell","arguments":{"command":"ls"}}</tool_call>"

Agent parses XML tags, executes tools, returns results as:
  "<tool_result name="shell">file1.txt\nfile2.txt</tool_result>"

Conversation continues as Chat messages (role: assistant/user).
```

### Native Dispatcher (OpenAI function calling, Claude tool_use, etc.)

```
Tools sent as structured JSON in the API request:
  tools: [{ "type": "function", "function": { "name": "shell", … } }]

LLM responds with structured tool_calls field:
  response.tool_calls: [{ "id": "call_1", "function": { "name": "shell", "arguments": "{…}" } }]

Agent executes tools, returns results as:
  ConversationMessage::ToolResults([{ tool_call_id: "call_1", content: "…" }])

Conversation uses native AssistantToolCalls / ToolResults message types.
```

The dispatcher is selected automatically based on `provider.capabilities().native_tool_calling`. Providers that support native tools use `NativeToolDispatcher`; others fall back to `XmlToolDispatcher`.

---

*[← Overview](overview.md) | [Providers →](providers/)*
