# Changelog

All notable changes to this project are documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/).

## [1.0.0] - 2026-07-14

First public release.

### Added

- Gen1 (`PMD*`, `PWS*`) and Gen2 (`WD_*`) device support, with generation auto-detected from the advertised name.
- Dual-line (50A) support: L1/L2 entities created automatically when dual-line operation is detected.
- Cumulative Total Power and Total Energy sensors for dual-line units.
- Gen2 enhanced model (E8/V8/E9/V9) support: output voltage and temperature sensors.
- Gen2 controls: relay on/off, backlight level, neutral detection toggle, energy reset, time sync.
- Signal Strength (RSSI) diagnostic sensor, refreshed on every reconnect.
- Connection and data-health diagnostic binary sensors.
- Auto-discovery via BLE advertisement, plus manual setup by MAC address.
- MAC-derived startup delay to stagger BLE connection attempts across integrations after an HA restart.

### Fixed

- **Gen1 chunk-pairing desync**: a dropped or reordered BLE notification (most often right after a reconnect) could leave the 20-byte chunk pairing off by one, silently producing frames with physically impossible values (e.g. L2 Frequency spiking into the tens of thousands of Hz) that persisted until the next full Home Assistant restart. `Gen1FrameAssembler` now detects and resyncs a desynced pairing automatically instead of mispairing.
- **Frame plausibility validation**: `parse_gen1_frame()` now rejects any frame where voltage, current, power, energy, or frequency falls outside a physically possible range, as a backstop against corrupted data reaching entity state or long-term statistics.
- BlueZ notify-slot leakage (`NotPermitted` / ATT 0x0e errors) on reconnect, fixed via `stop_notify` before disconnect and a retry loop around `start_notify`.
- Duplicate energy state writes suppressed to avoid log spam after a power outage.

### Changed

- Energy sensors (`energy_l1`, `energy_l2`, `energy_total`) now use `state_class: total` with a fixed reset epoch instead of `total_increasing`. A `total_increasing` sensor treats any decrease as a meter reset and pads the long-term statistics sum accordingly; `total` accumulates signed deltas instead, so a rare corrupted reading is cancelled out by its own recovery drop rather than permanently corrupting Energy Dashboard statistics.
- Integration domain renamed to `ha_hughes`.

### Known limitations

- Sustained BLE packet loss on Gen1 dual-line units causes affected frames to be discarded (via the resync/plausibility logic above) rather than corrupting data — this can reduce effective sample rate under poor signal conditions, but won't produce bad readings.
- RSSI reflects the signal at the most recent connection attempt, not a continuous live reading, since the device stops advertising once connected.
