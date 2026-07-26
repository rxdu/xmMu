# HiPNUC serial IMU — protocol review & proposed updates

**Status: findings recorded — BLOCKED on vendor firmware confirmation before any driver change.** The core question (offset 1–2 semantics) is firmware-dependent; do not modify the driver until HiPNUC confirms the protocol/firmware for the deployed unit.

- **Reviewed:** `src/devices/sensor_imu/` — `src/ch_serial.c`, `src/ch_serial.h`, `src/imu_hipnuc.cpp`, `include/sensor_imu/imu_hipnuc.hpp` (identical at pin `8b95030` and `origin/main` `08f1568` at review time).
- **Reference:** HiPNUC unified manual (CUM), serial binary protocol, **Rev 1.7.2** — <https://download.hipnuc.com/en/products/imu/cum.html#serial-binary-protocol> (applies to firmware 1.7.1 and above).
- **Reviewed:** 2026-07-26.

## What is correct and current

The frame transport layer matches the current spec exactly and is hardened beyond a naive port. SOF `5A A5`, 6-byte header (2 SOF + 2 little-endian length + 2 CRC), and the checksum (`ch_serial.c:37` `crc16_update`, applied in `decode_ch` `ch_serial.c:197`) is CRC-16/XMODEM (poly `0x1021`, init `0x0000`, no reflection, no final XOR) computed over SOF+length+payload — exactly as specified.

The HI91 (`0x91`) fixed 76-byte payload is decoded at the correct byte offsets (`ch_serial.c:162` `KItemIMUSOL`): temp@3, pressure@4, timestamp@8, acc@12, gyr@24, mag@36, eul@48, quat@60 — byte-for-byte identical to the Rev 1.7.2 HI91 layout. Item decoding is bounds-guarded (`CH_ITEM_FITS`, `ch_serial.c:105`) so a truncated/malformed frame resyncs instead of reading past the payload.

## Findings

### F1 — Status word parsed as a sync counter (moderate; missed device-health data)

Spec Rev 1.7.2 defines HI91 bytes **1–2 as `main_status`**, a 16-bit health word: `WB_CONV` (gyro bias not converged), `MAG_DIST` (magnetic disturbance), `ACC_SAT`/`GYR_SAT` (saturation), `ATT_CONV` (attitude not converged), `STATIC` (stationary), `MAG_AIDING`, `UTC_UNSYNC`, `SOUT_PULSE`. The driver reads that field as **`pps_sync_ms`** ("time since last sync pulse", `ch_serial.h:33`, parsed at `ch_serial.c:164`) and `ToSample` never consumes it — so the device's own health/quality bits are parsed as a meaningless counter and discarded.

Impact: `ImuHipnuc::Health()` (`imu_hipnuc.cpp:112`) reports on freshness + transport-link fault only. The device is reporting `ATT_CONV`/`MAG_DIST`/saturation/`STATIC` and we throw it away — exactly the signals the HAL health surface and the MEKF/onboard-quaternion trust logic (in `xmAppBotBench` `imu_attitude`) would want.

⚠️ **Firmware-dependent — this is the item to confirm with the vendor.** Some older HiPNUC product lines historically carried a sync/pps value at this offset. The CUM is the *unified* manual; confirm the offset 1–2 meaning for the deployed unit's firmware before consuming it.

### F2 — No HI83 (`0x83`) support (feature gap)

The parser handles only HI91 + the legacy item format; the modern configurable **HI83** frame is absent. HI83 reports **accel in m/s² and gyro in rad/s natively** — the exact units the driver converts to at `imu_hipnuc.cpp:30-36` — plus a **uint64 µs device timestamp** and UTC. Supporting it would eliminate the G→m/s² and deg/s→rad/s conversions and provide a hardware timestamp instead of a host `steady_clock::now()`. Not a bug (HI91 remains a production format); the driver simply can't consume a unit configured for HI83.

### F3 — Legacy item handlers ARE the backward-compat layer (do NOT remove)

The int16-scaled item tags `0xA0/0xB0/0xC0/0xD0/0xD1/0xF0` (`ch_serial.c:120-160`) are not dead cruft — they decode the older item-based output that older firmware/products emit instead of `0x91`. **Removing them would break those devices.** Keep them.

### Minor notes (no change proposed)

- The HI91 device timestamp (`raw.imu.ts`, offset 8–11) is parsed but unused; `ToSample` stamps host receive time (`steady_clock::now()`, `imu_hipnuc.cpp:46`). This is a deliberate HAL choice (monotonic host-receive time), documented on `ImuSample` — noted only because the device clock is available if a hardware timestamp is ever wanted.
- Accel is negated on all three axes (`imu_hipnuc.cpp:30-32`) — a frame/sign convention, not a protocol issue. Worth confirming against the real unit, since the MEKF expects gravity as `(0,0,−g)`.

## Proposed updates (all BLOCKED on vendor firmware confirmation)

1. **[safe / cosmetic]** Rename `pps_sync_ms` → `main_status` in the struct and parser to match the spec. No offset change, no behavioral change — documentation accuracy only.
2. **[gated on firmware]** Surface the `main_status` health bits (`ATT_CONV`, `MAG_DIST`, `ACC_SAT`, `GYR_SAT`, `STATIC`) into `ImuSample` and/or `ImuHipnuc::Health()`. Only after the vendor confirms the field is `main_status` on the deployed firmware — otherwise risks false health signals.
3. **[additive]** Add HI83 (`0x83`) frame support for native m/s²/rad/s units + µs device timestamp. Opt-in; no impact on the HI91 or legacy-item paths.

## Backward-compatibility analysis

The HI91 frame *layout* is stable across firmware, and offset 1–2 is a fixed 2 bytes regardless of its meaning — so relabeling or consuming it does **not** shift any other field. Proposed updates #1 (rename) and #3 (add HI83) therefore break **no** firmware, old or new: all consumed fields (acc/gyro/mag/quat/temp) keep decoding identically.

The only two break risks, both avoidable: (a) removing the legacy item handlers (F3) would break older item-format firmware — so we keep them; (b) consuming `main_status` for `Health()` (#2) on firmware where offset 1–2 is *not* `main_status` would produce false health warnings — still not a data-parsing break, but a reason to gate #2 on confirmed firmware.

## Blocker / next step

Awaiting the latest firmware and protocol confirmation from HiPNUC — specifically the offset 1–2 semantics for the deployed unit. No driver change until confirmed. When confirmed: apply #1 unconditionally, #2 if the field is `main_status`, and #3 as an additive enhancement.
