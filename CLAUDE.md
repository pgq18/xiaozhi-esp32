# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

Xiaozhi AI (小智AI) is an ESP32-based voice chatbot system that integrates with large language models (LLMs) like Qwen and DeepSeek. It supports 70+ hardware boards based on ESP32-C3, ESP32-S3, and ESP32-P4 chips. The system features offline wake word detection (ESP-SR), streaming ASR + LLM + TTS architecture, MCP (Model Context Protocol) for device control and cloud extensions, and multiple communication protocols (WebSocket or MQTT+UDP).

Current version: 2.1.0. Note that v2 partition tables are incompatible with v1 (v1 is maintained until Feb 2026).

## Build Commands

### Build Specific Board (Recommended)
Use the automated build script which handles target chip selection, configuration, and packaging:
```bash
python scripts/release.py <board-directory-name>
# Example: python scripts/release.py xmini-c3-v3

# List all available boards
python scripts/release.py --list-boards

# Build all boards
python scripts/release.py all
```

### Manual Build with ESP-IDF
```bash
# Set target chip (esp32, esp32c3, esp32s3, esp32c6, esp32p4)
idf.py set-target esp32c3

# Clean previous build (recommended when switching boards)
idf.py fullclean

# Configure board type
idf.py menuconfig
# Navigate to: Xiaozhi Assistant -> Board Type -> Select your board

# Build
idf.py build

# Flash to device
idf.py flash

# Monitor serial output
idf.py monitor
```

## Board Configuration System

### Board Directory Structure
Each board in `main/boards/<board-name>/` contains:
- `config.h` - Hardware pin mappings (I2S audio, I2C codec, display, buttons, LEDs)
- `config.json` - Build configuration (target chip, SDK config overrides)
- `<board>_board.cc` - Board-specific initialization code
- `README.md` - Board documentation

### Board Inheritance Architecture
```
Board (base class)
├── WifiBoard - Wi-Fi connectivity with automatic provisioning fallback
│   └── <Most ESP32 boards>
└── Ml307Board - 4G cellular via ML307 module
    └── DualNetworkBoard - Combines Wi-Fi + 4G

Common utilities in main/boards/common/:
- adc_battery_monitor.cc - Battery voltage monitoring via ADC
- axp2101.cc - Power management chip driver
- blufi.cpp - Bluetooth provisioning for Wi-Fi credentials
- button.cc - Button handling with debounce
- wifi_board.cc / ml307_board.cc - Network base classes
```

Each board class must implement:
- `GetAudioCodec()` - Return audio codec instance (ES8311, ES8374, ES8388, etc.)
- `GetDisplay()` - Return display instance (OLED, LCD, LVGL, or NoDisplay)
- `StartNetwork()` - Initiate network connection async
- `SetPowerSaveLevel()` - Adjust CPU frequency based on state

**Critical**: Never override an existing board's config directly. Always create a new board directory or use `config.json` `builds[].name` to ensure unique firmware for OTA updates.

## State Machine Architecture

The application uses a strict state machine (`DeviceStateMachine` in `main/device_state_machine.cc`) with validated transitions:

**States** (`main/device_state.h`):
- `kDeviceStateStarting` - Initialization phase
- `kDeviceStateActivating` - Server activation/OTA check
- `kDeviceStateIdle` - Ready for wake word
- `kDeviceStateConnecting` - Opening audio channel to server
- `kDeviceStateListening` - Capturing user voice input
- `kDeviceStateSpeaking` - Playing TTS response
- `kDeviceStateWifiConfiguring` - Wi-Fi provisioning mode
- `kDeviceStateUpgrading` - Firmware OTA upgrade

**State Change Notification**:
The state machine uses observer pattern. To react to state changes, add a listener in `Application::Initialize()`:
```cpp
state_machine_.AddStateChangeListener([this](DeviceState old_state, DeviceState new_state) {
    // Store previous state if needed for transition logic
    previous_state_ = old_state;
    xEventGroupSetBits(event_group_, MAIN_EVENT_STATE_CHANGED);
});
```

Then handle in `Application::HandleStateChangedEvent()` via the event loop. When adding state-based behavior (like playing sounds), always check `previous_state_` to ensure the action only applies to specific transitions.

## Audio System

### Audio Pipeline
```
Microphone → [AudioProcessor] → [Opus Encoder] → [SendQueue] → Network
Network → [ReceiveQueue] → [Opus Decoder] → [PlaybackQueue] → Speaker
```

### Audio Service (main/audio/audio_service.h)
Key methods:
- `Start()` / `Stop()` - Start/stop audio processing threads
- `EnableVoiceProcessing(bool)` - Enable audio capture and encoding
- `EnableWakeWordDetection(bool)` - Enable ESP-SR wake word detection
- `PlaySound(std::string_view ogg)` - Play OGG sound effect immediately
- `IsVoiceDetected()` - Check VAD (Voice Activity Detection) status

### Wake Word Detection
The system uses ESP-SR for offline wake word detection. When detected, `AudioService` invokes the `on_wake_word_detected` callback, setting `MAIN_EVENT_WAKE_WORD_DETECTED`. `Application::HandleWakeWordDetectedEvent()` then transitions to `kDeviceStateListening` and optionally plays `OGG_POPUP` via the `play_popup_on_listening_` flag pattern.

## Application Event Loop

The `Application` class (main/application.cc) runs the main event loop in `Run()`, processing bits from `xEventGroupWaitBits()`:

Key events:
- `MAIN_EVENT_STATE_CHANGED` - Device state transition (handle in `HandleStateChangedEvent()`)
- `MAIN_EVENT_WAKE_WORD_DETECTED` - Wake word heard
- `MAIN_EVENT_SEND_AUDIO` - Audio packet ready to send
- `MAIN_EVENT_NETWORK_CONNECTED` / `MAIN_EVENT_NETWORK_DISCONNECTED`
- `MAIN_EVENT_START_LISTENING` / `MAIN_EVENT_STOP_LISTENING` - Manual control (e.g., button press)

Use `Schedule(std::function<void()>&&)` to defer code execution to the main task thread-safely.

## Protocol Communication

### WebSocket Protocol (main/protocols/websocket_protocol.cc)
- Full-duplex JSON messages + binary audio streaming
- `OnIncomingJson` handles message types: "tts", "stt", "llm", "mcp", "system", "alert", "custom"
- `OnIncomingAudio` delivers Opus-encoded audio packets for playback

### MQTT+UDP Protocol (main/protocols/mqtt_protocol.cc)
- MQTT for JSON control messages, UDP for audio streaming
- Useful for networks requiring MQTT infrastructure

### Protocol Activation
The protocol is initialized in `Application::InitializeProtocol()` based on OTA server configuration (`ota_->HasMqttConfig()` or `ota_->HasWebsocketConfig()`).

## MCP (Model Context Protocol)

### Device-Side MCP (main/mcp_server.cc)
Registers tools that the LLM can invoke to control device hardware:
- Common tools: Volume control, reboot, shutdown, WiFi config, etc.
- User-only tools: Motor control, GPIO, custom board-specific features (add via `McpServer::AddUserOnlyTools()` in board init)

### Cloud MCP Extensions
The server can extend LLM capabilities with cloud MCP tools for smart home control, PC operations, web search, etc. Protocol communication in `docs/mcp-protocol.md`.

## Assets and Localization

### Audio Assets
Located in `main/assets/`:
- `common/` - Universal sounds (exclamation.ogg, popup.ogg, success.ogg, low_battery.ogg, vibration.ogg)
- `locales/<lang>/` - Language-specific assets (0-9.ogg, activation.ogg, welcome.ogg, etc.)

Assets are embedded into the firmware as binary data via ESP-IDF's binary embedding (`_binary_<filename>_start`/`_end` symbols). See `main/assets/lang_config.h` for generated constants (e.g., `Lang::Sounds::OGG_POPUP`).

### Customization
Users can customize wake words, fonts, emotions, and chat backgrounds using the [Xiaozhi Assets Generator](https://github.com/78/xiaozhi-assets-generator). Custom assets are loaded from the assets partition at runtime.

## Hardware Abstraction Layers

### Display (main/display/)
- `OledDisplay` - SSD1306 OLED via SPI
- `LcdDisplay` - ST7789/ILI9341 LCD via SPI
- `LvglDisplay` - LVGL GUI framework for touchscreens
- `NoDisplay` - For boards without screen

Methods: `SetStatus()`, `SetEmotion()`, `SetChatMessage()`, `ShowNotification()`, `UpdateStatusBar()`

### Audio Codec (main/audio_codec/)
Supported chips: ES8311, ES8374, ES8388, ES8389, BoxAudioCodec, DummyAudioCodec. Each codec handles I2S configuration, I2C control, and power amplifier (PA) enable/disable.

### LED (main/led/)
- `LedSingle` - Single GPIO LED
- `LedRGB` - RGB LED (common anode/cathode)
- `DotStarLED` - APA102/SK9822 SPI LED
- `NoLed` - For boards without LED

## Development Guidelines

### Code Style
- Google C++ style (enforced in project)
- Existing code uses C++11/14 features
- FreeRTOS APIs for multitasking (tasks, queues, timers, event groups)

### Adding a New Feature
1. Identify if feature belongs in Application (app-level logic), board (hardware-specific), or a new module
2. For state-driven behavior, modify `HandleStateChangedEvent()` with proper `previous_state_` checks
3. Use existing patterns like `play_popup_on_listening_` flag for delayed actions after decoder reset
4. Test with multiple board types if change is in shared code

### Modifying Board Hardware
Never edit pin mappings in existing board directories. Create a new board directory to avoid OTA update conflicts. See `docs/custom-board.md` for detailed guide.

### Debugging
- Use `ESP_LOGI/E/W` macros from `esp_log.h`
- `SystemInfo::PrintHeapStats()` available (called periodically in main loop)
- Monitor via `idf.py monitor` or serial console

## Important Paths

- `main/application.cc` / `main/application.h` - Main application logic and event loop
- `main/device_state_machine.cc` - State transition validation
- `main/boards/common/` - Shared board implementation code
- `main/audio/` - Audio service, codec drivers, wake word, VAD
- `main/protocols/` - WebSocket and MQTT protocol implementations
- `main/mcp_server.cc` - Device-side MCP server implementation
- `docs/` - Protocol documentation, custom board guide
- `scripts/release.py` - Automated build script
