# Runtime Trait — Architecture Deep-Dive

> **Source**: `src/runtime/traits.rs`
> **Factory**: `src/runtime/mod.rs`
> **Parent doc**: [Architecture Overview](../overview.md)

---

## Purpose

The `RuntimeAdapter` trait abstracts the execution environment in which the agent runs. It answers questions like: "Can I execute shell commands?", "Can I read/write files?", "Where should I store data?", and "How should I construct a shell command?"

This abstraction lets the same agent binary run natively on a host, inside a Docker container, or (eventually) in a WebAssembly sandbox — each with different capabilities, paths, and security constraints.

**When to implement**: You want the agent to run in a new execution environment (e.g., Kubernetes pod, Firecracker microVM, remote SSH host, cloud function).

---

## Trait Definition

```rust
pub trait RuntimeAdapter: Send + Sync {
    // ── Identification ────────────────────────────────────────

    /// Runtime name (e.g., "native", "docker", "wasm").
    fn name(&self) -> &str;

    // ── Capability queries ────────────────────────────────────

    /// Whether the agent can execute shell commands in this runtime.
    fn has_shell_access(&self) -> bool;

    /// Whether the agent can read/write the filesystem.
    fn has_filesystem_access(&self) -> bool;

    /// Whether this runtime supports long-running background processes.
    fn supports_long_running(&self) -> bool;

    // ── Environment ───────────────────────────────────────────

    /// Root path for persistent storage (data, memory DB, etc.).
    fn storage_path(&self) -> std::path::PathBuf;

    /// Memory budget in bytes. 0 = unlimited. Default: 0.
    fn memory_budget(&self) -> usize { 0 }

    // ── Command construction ──────────────────────────────────

    /// Build a shell command appropriate for this runtime.
    /// Native: direct process spawn.
    /// Docker: wraps in `docker exec`.
    /// Wasm: may refuse or sandbox.
    fn build_shell_command(&self, command: &str) -> tokio::process::Command;
}
```

**Key design choice**: This trait is **not** `async_trait`. Runtime capability queries are pure, synchronous property lookups. Only `build_shell_command` does meaningful work, and it returns a `Command` (not a future) — execution is left to the caller.

### Method Summary

| Method | Required? | Notes |
|--------|-----------|-------|
| `name()` | **Yes** | Runtime identifier |
| `has_shell_access()` | **Yes** | Gate for shell tool availability |
| `has_filesystem_access()` | **Yes** | Gate for file tool availability |
| `supports_long_running()` | **Yes** | Gate for background process tools |
| `storage_path()` | **Yes** | Where to write memory.db, logs, etc. |
| `memory_budget()` | No | Default: 0 (unlimited) |
| `build_shell_command()` | **Yes** | Constructs the actual OS command |

---

## Existing Implementations

| Implementation | File | Key Characteristics |
|----------------|------|-------------------|
| `NativeRuntime` | `native.rs` | **Default**. Full host access. Shell via `sh -c` (Unix) or `cmd /C` (Windows). Storage at `~/.zeroclaw/workspace/`. |
| `DockerRuntime` | `docker.rs` | Runs commands inside a Docker container via `docker exec`. Storage mapped to container volume. Supports long-running. |
| `WasmRuntime` | `wasm.rs` | **Stub**. No shell, no filesystem, no long-running. Returns errors for unsupported operations. Placeholder for future WASI integration. |

### NativeRuntime — The Default

```rust
impl RuntimeAdapter for NativeRuntime {
    fn name(&self) -> &str { "native" }
    fn has_shell_access(&self) -> bool { true }
    fn has_filesystem_access(&self) -> bool { true }
    fn supports_long_running(&self) -> bool { true }

    fn storage_path(&self) -> PathBuf {
        // ~/.zeroclaw/workspace/ (configurable)
        self.workspace_dir.clone()
    }

    fn build_shell_command(&self, command: &str) -> tokio::process::Command {
        #[cfg(unix)]
        {
            let mut cmd = tokio::process::Command::new("sh");
            cmd.arg("-c").arg(command);
            cmd
        }
        #[cfg(windows)]
        {
            let mut cmd = tokio::process::Command::new("cmd");
            cmd.arg("/C").arg(command);
            cmd
        }
    }
}
```

### DockerRuntime

```rust
impl RuntimeAdapter for DockerRuntime {
    fn name(&self) -> &str { "docker" }
    fn has_shell_access(&self) -> bool { true }
    fn has_filesystem_access(&self) -> bool { true }  // Within container
    fn supports_long_running(&self) -> bool { true }

    fn build_shell_command(&self, command: &str) -> tokio::process::Command {
        let mut cmd = tokio::process::Command::new("docker");
        cmd.args(["exec", &self.container_id, "sh", "-c", command]);
        cmd
    }
}
```

### How Runtime Affects Tool Availability

The runtime's capability flags directly control which tools the agent has access to:

```
RuntimeAdapter
     │
     ├─ has_shell_access() ──► shell_exec tool enabled/disabled
     │
     ├─ has_filesystem_access() ──► file_read, file_write, file_list tools
     │
     ├─ supports_long_running() ──► background_process tools
     │
     └─ build_shell_command() ──► how shell_exec constructs commands
```

The tool factory (in `src/tools/mod.rs`) checks these flags before adding tools to the agent's toolset:

```rust
if runtime.has_shell_access() {
    tools.push(/* shell tools */);
}
if runtime.has_filesystem_access() {
    tools.push(/* file tools */);
}
```

---

## Supporting Modules

| File | Purpose |
|------|---------|
| `sandbox.rs` | Sandbox backends (Landlock, Bubblewrap) for constraining native runtime |
| `mod.rs` | Factory function and module exports |

### Sandbox Integration

The runtime layer also includes sandbox support for the native runtime, gated behind features:

- **`sandbox-landlock`** — Linux Landlock LSM. Restricts filesystem access to specific paths. Zero-overhead when not triggered.
- **`sandbox-bubblewrap`** — Wraps commands in `bwrap` for namespace isolation. Heavier but more comprehensive (network, PID, mount isolation).

Sandboxing is applied *within* the native runtime — `build_shell_command()` wraps the command in sandbox invocations when configured.

---

## Factory / Registration

**Location**: `src/runtime/mod.rs`

```rust
pub fn create_runtime(config: &RuntimeConfig) -> Arc<dyn RuntimeAdapter> {
    match config.adapter.as_str() {
        "native" => Arc::new(NativeRuntime::new(&config.workspace_dir)),
        "docker" => Arc::new(DockerRuntime::new(&config.docker)),
        "wasm"   => Arc::new(WasmRuntime::new()),
        _ => Arc::new(NativeRuntime::new(&config.workspace_dir)),  // Fallback
    }
}
```

---

## Configuration

```toml
[runtime]
adapter = "native"                      # "native", "docker", "wasm"
workspace_dir = "~/.zeroclaw/workspace"

[runtime.docker]
container_id = "zeroclaw-sandbox"
image = "zeroclaw/sandbox:latest"

[runtime.sandbox]
backend = "none"                        # "none", "landlock", "bubblewrap"
allowed_paths = ["/tmp", "~/.zeroclaw"]
```

---

## Extension Guide — Adding a New Runtime

### Step 1: Create the implementation file

Create `src/runtime/my_runtime.rs`:

```rust
use crate::runtime::traits::RuntimeAdapter;
use std::path::PathBuf;

pub struct MyRuntime {
    storage: PathBuf,
    // Connection details for your execution environment
}

impl MyRuntime {
    pub fn new(config: &MyRuntimeConfig) -> Self {
        Self {
            storage: config.storage_path.clone().into(),
        }
    }
}

impl RuntimeAdapter for MyRuntime {
    fn name(&self) -> &str {
        "my_runtime"
    }

    fn has_shell_access(&self) -> bool {
        true  // Set based on your runtime's capabilities
    }

    fn has_filesystem_access(&self) -> bool {
        true  // Set based on your runtime's capabilities
    }

    fn supports_long_running(&self) -> bool {
        false  // Set based on your runtime's capabilities
    }

    fn storage_path(&self) -> PathBuf {
        self.storage.clone()
    }

    fn memory_budget(&self) -> usize {
        512 * 1024 * 1024  // 512 MB, for example
    }

    fn build_shell_command(&self, command: &str) -> tokio::process::Command {
        // Construct the command for your execution environment
        // Examples:
        //   SSH: ssh user@host "command"
        //   K8s: kubectl exec pod -- sh -c "command"
        //   Firecracker: firecracker-exec "command"
        let mut cmd = tokio::process::Command::new("ssh");
        cmd.args([&self.host, command]);
        cmd
    }
}
```

### Step 2: Register in factory

In `src/runtime/mod.rs`:

```rust
mod my_runtime;
pub use my_runtime::MyRuntime;

// In create_runtime():
"my_runtime" => Arc::new(MyRuntime::new(&config.my_runtime)),
```

### Step 3: Add config section

```rust
#[derive(Debug, Clone, Serialize, Deserialize)]
pub struct MyRuntimeConfig {
    pub host: String,
    pub storage_path: String,
}
```

---

## Common Patterns & Gotchas

1. **Capability flags must be accurate**: If `has_shell_access()` returns `true` but `build_shell_command()` fails at runtime, the agent will encounter unexpected errors mid-turn. Be conservative — return `false` if capability is unreliable.

2. **`build_shell_command()` returns a `Command`, not output**: The caller (tool executor) handles spawning, timeout, output capture, and error handling. Your job is just constructing the right command invocation.

3. **Storage path must exist and be writable**: The agent writes memory databases, logs, and temp files to `storage_path()`. Ensure the directory exists and has proper permissions.

4. **WasmRuntime is a placeholder**: It's included to show the trait's extensibility. If you implement a real WASI runtime, you'll need to handle tool availability carefully since most tools assume filesystem/shell access.

5. **Sandbox is orthogonal to runtime**: Don't conflate runtime (where code runs) with sandboxing (what code is allowed to do). A native runtime can still be sandboxed via Landlock/Bubblewrap.

6. **Windows vs Unix in `build_shell_command()`**: If your runtime targets a specific OS, use conditional compilation (`#[cfg(...)]`) or document the requirement clearly.

---

*[← Observability](../observability/) | [Peripherals →](../peripherals/)*
