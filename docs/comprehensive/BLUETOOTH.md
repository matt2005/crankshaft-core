# Bluetooth

## 1. Purpose and Scope

- Provide Bluetooth device management (enable/disable, scan, pair, connect, disconnect).
- Support Bluetooth audio-device workflows and route selection through core audio services.
- Expose Bluetooth operations/events to UI clients through websocket topics.

Out of scope:

- Detailed UI workflows.
- Vendor-specific Bluetooth stack customization beyond BlueZ/DBus APIs.

## 2. Ownership and Boundaries

- HAL layer: `BluetoothHAL` for BlueZ DBus interactions.
- Manager interface: `BluetoothManager` abstraction, with functional implementation used by `ServiceManager`.
- Service layer: `BluetoothService` coordinates preferences and audio routing.
- Event/API bridge: websocket topic handlers under `bluetooth/*`.

## 3. Architecture Summary

Bluetooth support is layered: low-level DBus adapter/device operations in `BluetoothHAL`, service-friendly operations in `BluetoothManager` implementations, and cross-service orchestration in `BluetoothService`. The service layer handles discovery timeout policy, selected-device persistence, and attempts to route media output using `AudioRouter` after connect.

## 4. Component Map

| Component | File(s) | Responsibility |
| --- | --- | --- |
| Bluetooth service API | `src/services/bluetooth/BluetoothService.h`, `src/services/bluetooth/BluetoothService.cpp` | Discovery/pair/connect orchestration and audio routing handoff |
| Manager abstraction | `src/hal/wireless/BluetoothManager.h`, `src/hal/wireless/BluetoothManager.cpp` | Interface contract for Bluetooth operations |
| BlueZ HAL | `src/hal/wireless/BluetoothHAL.h`, `src/hal/wireless/BluetoothHAL.cpp` | DBus adapter and device operations |
| Service lifecycle | `src/services/service_manager/ServiceManager.cpp` | Starts/stops Bluetooth manager based on profile or fallback |
| Remote command path | `src/services/websocket/WebSocketServer.cpp` | Handles `bluetooth/*` control topics |

## 5. Runtime Sequence

1. `ServiceManager` starts Bluetooth manager when profile includes Bluetooth (or fallback start via websocket request).
2. `BluetoothService::initialize()` reads timeout config and persisted selected device.
3. Discovery starts/stops with timeout enforcement.
4. Pair/connect operations are delegated to manager.
5. On successful connect, `BluetoothService` attempts `AudioRouter` device routing and persists selected device.

Shutdown:

- Discovery timers stop and manager/service lifecycles are released during service stop.

## 6. Data and Control Flows

### 6.1 Inbound Inputs

- Websocket topics: `bluetooth/enabled`, `bluetooth/scan/request`, `bluetooth/pair`, `bluetooth/connect`, etc.
- Hardware/DBus events from BlueZ adapter/device changes.
- Config key for discovery timeout.

### 6.2 Processing

- Device discovery results filtered for audio-relevant devices.
- Pair/connect requests executed through manager.
- Audio routing handoff via `AudioRouter::setAudioDevice(...)`.
- Preference persistence through `PreferencesService` for selected audio endpoint.

### 6.3 Outbound Outputs

- Service signals (`deviceDiscovered`, `devicePaired`, `audioRoutingError`, etc.).
- Websocket result events (`bluetooth/*/result`).
- Structured logs from Bluetooth service and websocket layers.

## 7. Configuration

| Key | Default | Effect | Notes |
| --- | --- | --- | --- |
| `core.bluetooth.discovery_timeout_seconds` | 60 | Auto-stop discovery after timeout | Set to `0` to disable timeout behavior |
| `bluetooth/selectedAudioDevice` (preference key) | empty | Remember preferred output device | Stored/read via `PreferencesService` |

## 8. External Dependencies

- BlueZ services on system DBus.
- Qt DBus and Qt Bluetooth support.
- `AudioRouter` availability for routing integration.

Runtime assumptions:

- Bluetooth adapter is present and visible to system DBus.
- Service user has sufficient permissions for Bluetooth operations.

## 9. Observability and Diagnostics

Logs to watch:

- `[BluetoothService] Initialized`
- `[BluetoothService] Discovery started...`
- `[BluetoothService] Failed to connect to device ...`
- `[WebSocketServer] Bluetooth scan started; timeout in ...`

Checks:

- `journalctl -u crankshaft-core -b | grep -E "BluetoothService|bluetooth|WebSocketServer"`
- `bluetoothctl show`
- `bluetoothctl devices`

## 10. Failure Modes and Recovery

| Symptom | Likely Cause | Detection | Recovery |
| --- | --- | --- | --- |
| Scan does not return devices | Adapter unavailable or discovery failure | No `deviceDiscovered` events, scan result shows failure | Validate adapter state and DBus availability |
| Pair/connect fails repeatedly | Remote device rejects pairing/profile mismatch | `bluetooth/*/result` success false and service warnings | Remove stale pairing, retry from clean state, verify profiles |
| Audio not routed after connect | Router/device-id mismatch or backend unavailable | Routing error emitted after connect | Check `AudioRouter` backend status and mapping logic |

## 11. Security and Safety Notes

- Bluetooth pairing exposes external trust boundary; only trusted user flows should trigger pair/connect.
- Websocket command path must validate payloads and reject malformed addresses.
- Avoid persisting sensitive metadata beyond device identity and route preference.

## 12. Testing Strategy

- Unit tests for discovery timeout behavior and preference persistence logic.
- Integration tests with BlueZ test doubles or controlled adapter environment.
- Manual tests for pair/connect/disconnect plus audio-route handoff.

Manual checklist:

1. Enable Bluetooth and confirm adapter state transitions.
2. Run scan and verify timeout behavior.
3. Pair/connect known audio device.
4. Verify audio route update and persistence after restart.

## 13. Known Gaps and Follow-ups

- `BluetoothService::getDiscoveredAudioDevices()` currently contains a duplicated device-type condition and should be cleaned up for clarity.
- Add explicit profile-level validation for A2DP/HFP route compatibility before routing.
- Add richer telemetry for pairing/connection failure categories.

## 14. Change Log Notes

- 2026-06: Added comprehensive Bluetooth architecture documentation with service, manager, HAL, and websocket control-path coverage.
