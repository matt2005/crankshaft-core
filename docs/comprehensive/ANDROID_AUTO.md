# Android Auto

## 1. Purpose and Scope

- Implement Android Auto projection lifecycle, transport setup, channel management, and session tracking.
- Handle device discovery, connection negotiation, media channel binding, and UI-facing event updates.

Out of scope:

- Phone-side Android Auto logic.
- UI rendering internals (owned by crankshaft-ui-slim).

## 2. Ownership and Boundaries

- Interface: `AndroidAutoService`.
- Real implementation: `RealAndroidAutoService` (AASDK based).
- Optional/mock path: `MockAndroidAutoService` through factory selection.
- Closely related services: `SessionStore`, `AudioRouter`, `MediaPipeline`, EventBus, WebSocket relay.

## 3. Architecture Summary

The Android Auto stack is centered around `RealAndroidAutoService`, which configures AASDK transport/channel components, drives state transitions, and forwards media/control events to Crankshaft subsystems. It manages channel lifecycle (video, media/system/guidance audio, control, input, sensors, Bluetooth, wifi projection), while persisting projection session metadata via `SessionStore`.

## 4. Component Map

| Component | File(s) | Responsibility |
| --- | --- | --- |
| AA abstract service | `src/services/android_auto/AndroidAutoService.h`, `src/services/android_auto/AndroidAutoService.cpp` | Shared API contract and factory |
| Real AA implementation | `src/services/android_auto/RealAndroidAutoService.h`, `src/services/android_auto/RealAndroidAutoService.cpp` | AASDK integration, channel orchestration, state machine |
| Session persistence | `src/services/session/SessionStore.h`, `src/services/session/SessionStore.cpp` | Session DB setup and state updates |
| Service lifecycle | `src/services/service_manager/ServiceManager.h`, `src/services/service_manager/ServiceManager.cpp` | Startup/stop orchestration from active profile |
| Event bridge | `src/services/eventbus/EventBus.h`, `src/services/websocket/WebSocketServer.cpp` | Broadcast service events and handle remote commands |

## 5. Runtime Sequence

1. `ServiceManager` starts `AndroidAuto` service based on active profile.
2. Factory resolves mock or real service (`useMock` config).
3. `RealAndroidAutoService` initializes thread/timers, `SessionStore`, and `AudioRouter`.
4. Transport setup and channel wiring begin (`setupAASDK`, channel setup routines).
5. Connection states progress from disconnected to connected as device negotiation succeeds.
6. Media/control/input events flow through AA channels to media pipeline and event bus.

Shutdown:

- Service ends current session, disconnects transport, stops searching, and cleans up AASDK/channel resources.

## 6. Data and Control Flows

### 6.1 Inbound Inputs

- USB device events via libusb/AASDK USB wrappers.
- AA channel payloads (video frames, audio frames, input/control responses).
- Config values for transport/channel behavior.
- UI commands relayed through websocket topics to service operations.

### 6.2 Processing

- State machine transitions for connection and session lifecycle.
- Channel-level handlers convert protocol frames to local actions.
- Session metadata persisted and heartbeats updated.
- Audio/video data forwarded to media stack and router.

### 6.3 Outbound Outputs

- EventBus topics and websocket broadcast payloads.
- Logs tagged with `[AA]` and `[RealAndroidAutoService]` context.
- Session DB records under writable app data path.

## 7. Configuration

| Key | Default | Effect | Notes |
| --- | --- | --- | --- |
| Profile `AndroidAuto.useMock` | false in production profile | Chooses mock vs real implementation | Processed by service factory path |
| AA transport/channel keys | varies by config | Enables/disables optional channels and tuning | Parsed from ConfigService and profile settings |
| Session storage path | Qt AppDataLocation + `/session.db` | Session persistence location | Depends on valid HOME/XDG setup |

## 8. External Dependencies

- `libusb` for USB transport discovery.
- AASDK libraries and headers.
- OpenSSL/TLS pieces used by transport stack.
- Local services: MediaPipeline, AudioRouter, EventBus.

Runtime assumptions:

- Service user has writable app data path.
- Required AA packages installed and ABI-compatible.

## 9. Observability and Diagnostics

Logs to watch:

- `[ServiceManager] Starting AndroidAuto service...`
- `[AA][...]` channel telemetry
- `[RealAndroidAutoService] ...`
- `[SessionStore] ...`

Checks:

- `systemctl status crankshaft-core`
- `journalctl -u crankshaft-core -b | grep -E "AndroidAuto|\[AA\]|SessionStore|ServiceManager"`
- USB visibility: `lsusb`

## 10. Failure Modes and Recovery

| Symptom | Likely Cause | Detection | Recovery |
| --- | --- | --- | --- |
| Device not discovered | USB access/cable/hub issue | No AA device events, no connected state transition | Validate USB enumeration, permissions, hardware path |
| Session DB init failure | Invalid HOME/XDG path or directory permissions | `SessionStore` directory create/open errors | Fix service HOME/XDG env and writable dirs |
| Projection starts but media missing | Channel/path mismatch or media pipeline state issue | AA state connected but no audio/video updates | Inspect channel logs and pipeline startup sequence |

## 11. Security and Safety Notes

- Protocol-facing code handles untrusted channel data; strict validation is required at boundaries.
- Service should remain least-privilege with explicit writable paths.
- Avoid logging sensitive phone/session identifiers beyond operational need.

## 12. Testing Strategy

- Unit tests for state transitions and channel handler guard paths.
- Integration test with physical Android device for connect/disconnect and media playback.
- Regression test for session-store initialization under service account.

Manual checklist:

1. Start core service and verify AndroidAuto device processing logs.
2. Connect a phone and verify state transitions.
3. Validate session DB creation and updates.
4. Verify disconnect cleanup and service stability.

## 13. Known Gaps and Follow-ups

- Some paths are still heavily tied to real hardware behavior and require richer integration testing.
- Add explicit health endpoints/metrics for channel readiness.
- Expand documentation coverage for wireless projection and fallback behavior.

## 14. Change Log Notes

- 2026-06: Documented startup dependencies and session-store environment coupling to avoid service-account path regressions.
