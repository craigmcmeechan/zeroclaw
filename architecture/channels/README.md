# Channel Trait — Architecture Deep-Dive

> **Source**: `src/channels/traits.rs`
> **Factory**: `src/channels/mod.rs`
> **Parent doc**: [Architecture Overview](../overview.md)

---

## Purpose

The `Channel` trait abstracts bidirectional messaging platforms. A channel connects the agent to users through a specific communication medium — Telegram, Discord, Slack, email, IRC, or any other messaging system. Channels have two responsibilities:

1. **Listen** for inbound messages and push them into the agent via an mpsc channel.
2. **Send** outbound responses (including streaming drafts, reactions, and threading).

**When to implement**: You want to connect ZeroClaw to a new messaging platform (e.g., Microsoft Teams, Mastodon, a custom chat UI, an SMS gateway).

---

## Trait Definition

```rust
#[async_trait]
pub trait Channel: Send + Sync {
    // ── Required ──────────────────────────────────────────────

    /// Unique channel identifier (e.g., "telegram", "discord", "slack").
    fn name(&self) -> &str;

    /// Send an outbound message to a recipient.
    async fn send(&self, message: &SendMessage) -> anyhow::Result<()>;

    /// Start listening for inbound messages. Runs indefinitely.
    /// Push received messages into `tx`. This method should not return
    /// unless the channel is shutting down.
    async fn listen(
        &self,
        tx: tokio::sync::mpsc::Sender<ChannelMessage>,
    ) -> anyhow::Result<()>;

    // ── Health ────────────────────────────────────────────────

    /// Check if the channel is healthy. Default: always true.
    async fn health_check(&self) -> bool { true }

    // ── Typing indicators ─────────────────────────────────────

    async fn start_typing(&self, _recipient: &str) -> anyhow::Result<()> { Ok(()) }
    async fn stop_typing(&self, _recipient: &str) -> anyhow::Result<()> { Ok(()) }

    // ── Draft / streaming updates ─────────────────────────────

    /// Does this channel support editing a message in-place (for streaming)?
    fn supports_draft_updates(&self) -> bool { false }

    /// Does this channel support splitting long responses into multiple messages?
    fn supports_multi_message_streaming(&self) -> bool { false }

    /// Delay between multi-message chunks (milliseconds). Default: 800ms.
    fn multi_message_delay_ms(&self) -> u64 { 800 }

    /// Send an initial draft message, return its message_id for later updates.
    async fn send_draft(&self, _message: &SendMessage)
        -> anyhow::Result<Option<String>> { Ok(None) }

    /// Update draft content (full replacement).
    async fn update_draft(
        &self, _recipient: &str, _message_id: &str, _text: &str,
    ) -> anyhow::Result<()> { Ok(()) }

    /// Update draft with progress indicator (e.g., "🔧 Running shell…").
    async fn update_draft_progress(
        &self, _recipient: &str, _message_id: &str, _text: &str,
    ) -> anyhow::Result<()> { Ok(()) }

    /// Finalize draft — last update, removes any "typing" indicator.
    async fn finalize_draft(
        &self, _recipient: &str, _message_id: &str, _text: &str,
    ) -> anyhow::Result<()> { Ok(()) }

    /// Cancel / delete a draft message.
    async fn cancel_draft(
        &self, _recipient: &str, _message_id: &str,
    ) -> anyhow::Result<()> { Ok(()) }

    // ── Reactions & message management ────────────────────────

    async fn add_reaction(
        &self, _channel_id: &str, _message_id: &str, _emoji: &str,
    ) -> anyhow::Result<()> { Ok(()) }

    async fn remove_reaction(
        &self, _channel_id: &str, _message_id: &str, _emoji: &str,
    ) -> anyhow::Result<()> { Ok(()) }

    async fn pin_message(
        &self, _channel_id: &str, _message_id: &str,
    ) -> anyhow::Result<()> { Ok(()) }

    async fn unpin_message(
        &self, _channel_id: &str, _message_id: &str,
    ) -> anyhow::Result<()> { Ok(()) }

    async fn redact_message(
        &self, _channel_id: &str, _message_id: &str, _reason: Option<String>,
    ) -> anyhow::Result<()> { Ok(()) }
}
```

### Required vs. Default Methods

| Method | Required? | Notes |
|--------|-----------|-------|
| `name()` | **Yes** | Return your channel identifier string |
| `send()` | **Yes** | Deliver an outbound message |
| `listen()` | **Yes** | Long-running listener — push `ChannelMessage` into `tx` |
| `health_check()` | No | Default: `true` |
| `start_typing()` / `stop_typing()` | No | Default: no-op |
| `supports_draft_updates()` | No | Default: `false` — override if your platform supports message editing |
| Draft methods (`send_draft`, `update_draft`, etc.) | No | Default: no-op — implement if `supports_draft_updates` is true |
| Reaction/pin/redact methods | No | Default: no-op — implement if your platform supports them |

---

## Associated Types

### `ChannelMessage` (inbound)

Represents a message received from a user on any platform:

```rust
pub struct ChannelMessage {
    pub id: String,                              // Platform message ID
    pub sender: String,                          // User identifier
    pub reply_target: String,                    // Where to send the response
    pub content: String,                         // Message text
    pub channel: String,                         // Channel name (e.g., "telegram")
    pub timestamp: String,                       // ISO 8601 timestamp
    pub thread_ts: Option<String>,               // Thread ID (Slack ts, Discord thread)
    pub interruption_scope_id: Option<String>,   // Reply thread isolation
    pub attachments: Vec<MediaAttachment>,        // Audio, images, video
}
```

### `SendMessage` (outbound)

Represents a message being sent to a user:

```rust
pub struct SendMessage {
    pub content: String,
    pub recipient: String,
    pub subject: Option<String>,                  // For email channels
    pub thread_ts: Option<String>,                // Reply in thread
    pub cancellation_token: Option<CancellationToken>,  // Interruptible delivery
}

// Constructors:
SendMessage::new(content, recipient)
SendMessage::with_subject(content, recipient, subject)
SendMessage::in_thread(content, recipient, thread_ts)
SendMessage::with_cancellation(content, recipient, token)
```

### `MediaAttachment`

```rust
pub struct MediaAttachment {
    pub media_type: MediaType,  // Audio, Image, Video, Document
    pub url: Option<String>,
    pub data: Option<Vec<u8>>,
    pub mime_type: Option<String>,
    pub filename: Option<String>,
}
```

---

## Existing Implementations

### Messaging Platforms

| Implementation | File | Protocol | Key Features |
|----------------|------|----------|-------------|
| Telegram | `telegram.rs` | HTTP long-polling | Draft updates, reactions, threading, vision (photos) |
| Discord | `discord.rs` | WebSocket gateway | Draft updates, reactions, threads, slash commands |
| Slack | `slack.rs` | WebSocket (Socket Mode) | Draft updates, reactions, threads, `thread_ts` |
| Signal | `signal.rs` | Signal CLI / REST | End-to-end encryption, group messages |
| WhatsApp | `whatsapp.rs` | WhatsApp Business API | Templates, media, reactions |
| WhatsApp Web | `whatsapp_web.rs` | wa-rs (feature-gated) | Web protocol, session persistence |
| Email | `email_channel.rs` | IMAP/SMTP (lettre) | Subject lines, HTML, attachments |
| Gmail Push | `gmail_push.rs` | Google Pub/Sub | Push notifications, no polling |
| IRC | `irc.rs` | IRC protocol | Channels, private messages |
| Matrix | `matrix.rs` | Matrix SDK (E2EE) | End-to-end encryption, rooms |
| Twitter/X | `twitter.rs` | Twitter API v2 | DMs, mentions |
| Bluesky | `bluesky.rs` | AT Protocol | Posts, DMs |
| Nostr | `nostr.rs` | Nostr protocol | Relays, NIP-04 encryption |
| Reddit | `reddit.rs` | Reddit API | DMs, comment replies |
| Mattermost | `mattermost.rs` | WebSocket | Draft updates, reactions, threads |
| Notion | `notion.rs` | Notion API | Page comments, database entries |
| DingTalk | `dingtalk.rs` | DingTalk API | Corporate messaging |
| Lark/Feishu | `lark.rs` | Lark API | Corporate messaging |
| WeCom | `wecom.rs` | WeCom API | Enterprise WeChat |
| QQ | `qq.rs` | QQ API | Consumer messaging |
| Nextcloud Talk | `nextcloud_talk.rs` | Nextcloud API | Self-hosted chat |
| MQTT | `mqtt.rs` | MQTT protocol | IoT message bus |
| WATI | `wati.rs` | WATI WhatsApp API | WhatsApp via WATI |

### Special-Purpose Channels

| Implementation | File | Purpose |
|----------------|------|---------|
| CLI | `cli.rs` | Terminal stdin/stdout (interactive mode) |
| Webhook | `webhook.rs` | Generic HTTP webhook ingest |
| MoChat | `mochat.rs` | Internal debug/test channel |
| ClawdTalk | `clawdtalk.rs` | ZeroClaw-native chat protocol |
| LinQ | `linq.rs` | Custom enterprise channel |
| ACP Server | `acp_server.rs` | Agent Communication Protocol server |
| Voice Call | `voice_call.rs` | Voice telephony |
| iMessage | `imessage.rs` | Apple iMessage (macOS only) |

### Supporting Modules

| File | Purpose |
|------|---------|
| `debounce.rs` | Message debouncing to prevent rapid-fire responses |
| `link_enricher.rs` | URL preview/metadata extraction for outbound messages |
| `media_pipeline.rs` | Media processing for attachments (resize, transcode) |
| `transcription.rs` | Audio-to-text for voice messages |
| `tts.rs` | Text-to-speech for voice responses |
| `voice_wake.rs` | Wake-word detection (feature-gated) |
| `session_backend.rs` | Session persistence trait |
| `session_sqlite.rs` | SQLite session storage |
| `session_store.rs` | In-memory session store |

---

## Factory / Registration

**Location**: `src/channels/mod.rs`

Channels are not constructed via a single match statement. Instead, the factory iterates over enabled channels in the config and explicitly constructs each one:

```rust
let mut channels: Vec<Box<dyn Channel>> = Vec::new();

if let Some(telegram_config) = &config.channels_config.telegram {
    if telegram_config.enabled {
        channels.push(Box::new(
            TelegramChannel::new(telegram_config.clone())
        ));
    }
}

if let Some(discord_config) = &config.channels_config.discord {
    if discord_config.enabled {
        channels.push(Box::new(
            DiscordChannel::new(discord_config.clone())
        ));
    }
}

// ... repeat for each channel type
```

Channels are then run concurrently in daemon mode:

```rust
for channel in channels {
    let tx = tx.clone();
    tokio::spawn(async move {
        channel.listen(tx).await
    });
}
```

---

## Configuration

Each channel has its own config section. Examples:

```toml
[channels.telegram]
enabled = true
bot_token = "123456:ABC-..."  # or TELEGRAM_BOT_TOKEN env
allowed_users = ["user_id_1", "user_id_2"]

[channels.discord]
enabled = true
bot_token = "..."  # or DISCORD_BOT_TOKEN env
guild_id = "..."

[channels.slack]
enabled = true
bot_token = "xoxb-..."
app_token = "xapp-..."

[channels.email]
enabled = true
imap_host = "imap.gmail.com"
smtp_host = "smtp.gmail.com"
username = "agent@example.com"
password = "..."
```

The `ChannelConfig` trait in `src/config/traits.rs` provides a uniform interface:

```rust
pub trait ChannelConfig: Serialize + Deserialize {
    const NAME: &'static str;
    fn name(&self) -> &'static str { Self::NAME }
}
```

---

## Extension Guide — Adding a New Channel

### Step 1: Create the implementation file

Create `src/channels/my_platform.rs`:

```rust
use async_trait::async_trait;
use crate::channels::traits::*;

pub struct MyPlatformChannel {
    config: MyPlatformConfig,
    client: reqwest::Client,
}

impl MyPlatformChannel {
    pub fn new(config: MyPlatformConfig) -> Self {
        Self {
            config,
            client: reqwest::Client::new(),
        }
    }
}

#[async_trait]
impl Channel for MyPlatformChannel {
    fn name(&self) -> &str {
        "myplatform"
    }

    async fn send(&self, message: &SendMessage) -> anyhow::Result<()> {
        // POST to your platform's send-message API
        // Handle thread_ts for threaded replies
        // Handle subject if relevant (email-like)
        todo!()
    }

    async fn listen(
        &self,
        tx: tokio::sync::mpsc::Sender<ChannelMessage>,
    ) -> anyhow::Result<()> {
        // Long-polling, WebSocket, or webhook receiver loop
        loop {
            let raw_message = self.poll_for_messages().await?;

            let channel_message = ChannelMessage {
                id: raw_message.id,
                sender: raw_message.author_id,
                reply_target: raw_message.chat_id,
                content: raw_message.text,
                channel: "myplatform".into(),
                timestamp: chrono::Utc::now().to_rfc3339(),
                thread_ts: raw_message.thread_id,
                interruption_scope_id: None,
                attachments: vec![],
            };

            tx.send(channel_message).await?;
        }
    }

    // Optional: streaming support
    fn supports_draft_updates(&self) -> bool { true }

    async fn send_draft(&self, message: &SendMessage)
        -> anyhow::Result<Option<String>> {
        // Send initial message, return its ID
        let msg_id = self.send_and_get_id(message).await?;
        Ok(Some(msg_id))
    }

    async fn update_draft(
        &self, _recipient: &str, message_id: &str, text: &str,
    ) -> anyhow::Result<()> {
        // Edit the message in-place
        self.edit_message(message_id, text).await
    }

    async fn finalize_draft(
        &self, _recipient: &str, message_id: &str, text: &str,
    ) -> anyhow::Result<()> {
        self.edit_message(message_id, text).await
    }
}
```

### Step 2: Register the module

In `src/channels/mod.rs`:

```rust
mod my_platform;
pub use my_platform::MyPlatformChannel;
```

### Step 3: Add config

In `src/config/schema.rs`, add a config struct:

```rust
#[derive(Debug, Clone, Serialize, Deserialize, JsonSchema)]
pub struct MyPlatformConfig {
    pub enabled: bool,
    pub api_token: Option<String>,
    pub base_url: Option<String>,
}
```

Add it to `ChannelsConfig`:

```rust
pub struct ChannelsConfig {
    // ... existing channels ...
    pub myplatform: Option<MyPlatformConfig>,
}
```

### Step 4: Add to factory

In `src/channels/mod.rs`, add the construction block:

```rust
if let Some(myplatform_config) = &config.channels_config.myplatform {
    if myplatform_config.enabled {
        channels.push(Box::new(
            MyPlatformChannel::new(myplatform_config.clone())
        ));
    }
}
```

### Step 5: Test

```bash
cargo test --lib channels::my_platform
```

---

## Testing Your Channel

Key test scenarios:

1. **Send** — `send(SendMessage::new("Hello", "recipient"))` completes without error.
2. **Listen** — `listen(tx)` pushes a `ChannelMessage` when the platform sends a message (mock the API).
3. **Threading** — `send(SendMessage::in_thread("Reply", "recipient", "thread_id"))` sends in the correct thread.
4. **Draft lifecycle** — `send_draft` → `update_draft` → `finalize_draft` sequence works.
5. **Health check** — `health_check()` returns false when the API is unreachable.

---

## Common Patterns & Gotchas

1. **`listen()` must not return**: The method runs indefinitely as a background tokio task. If the connection drops, reconnect with exponential backoff inside the loop. Only return on fatal errors or shutdown signal.

2. **Thread isolation via `interruption_scope_id`**: If your platform supports threads, set `interruption_scope_id` on the `ChannelMessage` so the agent can scope its conversation history per-thread.

3. **Draft update lifecycle**: The flow is always: `send_draft` → (0 or more `update_draft`/`update_draft_progress`) → `finalize_draft` or `cancel_draft`. Never call `update_draft` without a prior `send_draft`.

4. **Debouncing**: If your platform emits rapid-fire messages (e.g., user typing corrections), use the debounce module (`src/channels/debounce.rs`) to batch them.

5. **Media attachments**: If your platform supports images/audio/video, populate `ChannelMessage.attachments` so the agent can process them via multimodal providers.

6. **Rate limits**: Most platforms have API rate limits. Implement back-off in both `send()` and `listen()` to avoid hitting them.

7. **Session backend**: The channel itself doesn't manage sessions — that's handled by `session_backend.rs` / `session_sqlite.rs`. But the `thread_ts` field is the key for session lookup.

---

*[← Providers](../providers/) | [Tools →](../tools/)*
