# Provider Trait — Architecture Deep-Dive

> **Source**: `src/providers/traits.rs`
> **Factory**: `src/providers/mod.rs`
> **Parent doc**: [Architecture Overview](../overview.md)

---

## Purpose

The `Provider` trait abstracts interaction with Large Language Models (LLMs). Any service that can take a conversation and produce a text response — optionally with structured tool calls and streaming — implements this trait. The agent orchestrator never calls an LLM API directly; it always goes through `Box<dyn Provider>`.

**When to implement**: You want to integrate a new LLM service (e.g., Mistral, Cohere, a local model server, a custom fine-tune API).

---

## Trait Definition

```rust
#[async_trait]
pub trait Provider: Send + Sync {
    // ── Capability discovery ──────────────────────────────────

    /// Reports provider capabilities. Default: all false.
    fn capabilities(&self) -> ProviderCapabilities {
        ProviderCapabilities::default()
    }

    /// Converts tool specs to the provider's format.
    /// Default: PromptGuided (XML-based text injection).
    fn convert_tools(&self, tools: &[ToolSpec]) -> ToolsPayload {
        ToolsPayload::PromptGuided {
            instructions: build_tool_instructions_text(tools),
        }
    }

    fn supports_native_tools(&self) -> bool   // delegates to capabilities()
    fn supports_vision(&self) -> bool          // delegates to capabilities()
    fn supports_streaming(&self) -> bool       // default: false
    fn supports_streaming_tool_events(&self) -> bool  // default: false

    // ── Simple chat (text in → text out) ──────────────────────

    /// Send a single message. Default: delegates to chat_with_system.
    async fn simple_chat(&self, message: &str, model: &str, temperature: f64)
        -> anyhow::Result<String>

    /// REQUIRED. Send a message with optional system prompt.
    async fn chat_with_system(
        &self,
        system_prompt: Option<&str>,
        message: &str,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String>;

    /// Send a multi-turn conversation. Default: extracts system + last user.
    async fn chat_with_history(
        &self,
        messages: &[ChatMessage],
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String>

    // ── Structured chat (tool calling) ────────────────────────

    /// REQUIRED. Full chat with structured request/response.
    async fn chat(
        &self,
        request: ChatRequest<'_>,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<ChatResponse>;

    /// Chat with JSON tool schemas. Used by native tool dispatchers.
    async fn chat_with_tools(
        &self,
        messages: &[ChatMessage],
        tools: &[serde_json::Value],
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<ChatResponse>;

    // ── Streaming ─────────────────────────────────────────────

    fn stream_chat_with_system(
        &self,
        system_prompt: Option<&str>,
        message: &str,
        model: &str,
        temperature: f64,
        options: StreamOptions,
    ) -> stream::BoxStream<'static, StreamResult<StreamChunk>>;

    fn stream_chat_with_history(
        &self,
        messages: &[ChatMessage],
        model: &str,
        temperature: f64,
        options: StreamOptions,
    ) -> stream::BoxStream<'static, StreamResult<StreamChunk>>;

    fn stream_chat(
        &self,
        request: ChatRequest<'_>,
        model: &str,
        temperature: f64,
        options: StreamOptions,
    ) -> stream::BoxStream<'static, StreamResult<StreamEvent>>;

    // ── Lifecycle ─────────────────────────────────────────────

    /// Optional warmup (e.g., connection pooling, model loading).
    async fn warmup(&self) -> anyhow::Result<()> { Ok(()) }
}
```

### Required vs. Default Methods

| Method | Required? | Notes |
|--------|-----------|-------|
| `chat_with_system()` | **Yes** | Minimum viable provider — text in, text out |
| `chat()` | **Yes** | Structured request/response with tool support |
| `chat_with_tools()` | **Yes** | Native tool calling path |
| `capabilities()` | No | Default: all capabilities false |
| `convert_tools()` | No | Default: PromptGuided (XML text) |
| `simple_chat()` | No | Delegates to `chat_with_system` |
| `chat_with_history()` | No | Extracts system + last user message |
| `supports_streaming()` | No | Default: false |
| `stream_*` methods | No | Default: empty stream / delegates |
| `warmup()` | No | Default: no-op |

---

## Associated Types

### `ChatMessage`

```rust
pub struct ChatMessage {
    pub role: String,    // "system", "user", "assistant"
    pub content: String,
}
```

Constructors: `ChatMessage::system()`, `::user()`, `::assistant()`, `::tool()`

### `ChatRequest<'a>`

```rust
pub struct ChatRequest<'a> {
    pub messages: &'a [ChatMessage],
    pub tools: Option<&'a [ToolSpec]>,
    // … additional fields for tool_choice, max_tokens, etc.
}
```

### `ChatResponse`

```rust
pub struct ChatResponse {
    pub text: Option<String>,
    pub tool_calls: Vec<ToolCall>,
    pub usage: TokenUsage,
    pub reasoning_content: Option<String>,
}
```

### `ToolCall`

```rust
pub struct ToolCall {
    pub id: String,
    pub name: String,
    pub arguments: String,  // JSON string
}
```

### `TokenUsage`

```rust
pub struct TokenUsage {
    pub input_tokens: u64,
    pub output_tokens: u64,
    pub cached_input_tokens: u64,
}
```

### `ProviderCapabilities`

```rust
pub struct ProviderCapabilities {
    pub native_tool_calling: bool,  // Supports structured tool_calls in API
    pub vision: bool,               // Supports image input
    pub prompt_caching: bool,       // Supports prompt cache (Anthropic, etc.)
}
```

### `ToolsPayload` (enum)

Determines how tool schemas are sent to the provider:

```rust
pub enum ToolsPayload {
    Gemini(Vec<serde_json::Value>),
    Anthropic(Vec<serde_json::Value>),
    OpenAI(Vec<serde_json::Value>),
    PromptGuided { instructions: String },  // XML text fallback
}
```

### Streaming Types

```rust
pub struct StreamChunk {
    pub delta: String,
    pub reasoning: Option<String>,
    pub is_final: bool,
    pub token_count: Option<u64>,
}

pub enum StreamEvent {
    TextDelta(String),
    ToolCall { id: String, name: String, arguments: String },
    PreExecutedToolCall { ... },
    PreExecutedToolResult { ... },
    Final(ChatResponse),
}

pub struct StreamOptions {
    pub enabled: bool,
    pub count_tokens: bool,
}

pub enum StreamError {
    Http(String),
    Json(String),
    InvalidSse(String),
    Provider(String),
    Io(String),
}
```

---

## Existing Implementations

### Direct Providers

| Implementation | File | Key Characteristics |
|----------------|------|-------------------|
| `OpenAiProvider` | `openai.rs` | GPT models, native tool calling, streaming, vision |
| `AnthropicProvider` | `anthropic.rs` | Claude models, native tool calling, prompt caching |
| `OllamaProvider` | `ollama.rs` | Local models, streaming, optional tool calling |
| `GeminiProvider` | `gemini.rs` | Google models, Gemini-format tools |
| `AzureOpenAiProvider` | `azure_openai.rs` | Azure-hosted OpenAI, same API with Azure auth |
| `BedrockProvider` | `bedrock.rs` | AWS Bedrock, SigV4 auth, multiple model families |
| `OpenRouterProvider` | `openrouter.rs` | Multi-provider routing, OpenAI-compatible |
| `TelnyxProvider` | `telnyx.rs` | Telnyx AI models |
| `GlmProvider` | `glm.rs` | GLM/ChatGLM models |
| `CopilotProvider` | `copilot.rs` | GitHub Copilot API |
| `CompatibleProvider` | `compatible.rs` | Generic OpenAI-compatible endpoint (covers 50+ providers) |
| `ClaudeCodeProvider` | `claude_code.rs` | Claude Code CLI wrapper |
| `OpenAiCodexProvider` | `openai_codex.rs` | OpenAI Codex CLI |
| `GeminiCliProvider` | `gemini_cli.rs` | Gemini CLI wrapper |
| `KiloCliProvider` | `kilocli.rs` | Kilo CLI wrapper |

### Meta Providers

| Implementation | File | Purpose |
|----------------|------|---------|
| `ReliableProvider` | `reliable.rs` | Wraps a primary + fallback provider chain. Retries on failure, falls back to alternatives. |
| `RouterProvider` | `router.rs` | Routes requests to different providers/models based on query classification hints (e.g., `hint:code-review` → specific model). |

---

## Factory / Registration

**Location**: `src/providers/mod.rs`

The factory function takes the config and returns `Box<dyn Provider>`:

```
match config.default_provider {
    "openai"        → OpenAiProvider::new(credentials)
    "anthropic"     → AnthropicProvider::new(credentials)
    "ollama"        → OllamaProvider::new(base_url)
    "gemini"        → GeminiProvider::new(credentials)
    "azure-openai"  → AzureOpenAiProvider::new(credentials, deployment)
    "bedrock"       → BedrockProvider::new(region, credentials)
    "openrouter"    → OpenRouterProvider::new(credentials)
    "compatible"    → CompatibleProvider::new(base_url, credentials)
    …
}
```

### 3-Tier Credential Resolution

Credentials are resolved in this priority order:

1. **Explicit config** — `config.api_key` or provider-specific section
2. **Provider-specific env var** — `OPENAI_API_KEY`, `ANTHROPIC_API_KEY`, etc.
3. **Generic fallback** — `ZEROCLAW_API_KEY`

The factory function handles this resolution before constructing the provider.

### Wrapping with ReliableProvider

If `config.reliability` specifies fallback providers, the primary provider is wrapped:

```
let primary = build_provider(config.default_provider, ...);
let fallbacks = config.reliability.fallback_providers.iter()
    .map(|name| build_provider(name, ...));
ReliableProvider::new(primary, fallbacks, retry_config)
```

---

## Configuration

Relevant sections in `config.toml`:

```toml
# Primary provider
default_provider = "anthropic"
default_model = "claude-sonnet-4-20250514"
default_temperature = 0.7
api_key = "sk-..."  # or use ZEROCLAW_API_KEY env var

# Reliability wrapper
[reliability]
max_retries = 3
fallback_providers = ["openai", "ollama"]

# Model routing
[[model_routes]]
hint = "code-review"
provider = "anthropic"
model = "claude-sonnet-4-20250514"

[[model_routes]]
hint = "quick-answer"
provider = "ollama"
model = "llama3"
```

---

## Extension Guide — Adding a New Provider

### Step 1: Create the implementation file

Create `src/providers/my_provider.rs`:

```rust
use async_trait::async_trait;
use crate::providers::traits::*;

pub struct MyProvider {
    api_key: String,
    base_url: String,
    client: reqwest::Client,
}

impl MyProvider {
    pub fn new(api_key: String, base_url: Option<String>) -> Self {
        Self {
            api_key,
            base_url: base_url.unwrap_or_else(|| "https://api.myprovider.com/v1".into()),
            client: reqwest::Client::new(),
        }
    }
}

#[async_trait]
impl Provider for MyProvider {
    fn capabilities(&self) -> ProviderCapabilities {
        ProviderCapabilities {
            native_tool_calling: true,  // if your API supports it
            vision: false,
            prompt_caching: false,
        }
    }

    async fn chat_with_system(
        &self,
        system_prompt: Option<&str>,
        message: &str,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<String> {
        // Build HTTP request to your API
        // Parse response
        // Return text
        todo!()
    }

    async fn chat(
        &self,
        request: ChatRequest<'_>,
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<ChatResponse> {
        // Build structured request
        // Parse ChatResponse with tool_calls
        todo!()
    }

    async fn chat_with_tools(
        &self,
        messages: &[ChatMessage],
        tools: &[serde_json::Value],
        model: &str,
        temperature: f64,
    ) -> anyhow::Result<ChatResponse> {
        // Send tools as structured schemas
        todo!()
    }

    // Streaming (optional — implement if your API supports SSE)
    fn supports_streaming(&self) -> bool { true }

    fn stream_chat_with_system(
        &self,
        system_prompt: Option<&str>,
        message: &str,
        model: &str,
        temperature: f64,
        options: StreamOptions,
    ) -> stream::BoxStream<'static, StreamResult<StreamChunk>> {
        // Return a stream of chunks
        todo!()
    }

    // ... implement remaining stream methods
}
```

### Step 2: Register the module

In `src/providers/mod.rs`, add:

```rust
mod my_provider;
pub use my_provider::MyProvider;
```

### Step 3: Add to factory

In the factory `match` statement in `src/providers/mod.rs`:

```rust
"myprovider" => {
    let api_key = resolve_credential("MYPROVIDER_API_KEY", &config)?;
    Box::new(MyProvider::new(api_key, config.myprovider_base_url.clone()))
}
```

### Step 4: Add config fields (if needed)

In `src/config/schema.rs`:

```rust
pub myprovider_api_key: Option<String>,
pub myprovider_base_url: Option<String>,
```

### Step 5: Test

```bash
# Unit test
cargo test --lib providers::my_provider

# Integration test (requires API key)
MYPROVIDER_API_KEY=... cargo test --test integration_tests -- myprovider
```

---

## Testing Your Provider

Key test scenarios:

1. **Simple chat** — `simple_chat("Hello", "model-name", 0.7)` returns non-empty string.
2. **System prompt** — `chat_with_system(Some("You are a pirate"), "Hello", ...)` response reflects the system prompt.
3. **Tool calling** — `chat()` with tools returns `ChatResponse` with non-empty `tool_calls` when the model decides to use a tool.
4. **Streaming** — `stream_chat_with_system()` yields multiple `StreamChunk`s, the last with `is_final = true`.
5. **Error handling** — Invalid API key returns `Err`, not a panic.
6. **Credential resolution** — Factory correctly picks up env vars.

---

## Common Patterns & Gotchas

1. **OpenAI-compatible providers**: If your API speaks the OpenAI format, consider using `CompatibleProvider` with a custom `base_url` instead of writing a new implementation. This covers 50+ providers out of the box.

2. **`convert_tools()` matters**: If your API uses a different tool schema format (like Gemini or Anthropic), override `convert_tools()` to return the correct `ToolsPayload` variant. The agent will call this before sending tools to the API.

3. **Streaming is optional but valuable**: The gateway streams responses to WebSocket clients in real time. If you don't implement streaming, responses will appear all-at-once after the full LLM call completes.

4. **`warmup()` for connection pooling**: If your provider benefits from pre-warming (e.g., establishing a persistent connection or loading a local model), implement `warmup()`. It's called once at startup.

5. **The `chat()` method is the core path**: The agent orchestrator primarily calls `chat()` (or `stream_chat()`) for turn execution. `simple_chat()` and `chat_with_system()` are used for auxiliary operations (memory summarization, query classification).

6. **Token counting**: Populate `ChatResponse.usage` accurately — it drives cost tracking and budget enforcement.

---

*[← Architecture Overview](../overview.md) | [Channels →](../channels/)*
