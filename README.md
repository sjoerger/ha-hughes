# ha-hughes

Home Assistant HACS integration for **Hughes Power Watchdog** surge protectors and power management devices.

> **Disclaimer:** This is an independent community integration and is not affiliated with, endorsed by, or supported by Hughes or any of its affiliates. Use it at your own risk.

Connects directly to the device over Bluetooth Low Energy — there is no cloud or internet dependency.

> Power Watchdog EPO devices can only connect to one bluetooth device at a time. If you have issues, ensure there is not another app trying to connect to the EPO.

## Features

- **Auto-discovery** via BLE advertisements (name prefix `PMD*`, `PWS*`, or `WD_*`)
- **Real-time power monitoring**: voltage, current, power, energy, frequency
- **Dual-line (50A) support**: L1 and L2 entities created automatically when dual-line operation is detected
- **Gen1 and Gen2 device support**: generation auto-detected from device name
- **Gen2 control commands**: relay on/off, backlight level, neutral detection, energy reset, time sync
- **Diagnostics**: connection health binary sensor, BLE signal strength, raw frame dump, one-click diagnostics download

## Entities

### All devices (Gen1 and Gen2)

| Entity | Type | Device Class | Unit | Notes |
|--------|------|-------------|------|-------|
| L1 Voltage | Sensor | voltage | V | |
| L1 Current | Sensor | current | A | |
| L1 Power | Sensor | power | W | |
| L1 Energy | Sensor | energy | kWh | Total (see note) |
| L1 Frequency | Sensor | frequency | Hz | |
| L1 Error | Sensor (diag) | — | — | Text description |
| L1 Error Code | Sensor (diag) | — | — | Numeric code |
| L2 Voltage | Sensor | voltage | V | Dual-line only |
| L2 Current | Sensor | current | A | Dual-line only |
| L2 Power | Sensor | power | W | Dual-line only |
| L2 Energy | Sensor | energy | kWh | Dual-line only; Total (see note) |
| L2 Frequency | Sensor | frequency | Hz | Dual-line only |
| L2 Error | Sensor (diag) | — | — | Dual-line only |
| L2 Error Code | Sensor (diag) | — | — | Dual-line only |
| Connected | Binary Sensor (diag) | connectivity | — | |
| Data Healthy | Binary Sensor (diag) | problem | — | |
| Signal Strength | Sensor (diag) | signal_strength | dBm | Updated on reconnect only — see note |

> **Signal Strength (RSSI) note:** The PMD stops advertising once a GATT connection is established — standard BLE behavior. The Signal Strength sensor therefore captures the RSSI seen during the most recent connection attempt rather than updating continuously during a session. It refreshes on every reconnect, making it useful for comparing signal quality when moving to a new location or diagnosing range-related instability. As a guide: -55 dBm or better is excellent; -75 dBm is marginal; -80 dBm or below may contribute to connection instability.
>
> **Energy (Total vs. Total Increasing) note:** Energy sensors use `state_class: total` with a fixed reset epoch rather than `total_increasing`. A cumulative energy counter should never decrease in normal operation, but a corrupted Gen1 frame occasionally could (see Protocol below) — under `total_increasing`, HA's statistics engine treats any decrease as a meter reset and pads the long-term statistics sum with the full new value, which can corrupt historical totals into wildly implausible numbers even though the live sensor value stays correct. `total` instead accumulates signed deltas, so a glitch spike is cancelled out by its own recovery drop.

### Gen2 only

| Entity | Type | Notes |
|--------|------|-------|
| Relay | Switch | Main output relay |
| Neutral Detection | Switch | Enable/disable neutral monitoring |
| Backlight | Number | Display brightness (0–5) |
| Reset Energy | Button | Clear cumulative energy counter |
| Sync Time | Button | Push current UTC time to device |

### Gen2 enhanced models (E8, V8, E9, V9) only

| Entity | Type | Device Class | Unit |
|--------|------|-------------|------|
| L1 Output Voltage | Sensor | voltage | V |
| Temperature | Sensor | temperature | °F |
| L1 Boost Active | Binary Sensor (diag) | — | — |

## Requirements

- Home Assistant 2024.1+ with Bluetooth integration
- Bluetooth adapter on the HA host (or ESPHome BT proxy)
- Hughes Power Watchdog device within BLE range

## Installation (HACS)

1. Add this repository as a custom HACS repository
2. Install "Hughes Power Watchdog"
3. Restart Home Assistant
4. The device should auto-discover — or add manually via Settings → Devices & Services → Add Integration → Hughes Power Watchdog and enter the Bluetooth MAC address

## Startup Delay

When Home Assistant restarts, all BLE integrations attempt to connect simultaneously, which can exhaust the Bluetooth adapter's connection slots and cause failures. This integration uses a MAC-derived startup delay to stagger its connection attempt relative to other BLE integrations.

The delay is calculated from the last octet of the device's MAC address:

```
delay = (last MAC octet) / 20.0   →   range: 0–12.75 seconds
```

For example, a device with MAC `40:79:12:B6:33:9B` has last octet `0x9B` = 155, giving a startup delay of `155 / 20 = 7.75s`.

The delay is deterministic — the same device always gets the same delay — and requires no user configuration.

**To adjust the spread**, edit `__init__.py` and change the divisor:

```python
startup_delay = mac_offset / 20.0   # default: 0–12.75s spread
```

| Divisor | Max delay | When to use |
|---------|-----------|-------------|
| `10.0` | 25.5s | Many co-located BLE integrations |
| `20.0` | 12.75s | Default — good for up to 5 devices |
| `40.0` | 6.4s | Fast adapter, few integrations |

To disable the delay entirely, set `startup_delay = 0.0`.

## Protocol

### Gen1 (PMD\*, PWS\* name prefix)

- **Service**: `0000FFE0-0000-1000-8000-00805F9B34FB`
- **Notify** (`FFE2`): 20-byte chunks assembled in pairs into 40-byte frames
- **Notify** (`FFF5`): command response channel, enabled on connect
- Big-endian int32 values ÷ 10000 for electrical measurements
- No authentication or pairing required
- **Chunk resync**: a dropped or reordered BLE notification (most often right after a reconnect) can leave the 20-byte chunk pairing off by one, silently producing frames with garbage in one or more fields while other fields still look plausible. `Gen1FrameAssembler` detects this — an incoming chunk that itself starts with the frame header while a chunk is already pending means the true second chunk never arrived — and resyncs immediately instead of mispairing. On some connections this can trigger routinely (observed as bursts of several resyncs per second, not just at reconnect); this was verified against HA history to still produce continuous, physically plausible data, so resync events are logged at `debug` rather than `warning` to avoid flooding the log for expected, self-correcting behavior. Enable debug logging (see below) if you want to see how often it's happening on your connection.
- **Plausibility backstop**: as a second layer of defense, `parse_gen1_frame()` rejects any frame where voltage, current, power, energy, or frequency falls outside a physically plausible range (e.g. frequency outside 45–65 Hz) rather than letting it reach Home Assistant, even if the resync heuristic above misses a case.

### Gen2 (WD\_\* name prefix)

- **Service**: `000000FF-0000-1000-8000-00805F9B34FB`
- **Read/Write/Notify** (`FF01`): Binary framed packets with magic header `0x247C2740`
- ASCII handshake (`!%!%,protocol,open,` → `ok`) enters binary mode
- `DLReport` (0x01) carries 34-byte (single) or 68-byte (dual-line) measurement body
- No authentication or pairing required

## Debug Logging

```yaml
logger:
  logs:
    custom_components.hughes: debug
```

Use **Download diagnostics** from the integration page for a full runtime state dump including raw frame bytes (Gen2), connection health, and all parsed values.

## Troubleshooting

### BLE connection instability

- Ensure no other app (e.g. the Hughes mobile app) has the device paired or connected — Power Watchdog EPO devices support only one BLE connection at a time.
- If running multiple BLE integrations (EasyTouch, OneControl, etc.) on a single adapter, consider adding an [ESPHome Bluetooth proxy](https://esphome.github.io/bluetooth-proxies/) near the Power Watchdog to give it a dedicated connection path.
- Check that the device is within reliable BLE range. RSSI below -80 dBm indicates marginal range and may cause intermittent drops.

### Device drops connection under heavy load

- Hughes Power Watchdog devices may reset their BLE radio when the surge protector trips or responds to voltage events (overvoltage, undervoltage, overcurrent).
- On 50A (dual-line) installations with long shore power cables, voltage sag under heavy load (AC compressor starts, etc.) can trigger protection events that reset the BLE stack. This is normal device behavior.

### Both L1 and L2 not appearing

- L2 entities are created automatically when the first L2 telemetry frame is received. On first installation, wait up to 60 seconds after initial connection for L2 data to appear.
- Confirm dual-line operation in the logs: `Gen1 data from <addr>: L2 <V> / <A> / error=OK`

### L1/L2 values spike to impossible numbers / graphs look erratic

- This is a known Gen1 issue: a BLE notification dropped or reordered during a reconnect could desync the 20-byte chunk pairing, producing frames with garbage in one or more fields that still passed the (too-weak) frame header check. It typically showed up as `L2 Frequency` reading tens of thousands of Hz, and could cause energy readings to appear non-monotonic.
- Fixed by chunk-pairing resync detection plus a plausibility check across all fields (see **Chunk resync** and **Plausibility backstop** under Protocol above), together with switching energy sensors from `total_increasing` to `total` so a decrease is never misread as a meter reset — update to the latest version if you're still seeing this.
- Historical corrupted data recorded before updating is mostly cosmetic (it lives in the History graph for that period) and does not need manual cleanup unless those entities feed your Energy Dashboard, in which case check for a phantom consumption spike on the affected date and use Home Assistant's statistics "Adjust sum" tool to correct it.

## License

MIT
