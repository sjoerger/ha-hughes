# Hughes Power Watchdog HACS Integration — Technical Specification

## 1. Purpose and Scope

`ha_hughes` integrates Hughes Power Watchdog BLE devices (Gen1 and Gen2) into Home Assistant, including line telemetry, diagnostics, and Gen2 controls.

## 2. Integration Snapshot

- **Domain:** `ha_hughes`
- **Primary runtime component:** `HughesCoordinator`
- **Platforms:**
  - common: `binary_sensor`, `sensor`
  - Gen2 only: `button`, `number`, `switch`
- **Transport:** BLE GATT
- **Coordinator mode:** notification-driven with watchdog/reconnect handling

## 3. Configuration and Entry Setup

- discovery identifies likely Hughes devices by name/service patterns
- config flow stores address, detected generation, and device name
- setup conditionally loads Gen2-only platforms when generation is Gen2

## 4. Runtime Lifecycle

1. Entry setup creates coordinator and starts background connect.
2. Coordinator performs generation-specific service validation.
3. Notification handlers are registered.
4. Gen1/Gen2 parser pipelines decode telemetry into state model.
5. Watchdog enforces stale-data recovery behavior.
6. Disconnect resets parser/session state and schedules reconnect.

## 5. Protocol and Transport Model

### 5.1 Gen1 model

- notification chunks are assembled into fixed-size frames
- parsed fields provide line metrics and error codes

Gen1 transport/protocol details:

- service: `0000ffe0-0000-1000-8000-00805f9b34fb`
- notify characteristic: `0000ffe2-0000-1000-8000-00805f9b34fb`
- optional write characteristic defined for future control: `0000fff5-0000-1000-8000-00805f9b34fb`
- chunk size: `20` bytes, assembled frame size: `40` bytes
- frame header validation: `0x01 0x03 0x20`
- numeric scaling: big-endian int32 values with `÷10000` for voltage/current/power/energy, frequency `÷100`

Gen1 chunk-pairing resync and plausibility validation (`protocol/gen1.py`):

- the frame header check only validates the first chunk's bytes, so a dropped or reordered BLE notification (most often right after a reconnect) can leave the two-chunk pairing off by one without failing that check, silently yielding a frame with garbage in one or more fields
- `Gen1FrameAssembler` detects this case — an incoming chunk that itself starts with the frame header while a chunk is already pending means the true second chunk never arrived — and resyncs onto it immediately instead of mispairing; this has been observed to trigger routinely on some connections (multiple times per second), not only at reconnect, and is logged at `debug` since it is expected, self-correcting behavior
- as a backstop, `parse_gen1_frame()` rejects (returns `None` for) any frame where voltage, current, power, energy, or frequency falls outside a physically plausible bound (`GEN1_VOLTAGE_MAX`, `GEN1_CURRENT_MAX`, `GEN1_POWER_MAX`, `GEN1_ENERGY_MAX`, `GEN1_FREQ_MIN`/`GEN1_FREQ_MAX` in `const.py`), logged at `warning` since a rejected frame is real (if brief) data loss

### 5.2 Gen2 model

- RW characteristic with protocol-open handshake
- binary framing parser reconstructs packets
- DLReport command payloads provide line telemetry and status

Gen2 transport/protocol details:

- service: `000000ff-0000-1000-8000-00805f9b34fb`
- read/write/notify characteristic: `0000ff01-0000-1000-8000-00805f9b34fb`
- open command: ASCII `!%!%,protocol,open,`
- packet header magic: `0x24 0x7C 0x27 0x40`
- packet tail: `0x71 0x21`
- header fields: `magic(4) + version(1) + msg_id(1) + cmd(1) + data_len(2)`
- DLReport body lengths: `34` (single line) or `68` (dual line)

## 6. State and Entity Model

- coordinator tracks generation/enhanced/dual-line capabilities
- `HughesState` contains line metrics, errors, and health-relevant telemetry
- enhanced Gen2 devices expose additional values (e.g., output voltage/boost/temp)
- entity platforms mirror capability and generation-specific availability

### 6.1 Cumulative (L1 + L2) sensors

`HughesCumulativeSensor` provides two derived entities for dual-line (50amp) units:

| Entity | Key | Value | Unit |
|---|---|---|---|
| Total Power | `power_total` | `line1.power + line2.power` | W |
| Total Energy | `energy_total` | `line1.energy + line2.energy` | kWh |

- both are marked unavailable when `state.is_dual_line` is `False` or `line2` is `None`
- implemented as a separate entity class to allow reading both line fields simultaneously, rather than the single-line `value_fn` pattern used by `HughesSensor`
- `energy_total`, and the per-line `energy_l1`/`energy_l2` entities in `HughesSensor`, use `SensorStateClass.TOTAL` (not `TOTAL_INCREASING`) with a fixed `last_reset` epoch of `1970-01-01T00:00:00Z`. `TOTAL_INCREASING` treats any decrease as a meter reset and pads the long-term statistics sum accordingly; `TOTAL` accumulates signed deltas instead, so a corrupted reading that briefly slips past frame-plausibility validation is cancelled out by its own recovery drop rather than permanently corrupting recorder statistics

## 7. Command and Control Surface

Gen2 write operations include:

- relay control
- neutral-detection toggling
- backlight adjustment
- energy reset
- time sync

Writes are generated by Gen2 command builders with characteristic guards.

Gen2 command codes used:

- `0x0B` set relay open/close
- `0x07` set backlight
- `0x0D` neutral detection
- `0x03` energy reset
- `0x06` set time

Relay values are `0x01` (on) and `0x02` (off); backlight is clamped to `0..5`.

## 8. Reliability and Recovery

- background startup connection to avoid HA boot blocking
- reconnect backoff with cap
- stale-data watchdog and recovery
- parser/session reset on disconnect
- tolerant Gen2 initialization when optional handshake ack is absent
- Gen1 chunk-pairing resync: automatic recovery from a desynced chunk pairing without requiring a reconnect (see §5.1)
- Gen1 frame-level plausibility rejection across all telemetry fields as a backstop against corrupted frames reaching entity state or recorder statistics (see §5.1)

Timing constants:

- service discovery delay: `0.2s`
- operation delay: `0.1s`
- Gen2 init timeout: `3.0s`
- Gen2 settle delay: `0.2s`
- reconnect base/cap: `5s` / `120s`
- stale timeout: `300s`
- watchdog interval: `60s`

## 9. Diagnostics and Observability

- connection and data-health binary sensors
- line error code plus mapped diagnostic text
- generation/device metadata diagnostics
- integration diagnostics export for coordinator/parser state
- Signal Strength (RSSI) diagnostic sensor: reads the most recent BLE advertisement RSSI via `bluetooth.async_last_service_info` (connectable, falling back to non-connectable). Since the device stops advertising once a GATT connection is established, this reflects the RSSI seen at the most recent connection attempt rather than updating continuously through a session — it refreshes on every reconnect

## 10. Security and Safety Notes

- generation-gated command exposure avoids invalid control paths
- conservative parser boundary checks reduce malformed-state risk
- local BLE-only control path (no cloud dependency in integration)

## 11. Evolution Notes (Commit History)

Recent trajectory includes:

- initial Gen1/Gen2 HA-native implementation
- startup/connect robustness updates
- migration to `ha_hughes` domain naming
- HACS/CI/repository hardening
- **v1.0.3:** cumulative `Total Power` and `Total Energy` sensors for dual-line (50amp) units via new `HughesCumulativeSensor` entity class
- **v1.0.4:** HACS default store publication prep — brand `icon.png` added at `custom_components/ha_hughes/brand/`, `manifest.json` keys sorted to `domain`/`name`/alphabetical per Hassfest requirements, removed invalid `domains` key from `hacs.json`
- **v1.0.0 (this release):** Gen1 frame-corruption fix, root-caused via Home Assistant history analysis after a dual-line unit was observed producing physically impossible readings (e.g. L2 Frequency spiking to ~96,000-300,000 Hz) that persisted continuously after a BLE reconnect with no self-recovery. `Gen1FrameAssembler` now detects and resyncs a desynced chunk pairing instead of silently mispairing (see §5.1); `parse_gen1_frame()` rejects frames with any field outside a physically plausible range as a backstop; `energy_l1`/`energy_l2`/`energy_total` moved from `TOTAL_INCREASING` to `TOTAL` with a fixed `last_reset` so a corrupted reading can no longer be misread as a meter reset and corrupt long-term statistics (see §6.1). The broadened field validation and energy state-class change mirror an equivalent fix independently discovered upstream (`phurth/ha-hughes`, issue #3); this fork's chunk-resync mechanism was kept as the primary fix since upstream's approach only rejects bad frames without correcting the underlying pairing.

## 12. Known Constraints

- wrong generation detection can limit platform/features until corrected
- Gen1 BLE packet loss triggers chunk-pairing resync (§5.1), which discards the affected frame rather than completing it — under sustained loss this can noticeably reduce effective sample rate even though no corrupted values reach entity state
- Gen2 firmware behavior may vary around optional open-ack timing

## 13. Extension Guidelines

1. Keep generation-specific protocol logic isolated in protocol modules.
2. Preserve strict frame boundary and payload-size validation.
3. Gate new writable features by generation and capability detection.
4. Maintain dual-line conditional entity creation rules.
5. Add diagnostic mapping updates alongside new protocol features.
