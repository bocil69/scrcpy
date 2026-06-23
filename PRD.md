# Product Requirements Document (PRD)
## Multi-Device Scrcpy Dashboard

**Project Name:** Multi-Device Scrcpy Controller  
**Location:** `D:\PROJECT\multiscrcpy`  
**Build Machine:** Separate AI machine  
**Status:** In Development  

---

## 1. Executive Summary

Build a desktop application (Rust + Tauri + React) that allows simultaneous control of multiple Android devices through a single unified modern UI. Users can:
- View multiple device screens in a responsive grid layout
- Control devices individually or synchronize input to multiple devices via a "master device" mode
- Control device power state and screen wake behavior
- Manage all devices from a single executable (no Docker, no web browser required)

---

## 2. Problem Statement

Current Issues:
- Running `scrcpy.exe` per device requires multiple terminal windows
- No unified interface for multi-device management
- Keyboard/mouse cursor visible on every connected device
- No programmatic control over screen power or wake-lock state
- Need to manually manage each device connection

---

## 3. Goals & Objectives

### Primary Goals
1. **Single Unified Dashboard** — All devices in one responsive grid UI, auto-arranged left-to-right, top-to-bottom
2. **Master-Slave Input Sync** — Master device broadcasts keyboard/mouse input to selected slave devices
3. **Device Management** — Screen on/off, always-awake toggle per device
4. **Native Desktop App** — Standalone executable (`.exe`, `.app`, `.AppImage`), no web browser dependency
5. **Modern UI/UX** — Contemporary fonts, dark theme, responsive layout for various monitor sizes

### Secondary Goals
1. **Performance** — 30+ FPS streaming per device, minimal latency
2. **Scalability** — Support 2-8 simultaneous devices without degradation
3. **Reliability** — Auto-reconnect on device disconnect, graceful error handling
4. **Cross-platform** — Windows, Linux, macOS support (Rust + Tauri enables this)

---

## 4. Scope

### In Scope
- Multi-device listing via ADB
- Scrcpy library modification to support programmatic API (not CLI-only)
- Frame capture and streaming from each device to UI
- Keyboard input routing (master → selected slaves)
- Mouse input routing (master → selected slaves, with coordinate scaling)
- Touch input simulation (optional, stretch goal)
- Screen power control (on/off) per device
- Stay-awake control per device
- Responsive grid layout (auto-adjust columns per screen size)
- Device name/serial display in tile header
- Master device selection dropdown
- "Follow Master" checkbox per device
- Settings panel (accessibility, debug logs)

### Out of Scope
- Multi-platform build CI/CD (manual builds per platform)
- Audio streaming (video-only initially)
- Recording functionality (future release)
- Device file transfer (future release)
- Gamepad simulation (use scrcpy's native gamepad support if needed)
- VPN/tunnel configuration (assume local network or manual ADB setup)
- Web-based interface
- Docker containerization

---

## 5. User Stories

### US-1: View Multiple Devices
**As a** test automation engineer  
**I want to** see up to 8 Android devices on one screen  
**So that** I can monitor test execution across multiple devices simultaneously

**Acceptance Criteria:**
- [ ] App detects and lists all ADB-connected devices on startup
- [ ] Each device displayed in a responsive grid tile with device serial/name
- [ ] Grid auto-adjusts columns based on monitor resolution (1x1, 2x2, 3x3, etc.)
- [ ] Tiles are 250px minimum width, scale responsively
- [ ] Frame updates at 30fps per device (16.67ms interval)

---

### US-2: Master Device Control
**As a** test automation engineer  
**I want to** select one device as "master" and sync inputs to other devices  
**So that** I can perform the same actions on multiple devices simultaneously

**Acceptance Criteria:**
- [ ] Dropdown in control panel to select master device
- [ ] Selected master device shows green border highlight
- [ ] Each other device has "Follow Master" checkbox
- [ ] When master device keyboard input is detected, it broadcasts to all checked followers
- [ ] When master device mouse input is detected, it broadcasts to all checked followers
- [ ] Input coordinate scaling handles different device resolutions

---

### US-3: Individual Device Control
**As a** test automation engineer  
**I want to** control each device independently via keyboard and mouse  
**So that** I can interact with individual devices without affecting others

**Acceptance Criteria:**
- [ ] Clicking on a device tile focuses input to that device
- [ ] Keyboard input goes to focused device only (if not in master mode)
- [ ] Mouse movement tracked per device tile with coordinate transformation
- [ ] Unfocused tiles do not receive input events

---

### US-4: Device Power Management
**As a** test automation engineer  
**I want to** turn device screens on/off and keep them awake  
**So that** I can control device behavior during test execution

**Acceptance Criteria:**
- [ ] Each device tile has "Screen On/Off" button (📱/📵 icon)
- [ ] Each device tile has "Always Awake" toggle (⚡ icon)
- [ ] Clicking screen toggle sends `scrcpy_device_set_screen_power()` command
- [ ] Clicking awake toggle sends `scrcpy_device_set_stay_awake()` command
- [ ] UI reflects current state after toggle

---

### US-5: Responsive Layout
**As a** user with a 1440p or 4K monitor  
**I want to** layout devices in a grid that adapts to my screen size  
**So that** I can see multiple devices clearly without wasted space

**Acceptance Criteria:**
- [ ] 1920px width → 2 columns
- [ ] 2560px width → 3 columns
- [ ] 3840px width → 4 columns
- [ ] Aspect ratio of tiles maintained (16:9)
- [ ] Gap between tiles consistent (16px)
- [ ] Grid reflows on window resize without restarting

---

### US-6: Keyboard/Mouse Cursor Management
**As a** user  
**I want to** not see the keyboard/mouse cursor on device screens  
**So that** I don't get distracted and the experience feels native

**Acceptance Criteria:**
- [ ] Scrcpy library compiled with option to hide input indicators on device
- [ ] When connected to scrcpy, device doesn't display remote input cursor
- [ ] Keyboard input is injected without visible keyboard UI

---

### US-7: Device Reconnection
**As a** user  
**I want to** have the app automatically reconnect if a device disconnects  
**So that** I don't need to manually re-connect

**Acceptance Criteria:**
- [ ] App detects device disconnection via ADB
- [ ] Auto-retry connection with exponential backoff (1s, 2s, 4s, 8s max)
- [ ] UI shows "Connecting..." state during retry
- [ ] Reconnect happens up to 10 times before giving up
- [ ] User can manually reconnect via button

---

## 6. Technical Architecture

### System Components

```
┌───────────────────────────────────────────┐
│  Frontend (React + TypeScript)            │
│  - DeviceGrid.tsx (responsive layout)     │
│  - DeviceTile.tsx (single device display) │
│  - MasterControl.tsx (master selector)    │
│  - SettingsPanel.tsx (power/awake toggles)│
└───────────────────────────────────────────┘
           ↓ Tauri IPC
┌───────────────────────────────────────────┐
│  Tauri Backend (Rust)                     │
│  - device_manager.rs (ADB listing)        │
│  - scrcpy_bridge.rs (FFI to C library)    │
│  - input_handler.rs (master sync logic)   │
│  - frame_streamer.rs (frame capture loop) │
│  - Tauri commands (IPC endpoints)         │
└───────────────────────────────────────────┘
           ↓ C FFI (DLL/SO/DYLIB)
┌───────────────────────────────────────────┐
│  scrcpy-core library (C)                  │
│  - Modified scrcpy/app/src (library API)  │
│  - Video decoder + frame buffer           │
│  - Control message injection              │
│  - Power/wake-lock control                │
│  - ADB tunnel management                  │
└───────────────────────────────────────────┘
           ↓ ADB + TCP
      Android Devices
```

### File Structure

```
D:\PROJECT\multiscrcpy\
├── README.md                    # Project overview
├── SETUP.md                     # Build & setup instructions
├── scrcpy-core/                 # Cloned scrcpy, modified for library
│   ├── app/
│   │   ├── src/
│   │   │   ├── main.c           # Modified: new entry for library init
│   │   │   ├── scrcpy_lib.c     # NEW: library API implementation
│   │   │   ├── scrcpy_lib.h     # NEW: library API header
│   │   │   ├── scrcpy.c         # Modified: refactored for reuse
│   │   │   ├── scrcpy.h         # Modified: expose library functions
│   │   │   ├── ... (rest unchanged)
│   │   └── meson.build          # Modified: build as library + CLI
│   ├── server/                  # Unchanged (Android server)
│   ├── meson.build              # Modified: optional library build
│   └── ... (rest unchanged)
├── src-tauri/                   # Rust backend
│   ├── main.rs                  # Tauri entry, command handlers
│   ├── device_manager.rs        # Device detection + lifecycle
│   ├── scrcpy_bridge.rs         # FFI bindings to scrcpy-core
│   ├── input_handler.rs         # Master/slave input sync
│   ├── frame_streamer.rs        # Frame capture loop
│   └── Cargo.toml
├── src/                         # React frontend
│   ├── components/
│   │   ├── DeviceGrid.tsx       # Main grid layout
│   │   ├── DeviceTile.tsx       # Single device tile
│   │   ├── MasterControl.tsx    # Master selector
│   │   ├── SettingsPanel.tsx    # Global settings
│   │   └── styles/
│   │       ├── DeviceGrid.css
│   │       ├── DeviceTile.css
│   │       └── global.css
│   ├── App.tsx
│   ├── main.tsx
│   └── package.json
├── build-scripts/               # Build helpers
│   ├── build-scrcpy-lib.sh      # Build scrcpy-core as library
│   ├── build-tauri.sh           # Build Tauri desktop app
│   └── deploy.sh                # Package final executable
├── Cargo.toml                   # Main Rust workspace
├── package.json                 # Main Node workspace
├── tauri.conf.json              # Tauri configuration
└── .gitignore
```

---

## 7. API Specification

### Tauri Commands (Backend → Frontend)

#### list_devices()
**Purpose:** Get list of connected ADB devices  
**Input:** None  
**Output:** `Vec<String>` (device serials)  
**Error:** ADB command failure  

```rust
#[tauri::command]
fn list_devices() -> Vec<String> { ... }
```

---

#### connect_device(serial: String)
**Purpose:** Initialize and start scrcpy connection to a device  
**Input:** Device serial (e.g., "emulator-5554", "192.168.1.100:5555")  
**Output:** `String` (success message)  
**Error:** Connection failed, invalid serial  

```rust
#[tauri::command]
fn connect_device(serial: String) -> Result<String, String> { ... }
```

---

#### get_device_frame(serial: String)
**Purpose:** Get current video frame from device as RGBA bytes  
**Input:** Device serial  
**Output:** `Vec<u8>` (RGBA frame data, width×height×4 bytes)  
**Error:** Device not found, frame capture failed  

```rust
#[tauri::command]
async fn get_device_frame(serial: String) -> Result<Vec<u8>, String> { ... }
```

Frame format:
- Width: 1920px (device resolution)
- Height: 1080px (device resolution)
- Format: RGBA (4 bytes per pixel)
- Size: 1920 × 1080 × 4 = 8,294,400 bytes

---

#### set_master_device(serial: String)
**Purpose:** Designate a device as the master for input sync  
**Input:** Device serial  
**Output:** Success  
**Error:** Device not found  

```rust
#[tauri::command]
fn set_master_device(serial: String) -> Result<(), String> { ... }
```

---

#### set_slave_follow(serial: String, follow: bool)
**Purpose:** Enable/disable input sync from master to a specific device  
**Input:** Device serial, follow flag  
**Output:** Success  
**Error:** Device not found  

```rust
#[tauri::command]
fn set_slave_follow(serial: String, follow: bool) -> Result<(), String> { ... }
```

---

#### send_mouse_input(serial: String, x: i32, y: i32)
**Purpose:** Send mouse movement to specific device  
**Input:** Device serial, X/Y coordinates (0-1920, 0-1080)  
**Output:** Success  
**Error:** Device not found, send failed  

```rust
#[tauri::command]
fn send_mouse_input(serial: String, x: i32, y: i32) -> Result<(), String> { ... }
```

---

#### send_key_input(serial: String, keycode: u32, pressed: bool)
**Purpose:** Send keyboard key to specific device  
**Input:** Device serial, keycode (e.g., 65 for 'A'), pressed flag  
**Output:** Success  
**Error:** Device not found, send failed  

```rust
#[tauri::command]
fn send_key_input(serial: String, keycode: u32, pressed: bool) -> Result<(), String> { ... }
```

---

#### broadcast_mouse_input(x: i32, y: i32)
**Purpose:** Send mouse movement to all slave devices following master  
**Input:** X/Y coordinates  
**Output:** Success  
**Error:** None (silent fail if no followers)  

```rust
#[tauri::command]
fn broadcast_mouse_input(x: i32, y: i32) -> Result<(), String> { ... }
```

---

#### broadcast_key_input(keycode: u32, pressed: bool)
**Purpose:** Send keyboard key to all slave devices following master  
**Input:** Keycode, pressed flag  
**Output:** Success  
**Error:** None (silent fail if no followers)  

```rust
#[tauri::command]
fn broadcast_key_input(keycode: u32, pressed: bool) -> Result<(), String> { ... }
```

---

#### set_screen_power(serial: String, on: bool)
**Purpose:** Turn device screen on/off  
**Input:** Device serial, on flag  
**Output:** Success  
**Error:** Device not found, command failed  

```rust
#[tauri::command]
fn set_screen_power(serial: String, on: bool) -> Result<(), String> { ... }
```

---

#### set_always_awake(serial: String, awake: bool)
**Purpose:** Enable/disable wake-lock on device  
**Input:** Device serial, awake flag  
**Output:** Success  
**Error:** Device not found, command failed  

```rust
#[tauri::command]
fn set_always_awake(serial: String, awake: bool) -> Result<(), String> { ... }
```

---

## 8. C Library API (scrcpy_lib.h)

Header file defining the programmatic API for scrcpy-core:

```c
// scrcpy_lib.h
#ifndef SCRCPY_LIB_H
#define SCRCPY_LIB_H

#include <stdint.h>
#include <stdbool.h>

typedef struct scrcpy_device scrcpy_device_t;

typedef struct {
    char device_serial[256];
    uint16_t max_width;
    uint16_t max_height;
    uint32_t bit_rate;
    uint32_t scid;
} scrcpy_config_t;

typedef struct {
    uint8_t *data;      // RGBA frame data
    uint32_t width;
    uint32_t height;
    uint32_t stride;    // bytes per row
} video_frame_t;

// Device lifecycle
scrcpy_device_t* scrcpy_device_init(const scrcpy_config_t *cfg);
bool scrcpy_device_start(scrcpy_device_t *dev);
void scrcpy_device_stop(scrcpy_device_t *dev);
void scrcpy_device_destroy(scrcpy_device_t *dev);

// Video frame capture
bool scrcpy_device_get_frame(scrcpy_device_t *dev, video_frame_t *out);
void scrcpy_device_frame_free(video_frame_t *frame);

// Input control
bool scrcpy_device_send_key(scrcpy_device_t *dev, uint32_t keycode, bool pressed);
bool scrcpy_device_send_mouse(scrcpy_device_t *dev, int x, int y);
bool scrcpy_device_send_touch(scrcpy_device_t *dev, int x, int y, bool pressed);

// Device control
bool scrcpy_device_set_screen_power(scrcpy_device_t *dev, bool on);
bool scrcpy_device_set_stay_awake(scrcpy_device_t *dev, bool awake);

#endif
```

---

## 9. Build & Deployment

### Prerequisites
- **Windows:** MSVC 2019+, Meson, Ninja, Tauri CLI
- **Linux:** GCC 9+, Meson, Ninja, Tauri CLI
- **macOS:** Xcode 12+, Meson, Ninja, Tauri CLI
- **Common:** Node 16+, Rust 1.70+, Python 3.8+ (for Meson)

### Build Steps (See SETUP.md)

1. **Clone & Modify scrcpy** → `D:\PROJECT\multiscrcpy\scrcpy-core\`
2. **Compile scrcpy-core library** → outputs `.dll`, `.so`, or `.dylib`
3. **Build Tauri app** → outputs standalone `.exe` / `.app` / `.AppImage`

### Final Deliverables

| Platform | Output | Size | Notes |
|----------|--------|------|-------|
| Windows  | `scrcpy-dashboard.exe` | ~80MB | Standalone, no runtime |
| Linux    | `scrcpy-dashboard.AppImage` | ~90MB | Portable, no dependencies |
| macOS    | `scrcpy-dashboard.app` | ~100MB | Code-signed |

---

## 10. Testing Strategy

### Unit Tests
- Device manager (list, connect, disconnect)
- Input handler (master sync logic)
- Frame conversion (RGBA formatting)

### Integration Tests
- Multi-device connection (2-4 devices)
- Master-slave input broadcast
- Frame streaming loop (frame rate validation)
- Reconnection on disconnect

### Manual Tests
- Grid responsiveness (1920px, 2560px, 3840px widths)
- Keyboard input latency (should be <50ms)
- Mouse coordinate mapping
- Screen on/off control
- Always-awake toggle

### Performance Benchmarks
- Frame rate per device (target: 30fps)
- Memory usage per device (target: <100MB)
- CPU usage for 4 devices (target: <40%)
- Startup time (target: <5s to first frame)

---

## 11. Success Criteria

### MVP (Minimum Viable Product)
- [x] Single window with responsive grid
- [x] Detect & connect 2-4 devices
- [x] Stream video at 30fps per device
- [x] Master device keyboard/mouse sync
- [x] Screen on/off toggle
- [x] Standalone Windows `.exe` output

### Phase 2 (Post-MVP)
- [ ] Always-awake toggle (wake-lock)
- [ ] Auto-reconnect logic
- [ ] Settings panel (debug, logging)
- [ ] Linux & macOS builds
- [ ] Touch input simulation

### Phase 3 (Future)
- [ ] Audio streaming
- [ ] Device screen recording
- [ ] File transfer
- [ ] Custom device layouts (manual positioning)
- [ ] Keyboard mapping profiles

---

## 12. Timeline

| Phase | Deliverable | Duration | Target |
|-------|-------------|----------|--------|
| 1     | scrcpy library (C API) | 1-2 weeks | Clone, modify, build as library |
| 2     | Rust FFI bindings | 1 week | Create Cargo bindings, test linking |
| 3     | Tauri backend (device mgr, frame streaming) | 1.5 weeks | Commands, IPC, frame loop |
| 4     | React frontend (grid, tiles, controls) | 1.5 weeks | UI components, styling, responsiveness |
| 5     | Integration & testing | 1 week | E2E tests, performance tuning |
| 6     | Packaging & deployment | 0.5 weeks | Build exe, document setup |

**Total: 6-7 weeks**

---

## 13. Assumptions & Constraints

### Assumptions
- All devices connected via ADB (USB or TCP/IP)
- Devices running Android 5.0+ with USB debugging enabled
- Network stable enough for 30fps streaming
- User has administrative privileges to install ADB drivers

### Constraints
- No audio streaming (v1)
- Single master only (not daisy-chaining)
- Coordinate mapping assumes device resolution known
- Scrcpy library modifications require maintenance with upstream updates

---

## 14. References & Dependencies

### Core Libraries
- **scrcpy** (https://github.com/Genymobile/scrcpy) — Screen mirroring & control
- **Tauri** (https://tauri.app) — Desktop app framework
- **React** (https://react.dev) — UI framework
- **FFmpeg** (https://ffmpeg.org) — Video codec handling

### Build Tools
- **Meson** — Build system for scrcpy-core library
- **Cargo** — Rust package manager
- **Node.js** — JavaScript package manager

---

## Approval & Sign-Off

| Role | Name | Date | Signature |
|------|------|------|-----------|
| PM   | - | - | |
| Tech Lead | - | - | |
| QA | - | - | |

---

**Document Version:** 1.0  
**Last Updated:** 2026-06-23  
**Status:** Draft
