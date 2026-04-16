# Tool Trait — Architecture Deep-Dive

> **Source**: `src/tools/traits.rs`
> **Factory**: `src/tools/mod.rs`
> **Parent doc**: [Architecture Overview](../overview.md)

---

## Purpose

The `Tool` trait defines an action that the agent can invoke during a conversation. When an LLM decides it needs to execute a command, read a file, search the web, or perform any side effect, it issues a tool call that maps to a `Tool` implementation. The agent orchestrator looks up the tool by name, validates the arguments, checks security policy, and delegates to `execute()`.

Tools are the agent's hands — they bridge the gap between language understanding and real-world action.

**When to implement**: You want to give the agent a new capability (e.g., query a database, call a third-party API, interact with a new service, expose a hardware operation).

---

## Trait Definition

```rust
#[async_trait]
pub trait Tool: Send + Sync {
    /// Unique tool name (used in LLM tool-call payloads).
    fn name(&self) -> &str;

    /// Human-readable description (shown to the LLM in the system prompt).
    fn description(&self) -> &str;

    /// JSON Schema defining the tool's input parameters.
    fn parameters_schema(&self) -> serde_json::Value;

    /// Execute the tool with the given arguments.
    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult>;

    /// Build the ToolSpec from name + description + schema.
    /// Default implementation — rarely needs overriding.
    fn spec(&self) -> ToolSpec {
        ToolSpec {
            name: self.name().to_string(),
            description: self.description().to_string(),
            parameters: self.parameters_schema(),
        }
    }
}
```

**All five methods are effectively required** — `spec()` has a sensible default, but the other four must be implemented for the tool to be functional.

---

## Associated Types

### `ToolResult`

The return value from tool execution:

```rust
pub struct ToolResult {
    pub success: bool,
    pub output: String,
    pub error: Option<String>,
}
```

- `success = true` — tool ran successfully, `output` contains the result text.
- `success = false` — tool failed, `error` contains the error description.

The agent sends `output` (or `error`) back to the LLM as the tool result.

### `ToolSpec`

The schema description sent to the LLM:

```rust
pub struct ToolSpec {
    pub name: String,
    pub description: String,
    pub parameters: serde_json::Value,  // JSON Schema object
}
```

The `parameters` field must be a valid JSON Schema describing the tool's input. Example:

```json
{
  "type": "object",
  "properties": {
    "command": {
      "type": "string",
      "description": "The shell command to execute"
    },
    "working_dir": {
      "type": "string",
      "description": "Optional working directory"
    }
  },
  "required": ["command"]
}
```

---

## Existing Implementations

ZeroClaw ships with 100+ tools organized by category:

### Core System Tools

| Tool | File | Purpose |
|------|------|---------|
| Shell | `shell.rs` | Execute shell commands (sandboxed) |
| File Read | `file_read.rs` | Read file contents |
| File Write | `file_write.rs` | Write/create files |
| File Edit | `file_edit.rs` | Search-and-replace in files |
| Glob Search | `glob_search.rs` | Find files by pattern |
| Content Search | `content_search.rs` | Grep/search within files |
| Ask User | `ask_user.rs` | Prompt the user for input |
| Escalate | `escalate.rs` | Escalate to human operator |
| Calculator | `calculator.rs` | Mathematical calculations |
| Sessions | `sessions.rs` | Session management operations |

### Web & Browser Tools

| Tool | File | Purpose |
|------|------|---------|
| Web Fetch | `web_fetch.rs` | HTTP GET to fetch web content |
| Web Search | `web_search_tool.rs` | Search engines (Google, Bing, DuckDuckGo, etc.) |
| Browser | `browser.rs` | Full browser automation (fantoccini) |
| Browser Open | `browser_open.rs` | Open URL in browser |
| Browser Delegate | `browser_delegate.rs` | Delegate browser tasks |
| Text Browser | `text_browser.rs` | Lightweight text-only browser |
| HTTP Request | `http_request.rs` | Arbitrary HTTP requests |
| Screenshot | `screenshot.rs` | Capture screenshots |

### Memory Tools

| Tool | File | Purpose |
|------|------|---------|
| Memory Store | `memory_store.rs` | Store a memory entry |
| Memory Recall | `memory_recall.rs` | Search/recall memories |
| Memory Forget | `memory_forget.rs` | Delete a memory entry |
| Memory Export | `memory_export.rs` | Export memories (GDPR) |
| Memory Purge | `memory_purge.rs` | Purge by namespace/session |

### Scheduling Tools

| Tool | File | Purpose |
|------|------|---------|
| Cron Add | `cron_add.rs` | Schedule a new cron job |
| Cron List | `cron_list.rs` | List existing cron jobs |
| Cron Update | `cron_update.rs` | Modify a cron job |
| Cron Remove | `cron_remove.rs` | Delete a cron job |
| Cron Run | `cron_run.rs` | Manually trigger a cron job |
| Cron Runs | `cron_runs.rs` | View recent cron run history |
| Schedule | `schedule.rs` | One-off scheduled tasks |

### SOP (Standard Operating Procedure) Tools

| Tool | File | Purpose |
|------|------|---------|
| SOP Execute | `sop_execute.rs` | Start an SOP workflow |
| SOP List | `sop_list.rs` | List available SOPs |
| SOP Status | `sop_status.rs` | Check SOP run status |
| SOP Advance | `sop_advance.rs` | Advance SOP to next step |
| SOP Approve | `sop_approve.rs` | Approve an SOP step |

### Integration Tools

| Tool | File | Purpose |
|------|------|---------|
| Notion | `notion_tool.rs` | Notion API operations |
| Discord Search | `discord_search.rs` | Search Discord history |
| Jira | `jira_tool.rs` | Jira issue operations |
| Google Workspace | `google_workspace.rs` | Google Docs/Sheets/Drive |
| Microsoft 365 | `microsoft365/` | Outlook, Teams, SharePoint |
| LinkedIn | `linkedin.rs` | LinkedIn operations |
| Composio | `composio.rs` | Composio OAuth integrations |
| Pushover | `pushover.rs` | Push notifications |
| Weather | `weather_tool.rs` | Weather data |

### Agent & Delegation Tools

| Tool | File | Purpose |
|------|------|---------|
| Delegate | `delegate.rs` | Delegate task to sub-agent |
| Swarm | `swarm.rs` | Multi-agent swarm coordination |
| LLM Task | `llm_task.rs` | Sub-LLM call for isolated reasoning |
| Model Switch | `model_switch.rs` | Switch model mid-conversation |
| Model Routing Config | `model_routing_config.rs` | Update model routing rules |
| Node Tool | `node_tool.rs` | Multi-node coordination |

### Hardware Tools

| Tool | File | Purpose |
|------|------|---------|
| Hardware Board Info | `hardware_board_info.rs` | Query board capabilities |
| Hardware Memory Map | `hardware_memory_map.rs` | Read hardware memory map |
| Hardware Memory Read | `hardware_memory_read.rs` | Read specific memory addresses |

### MCP (Model Context Protocol) Tools

| Tool | File | Purpose |
|------|------|---------|
| MCP Client | `mcp_client.rs` | MCP server connection |
| MCP Tool | `mcp_tool.rs` | Execute MCP-provided tools |
| MCP Deferred | `mcp_deferred.rs` | Lazy-loaded MCP tools |
| MCP Protocol | `mcp_protocol.rs` | MCP protocol handling |
| MCP Transport | `mcp_transport.rs` | MCP transport layer |

### Utility Tools

| Tool | File | Purpose |
|------|------|---------|
| Image Gen | `image_gen.rs` | Generate images (DALL-E, etc.) |
| Image Info | `image_info.rs` | Extract image metadata |
| PDF Read | `pdf_read.rs` | Read PDF contents |
| Git Operations | `git_operations.rs` | Git commands |
| Claude Code | `claude_code.rs` | Delegate to Claude Code CLI |
| Codex CLI | `codex_cli.rs` | Delegate to Codex CLI |
| Gemini CLI | `gemini_cli.rs` | Delegate to Gemini CLI |
| Knowledge | `knowledge_tool.rs` | RAG knowledge base queries |
| Read Skill | `read_skill.rs` | Read skill definitions |
| Workspace | `workspace_tool.rs` | Workspace operations |
| Poll | `poll.rs` | Create polls |
| Canvas | `canvas.rs` | Drawing canvas |
| Pipeline | `pipeline.rs` | Multi-step tool pipelines |
| Report Templates | `report_template_tool.rs` | Generate reports |
| Reaction | `reaction.rs` | Add reactions to messages |
| Backup | `backup_tool.rs` | Backup operations |
| Security Ops | `security_ops.rs` | Security operations |
| Data Management | `data_management.rs` | Data operations |
| Cloud Ops | `cloud_ops.rs` | Cloud infrastructure operations |
| Verifiable Intent | `verifiable_intent.rs` | Issue/verify credentials |
| Tool Search | `tool_search.rs` | Search available tools |
| Skill HTTP | `skill_http.rs` | HTTP-based skill execution |
| CLI Discovery | `cli_discovery.rs` | Discover available CLI tools |

---

## Factory / Registration

**Location**: `src/tools/mod.rs`

Tools are assembled into a `Vec<Box<dyn Tool>>` using conditional logic based on the config and security policy. This is not a simple match — it's a conditional build:

```rust
pub fn build_tools(
    config: &Config,
    security_policy: &SecurityPolicy,
    runtime: &dyn RuntimeAdapter,
    memory: Arc<dyn Memory>,
    // ... other dependencies
) -> Vec<Box<dyn Tool>> {
    let mut tools: Vec<Box<dyn Tool>> = Vec::new();

    // Always-included core tools
    tools.push(Box::new(ShellTool::new(runtime, security_policy)));
    tools.push(Box::new(FileReadTool::new(security_policy)));
    tools.push(Box::new(FileWriteTool::new(security_policy)));
    tools.push(Box::new(FileEditTool::new(security_policy)));
    tools.push(Box::new(GlobSearchTool::new(security_policy)));
    tools.push(Box::new(MemoryStoreTool::new(memory.clone())));
    tools.push(Box::new(MemoryRecallTool::new(memory.clone())));
    // ...

    // Conditional tools
    if config.browser.enabled {
        tools.push(Box::new(BrowserTool::new(config.browser.clone())));
    }

    if config.web_search.enabled {
        tools.push(Box::new(WebSearchTool::new(config.web_search.clone())));
    }

    // MCP tools (dynamically loaded from MCP servers)
    for mcp_config in &config.mcp.servers {
        let mcp_tools = load_mcp_tools(mcp_config);
        tools.extend(mcp_tools);
    }

    // Peripheral tools (from connected hardware boards)
    for peripheral in peripherals {
        tools.extend(peripheral.tools());
    }

    tools
}
```

**Key pattern**: All tools receive `SecurityPolicy` (or specific dependencies like `Arc<dyn Memory>`) via constructor injection. The factory is the single place where dependency wiring happens.

---

## Configuration

Tools are enabled/disabled and configured through various config sections:

```toml
# Shell tool
[shell_tool]
timeout_seconds = 30
max_output_bytes = 65536

# Browser automation
[browser]
enabled = true
headless = true
browser_path = "/usr/bin/chromium"

# Web search
[web_search]
enabled = true
provider = "duckduckgo"  # or "google", "bing", "searxng"
api_key = "..."

# HTTP requests
[http_request]
enabled = true
allowed_domains = ["api.example.com", "*.internal.corp"]

# MCP servers (dynamically loaded tools)
[[mcp.servers]]
name = "my-mcp-server"
command = "npx"
args = ["-y", "@my/mcp-server"]
```

---

## Extension Guide — Adding a New Tool

### Step 1: Create the implementation file

Create `src/tools/my_tool.rs`:

```rust
use async_trait::async_trait;
use crate::tools::traits::*;
use serde_json::json;

pub struct MyTool {
    // Any dependencies your tool needs
    api_key: String,
}

impl MyTool {
    pub fn new(api_key: String) -> Self {
        Self { api_key }
    }
}

#[async_trait]
impl Tool for MyTool {
    fn name(&self) -> &str {
        "my_tool"
    }

    fn description(&self) -> &str {
        "A brief description of what this tool does. \
         The LLM reads this to decide when to use the tool."
    }

    fn parameters_schema(&self) -> serde_json::Value {
        json!({
            "type": "object",
            "properties": {
                "query": {
                    "type": "string",
                    "description": "The search query"
                },
                "limit": {
                    "type": "integer",
                    "description": "Maximum number of results",
                    "default": 10
                }
            },
            "required": ["query"]
        })
    }

    async fn execute(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
        let query = args["query"]
            .as_str()
            .ok_or_else(|| anyhow::anyhow!("Missing 'query' parameter"))?;

        let limit = args["limit"].as_u64().unwrap_or(10);

        // Do the actual work
        match self.do_search(query, limit).await {
            Ok(results) => Ok(ToolResult {
                success: true,
                output: results,
                error: None,
            }),
            Err(e) => Ok(ToolResult {
                success: false,
                output: String::new(),
                error: Some(e.to_string()),
            }),
        }
    }
}
```

### Step 2: Register the module

In `src/tools/mod.rs`:

```rust
mod my_tool;
pub use my_tool::MyTool;
```

### Step 3: Add to the tool registry

In the `build_tools()` function in `src/tools/mod.rs`:

```rust
// Conditional inclusion
if config.my_tool.enabled {
    tools.push(Box::new(MyTool::new(
        config.my_tool.api_key.clone().unwrap_or_default(),
    )));
}
```

### Step 4: Add config (if needed)

In `src/config/schema.rs`:

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct MyToolConfig {
    pub enabled: bool,
    pub api_key: Option<String>,
}
```

### Step 5: Optionally add a tool description file

Create `tool_descriptions/my_tool.md` with a detailed description for documentation and LLM context.

### Step 6: Test

```bash
cargo test --lib tools::my_tool
```

---

## Testing Your Tool

Key test scenarios:

1. **Happy path** — Valid arguments → `ToolResult { success: true, output: "..." }`.
2. **Missing required param** — Returns error, not panic.
3. **Invalid param type** — String where integer expected → graceful error.
4. **API failure** — External service down → `ToolResult { success: false, error: Some("...") }`.
5. **Schema validation** — `parameters_schema()` returns valid JSON Schema.
6. **Name uniqueness** — `name()` doesn't collide with existing tools.

---

## Common Patterns & Gotchas

1. **Return `ToolResult`, not `Err`**: For expected failures (API errors, invalid input, no results), return `Ok(ToolResult { success: false, error: Some("...") })` instead of `Err(...)`. Reserve `Err` for truly unexpected situations (bugs, infrastructure failures). The LLM needs to see the error message to adjust its approach.

2. **Description quality matters**: The `description()` string is what the LLM reads to decide when to use your tool. Be specific about what the tool does and when it should (and shouldn't) be used. Include examples of inputs in the description if the tool has non-obvious behavior.

3. **JSON Schema precision**: The `parameters_schema()` must be valid JSON Schema. Use `"description"` on each property — the LLM uses these to understand what to pass. Mark required fields in `"required"`. Use `"default"` for optional parameters.

4. **Security policy awareness**: If your tool accepts file paths, validate them against the workspace boundary. If it executes external commands, consider whether it should go through the sandbox. Tools receive security context via constructor injection.

5. **Output size**: Tool output is sent back to the LLM as context. Keep output concise — truncate large results. The LLM's context window is finite and expensive.

6. **Tool names must be unique**: The agent looks up tools by `name()`. If two tools share a name, the second one shadows the first. Use descriptive, namespaced names (e.g., `jira_create_issue` rather than `create`).

7. **MCP tools are dynamic**: Tools loaded from MCP servers are discovered at runtime and added to the registry alongside built-in tools. They follow the same `Tool` trait contract but are constructed by the MCP client.

8. **Peripheral tools are injected**: Hardware peripherals expose tools via `Peripheral::tools()`. These are added to the registry during `build_tools()` and behave identically to built-in tools.

---

## Tool Execution Security Flow

Every tool execution goes through this security pipeline (see [Data Flow](../data-flow.md) for the full diagram):

```
LLM requests tool call
  → Tool registry lookup by name
  → SecurityPolicy validation:
    → Is tool in allowed set?
    → For shell: classify command risk (Low/Medium/High)
    → Check action rate limit (max_actions_per_hour)
    → If Supervised + Medium risk: approval gate
    → If block_high_risk + High risk: reject
  → RuntimeAdapter.build_shell_command() (for shell tools)
  → Sandbox execution (for shell tools)
  → Credential scrubbing on output
  → CostTracker recording
  → Return ToolResult to LLM
```

---

*[← Channels](../channels/) | [Memory →](../memory/)*
