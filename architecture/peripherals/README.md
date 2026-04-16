# Peripherals Trait — Architecture Deep-Dive

> **Source**: `src/peripherals/traits.rs`
> **Factory**: `src/peripherals/mod.rs`
> **Parent doc**: [Architecture Overview](../overview.md)

---

## Purpose

The `Peripheral` trait abstracts hardware boards and physical I/O devices. It enables the agent to interact with microcontrollers (STM32, Arduino), single-board computers (Raspberry Pi), and other hardware — connecting, disconnecting, checking health, and exposing hardware capabilities as agent tools.

Each peripheral, once connected, contributes additional tools to the agent's toolset (e.g., `flash_firmware`, `gpio_write`, `serial_send`). This makes hardware interaction a first-class citizen of the agent loop, not a bolted-on afterthought.

**When to implement**: You want the agent to control a new hardware board or device (e.g., ESP32, BeagleBone, custom FPGA, robotic arm controller).

---

## Trait Definition

```rust
#[async_trait]
pub trait Peripheral: Send + Sync {
    // ── Identification ────────────────────────────────────────

    /// Peripheral name (e.g., "nucleo-f401re", "arduino-uno", "rpi-gpio").
    fn name(&self) -> &str;

    /// Board/device type identifier.
    fn board_type(&self) -> &str;

    // ── Lifecycle ─────────────────────────────────────────────

    /// Connect to the peripheral (open serial port, establish link, etc.).
    async fn connect(&mut self) -> anyhow::Result<()>;

    /// Disconnect gracefully.
    async fn disconnect(&mut self) -> anyhow::Result<()>;

    /// Check if the peripheral is responding.
    async fn health_check(&self) -> bool;

    // ── Tool integration ──────────────────────────────────────

    /// Return the tools this peripheral provides to the agent.
    /// Called after connect() succeeds.
    fn tools(&self) -> Vec<Box<dyn crate::tools::traits::Tool>>;
}
```

**Key design choice**: `connect()` takes `&mut self` because connection may modify internal state (serial port handle, session token, etc.). After construction, the factory calls `connect()` before the peripheral is shared — once connected, it's wrapped in `Arc` for shared access.

### Method Summary

| Method | Required? | Notes |
|--------|-----------|-------|
| `name()` | **Yes** | Human-readable peripheral name |
| `board_type()` | **Yes** | Board identifier (e.g., "stm32f401", "arduino-uno") |
| `connect()` | **Yes** | Initialize connection to hardware |
| `disconnect()` | **Yes** | Clean shutdown |
| `health_check()` | **Yes** | Liveness probe |
| `tools()` | **Yes** | Expose hardware capabilities as agent tools |

All methods are required — there are no defaults.

---

## Existing Implementations

| Implementation | File(s) | Key Characteristics |
|----------------|---------|-------------------|
| Nucleo F401RE | `nucleo_flash.rs` | STM32 Nucleo board. Firmware flashing via OpenOCD/STM32CubeProgrammer. Serial communication. |
| Arduino | `arduino_flash.rs`, `arduino_upload.rs` | Arduino boards. Firmware upload via `avrdude`. Serial communication. |
| RPi GPIO | `rpi.rs` | Raspberry Pi GPIO. Direct pin read/write. **Feature-gated**: `peripherals-rpi`. |

### Supporting Modules

| File | Purpose |
|------|---------|
| `serial.rs` | Shared serial port utilities (open, read, write, baud rate config) |
| `capabilities_tool.rs` | Tool that lists connected peripheral capabilities |
| `uno_q_bridge.rs` | Arduino Uno communication bridge |
| `uno_q_setup.rs` | Arduino Uno initial setup/handshake |
| `mod.rs` | Factory function, module exports, peripheral discovery |

---

## How Peripherals Contribute Tools

The peripheral → tool integration flow:

```
 Config: peripherals = ["nucleo-f401re"]
                │
                ▼
     create_peripherals(config)
                │
                ▼
     NucleoF401RE::new(config)
                │
                ▼
     peripheral.connect().await
                │
                ▼
     peripheral.tools() ──► vec![
                               FlashFirmwareTool { ... },
                               SerialSendTool { ... },
                             ]
                │
                ▼
     Agent toolset (merged with core tools)
```

During the tool factory assembly (`src/tools/mod.rs`), after core tools are added, the factory iterates over connected peripherals and appends their tools:

```rust
for peripheral in &connected_peripherals {
    let hw_tools = peripheral.tools();
    tools.extend(hw_tools);
}
```

This means the agent's tool descriptions sent to the LLM dynamically include hardware capabilities when peripherals are connected.

---

## Factory / Registration

**Location**: `src/peripherals/mod.rs`

```rust
pub async fn create_peripherals(
    config: &[PeripheralConfig],
) -> anyhow::Result<Vec<Arc<dyn Peripheral>>> {
    let mut peripherals = Vec::new();

    for cfg in config {
        let mut peripheral: Box<dyn Peripheral> = match cfg.board_type.as_str() {
            "nucleo-f401re" => Box::new(NucleoF401RE::new(cfg)?),
            "arduino-uno" | "arduino" => Box::new(ArduinoPeripheral::new(cfg)?),
            #[cfg(feature = "peripherals-rpi")]
            "rpi-gpio" => Box::new(RpiGpio::new(cfg)?),
            _ => anyhow::bail!("Unknown peripheral: {}", cfg.board_type),
        };

        peripheral.connect().await?;

        if !peripheral.health_check().await {
            tracing::warn!("Peripheral {} failed health check", peripheral.name());
            continue;  // Skip unhealthy peripherals
        }

        peripherals.push(Arc::from(peripheral));
    }

    Ok(peripherals)
}
```

Note: Unhealthy peripherals are skipped rather than causing startup failure — the agent can still operate without hardware.

---

## Configuration

```toml
[[peripherals]]
board_type = "nucleo-f401re"
name = "my-nucleo"
port = "/dev/ttyACM0"           # Serial port (Linux)
# port = "COM3"                 # Serial port (Windows)
baud_rate = 115200

[[peripherals]]
board_type = "arduino-uno"
name = "my-arduino"
port = "/dev/ttyUSB0"
baud_rate = 9600

[[peripherals]]
board_type = "rpi-gpio"         # Requires feature: peripherals-rpi
name = "my-rpi"
```

Note: `[[peripherals]]` is a TOML array of tables — multiple peripherals can be configured.

---

## Extension Guide — Adding a New Peripheral

### Step 1: Create the implementation file

Create `src/peripherals/my_board.rs`:

```rust
use async_trait::async_trait;
use crate::peripherals::traits::Peripheral;
use crate::tools::traits::Tool;

pub struct MyBoard {
    name: String,
    port: String,
    connected: bool,
    // Your hardware connection handle
}

impl MyBoard {
    pub fn new(config: &PeripheralConfig) -> anyhow::Result<Self> {
        Ok(Self {
            name: config.name.clone(),
            port: config.port.clone().unwrap_or_default(),
            connected: false,
        })
    }
}

#[async_trait]
impl Peripheral for MyBoard {
    fn name(&self) -> &str {
        &self.name
    }

    fn board_type(&self) -> &str {
        "my-board"
    }

    async fn connect(&mut self) -> anyhow::Result<()> {
        // Open serial port, establish connection, handshake
        // Example using shared serial utilities:
        // self.serial = serial::open(&self.port, self.baud_rate)?;
        self.connected = true;
        Ok(())
    }

    async fn disconnect(&mut self) -> anyhow::Result<()> {
        // Close connection gracefully
        self.connected = false;
        Ok(())
    }

    async fn health_check(&self) -> bool {
        // Ping the device, check serial is open, etc.
        self.connected
    }

    fn tools(&self) -> Vec<Box<dyn Tool>> {
        vec![
            // Each tool should be a struct implementing the Tool trait
            Box::new(MyBoardFlashTool::new(self.port.clone())),
            Box::new(MyBoardReadSensorTool::new(self.port.clone())),
        ]
    }
}
```

### Step 2: Implement hardware-specific tools

Each tool returned by `tools()` must implement the `Tool` trait (see [Tools documentation](../tools/)):

```rust
pub struct MyBoardReadSensorTool {
    port: String,
}

#[async_trait]
impl Tool for MyBoardReadSensorTool {
    fn name(&self) -> &str { "my_board_read_sensor" }

    fn description(&self) -> &str {
        "Read a sensor value from MyBoard. Returns the current reading as a float."
    }

    fn parameters_schema(&self) -> serde_json::Value {
        serde_json::json!({
            "type": "object",
            "properties": {
                "sensor_id": {
                    "type": "string",
                    "description": "The sensor to read (e.g., 'temperature', 'humidity')"
                }
            },
            "required": ["sensor_id"]
        })
    }

    async fn call(&self, args: serde_json::Value) -> anyhow::Result<ToolResult> {
        let sensor_id = args["sensor_id"].as_str().unwrap_or("temperature");
        // Read from hardware via serial
        let value = read_sensor(&self.port, sensor_id).await?;
        Ok(ToolResult::text(format!("Sensor {sensor_id}: {value}")))
    }
}
```

### Step 3: Register in factory

In `src/peripherals/mod.rs`:

```rust
mod my_board;
pub use my_board::MyBoard;

// In create_peripherals():
"my-board" => Box::new(MyBoard::new(cfg)?),
```

### Step 4: Add config support

The `[[peripherals]]` array already supports arbitrary board types — just document your board's expected config fields.

### Step 5: Feature-gate if needed

If your peripheral requires platform-specific dependencies:

```toml
# Cargo.toml
[features]
peripherals-myboard = ["dep:my-hardware-lib"]
```

---

## Common Patterns & Gotchas

1. **Serial port access is platform-specific**: On Linux, serial ports are `/dev/ttyACM*` or `/dev/ttyUSB*`. On Windows, they're `COM*`. On macOS, `/dev/cu.*`. Handle all platforms or document requirements.

2. **`connect()` takes `&mut self`**: This is the only mutable method. After `connect()`, the peripheral is wrapped in `Arc` and shared immutably. Design your state management accordingly (e.g., use `Mutex` for internal serial port handles if tools need write access).

3. **Tools capture their own connection handles**: Since `tools()` returns `Vec<Box<dyn Tool>>` (owned tools), each tool must capture its own reference to the hardware connection (port path, shared handle via `Arc<Mutex<...>>`, etc.).

4. **Unhealthy peripherals are skipped, not fatal**: The factory logs a warning and continues. The agent remains functional without hardware. Don't depend on peripheral availability for core agent operation.

5. **Hot-plug is not supported**: Peripherals are discovered and connected at startup. If a device is plugged in later, the agent must be restarted (or a future enhancement could add dynamic discovery).

6. **Use `serial.rs` for common serial operations**: Don't reimplement serial port handling. The shared `serial.rs` module provides open, read, write, and configuration utilities.

7. **Test without hardware**: Create a mock implementation (`MockBoard`) that simulates hardware responses. The `NoneMemory` pattern (see [Memory](../memory/)) applies here too.

---

## Related: `crates/robot-kit/` and `crates/aardvark-sys/`

The `crates/` directory contains two hardware-related subcrates:

- **`robot-kit/`** — Higher-level robotics abstractions (motion planning, sensor fusion) that build on the peripheral trait.
- **`aardvark-sys/`** — FFI bindings for the Aardvark I2C/SPI adapter, enabling communication with I2C/SPI devices.

These are external crates, not direct trait implementations, but complement the peripheral system.

---

*[← Runtime](../runtime/) | [Back to Index](../)*
