# Empirical Bolt EV Pack Data — Standalone (No-BECM) Operation

Raw observable measurements from a Chevrolet Bolt EV battery (LG N2.2 NMC, 64 kWh, 96S3P) operated standalone — without vehicle BECM master. 49 days of monitoring including UDS PID polling responses, per-cell voltages, and rare deep-discharge events.

Goal: provide raw battery data for the open-source community working on Bolt BMS reverse engineering and BECM emulation, particularly cell balancing.

## Headline finding

**The standalone BICM never autonomously balances cells.** UDS PID 0x431C (`HYBRID_BATTERY_CELL_BALANCE_STATUS`) returns `0` (idle) across 4,113 polled responses including extreme conditions (cell delta 150 mV, cell min 3.265 V at deep discharge).

Confirms "BECM master required" hypothesis from prior community discussion ([Battery-Emulator #1848](https://github.com/dalathegreat/Battery-Emulator/discussions/1848)).

## Pack identification

| Field | Value |
|---|---|
| Source vehicle | Chevrolet Bolt EV 2022 |
| Manufacturer P/N | GM 24297220 |
| Manufacturing date | Week 12, 2022 (USA) |
| Platform | BEV2 / VITM |
| Chemistry | LG N2.2 (NMC) |
| Topology | 96S3P (96 series, 3 parallel) |
| Nominal capacity | 64 kWh |
| Nominal voltage | ~350 V |
| Cell module count | 10 modules |
| BMS | OEM BICM/VITM |

## Operating context

- Pack operated **standalone** — no vehicle BECM in circuit. BICM powered via external 12 V supply.
- Stationary inverter/storage application (Deye SUN-80K hybrid via BYD-CAN protocol).
- Internal CAN bus 500 kbps, listening + UDS polling on `0x7E7` (BICM target).
- No vehicle harness — only HV+/HV-, BMS data CAN, BMS power 12 V, contactor control.
- **Monitoring period**: 2026-03-23 → 2026-05-10 UTC (49 days).
- **Sampling**: ~30 s aggregation cadence.
- **Extended-PID polling window**: from 2026-05-09 11:10 UTC onwards (~31 hours, 4,113 responses to PID 0x431C and others).

## Datasets — raw observations only

| File | Rows | Period | Description |
|---|---|---|---|
| `01_pid_431C_balance_status_log.csv` | 4,113 | 31 h | Time series of UDS PID 0x431C raw responses + raw cell min/max/voltage/current/temp |
| `02_uds_pid_responses_raw.csv` | 4,113 | 31 h | Raw responses from polled PIDs (0x431C, 0xC218, 0x40D4, 0x438B, 0x40D3, 0x428F, 0x4290, 0x8043) |
| `03_extreme_imbalance_events.csv` | 1,980 | 49 days | Every sample where cell_delta ≥ 100 mV OR cell_min ≤ 3.30 V |
| `04_cell_voltage_snapshots.csv` | 5 | various | Full 96-cell voltage arrays at 5 representative operating points |
| `05_daily_pack_observations.csv` | 40 | 49 days | Daily aggregate: delta range, voltage extremes, peak currents, temperatures |
| `06_per_cell_rank_stability.csv` | 96 | 49 days | For each cell position: how often it was the pack min/max across 99,000+ samples |
| `07_module_temperatures_daily.csv` | 40 | 49 days | Daily per-module temperature averages (6 module probes) |
| `09_cell_min_voltage_histogram.csv` | 18 | 49 days | Distribution of pack min cell voltage in 50 mV bins |
| `10_recovery_event_2026-05-09_trace.csv` | 242 | 2 h | High-resolution trace of recovery from deep-discharge event (cell delta 148 → 35 mV) |
| `11_hourly_pack_aggregates.csv` | 835 | 49 days | Hourly averages of cell voltages, current, voltage, temperatures |
| `12_can_log_bms_bus_60s_raw.csv` | 31,800 | 60 s | **Raw CAN bus capture** (CAN-FD, 500 kbps): every frame on the BICM-side bus during a 60-second window. Hex payloads, decoded timestamps, direction (rx from BICM / tx from polling node), DLC, frame format flags. Standard candump-compatible columns. |
| `13_can_id_inventory_summary.csv` | 31 | 60 s | Per-CAN-ID aggregate from dataset 12: hex ID, direction, frame count, average frequency (Hz), average period (ms), first observed payload bytes. |

## UDS PID raw responses — observations needing community decoding

Polled on UDS service 0x22 (ReadDataByIdentifier) at `0x7E7` request → `0x7EF` response. PID names from [Battery-Emulator BOLT-AMPERA-BATTERY.cpp](https://github.com/dalathegreat/Battery-Emulator/blob/main/Software/src/battery/BOLT-AMPERA-BATTERY.cpp).

| PID | Documented name | Observed behavior in this dataset |
|---|---|---|
| `0x431C` | HYBRID_BATTERY_CELL_BALANCE_STATUS | Always `0` across 4,113 samples. Standalone BICM never reports active balancing. |
| `0xC218` | (cell average) | Range 3,378–3,650 mV, follows pack SOC. Consistent with computed cell average from frame 0x200-0x208. |
| `0x40D4` | INTERNAL_CURRENT (int16) | Narrow range -11823 .. -11812 (only 8 distinct values across 31 h, including ±10 A active discharge). Does NOT track instantaneous pack current. Possibly a register/baseline value. |
| `0x438B` | HV_BUS_VOLTAGE (uint8) | **Zero responses received.** BICM never replies to this PID. |
| `0x40D3` | 5V_REF_MAIN | ~4,809 mV. Consistent with name (5 V supply rail measurement). |
| `0x428F` | 5V_REF_1 | Only 2 distinct values: 3001 / 3020 mV. Does not match expected ~2,500 mV (Uc/2). Either wrong scaling or wrong PID label. |
| `0x4290` | 5V_REF_2 | Same as 0x428F. |
| `0x8043` | GMLAN_STATUS | Single-byte status code (see CSV for values). |

Each row in `02_uds_pid_responses_raw.csv` carries the raw bytes/decoded integer per PID with a UTC timestamp.

## CAN bus inventory (from dataset 13)

Standalone BICM emits the following CAN frames during steady-state idle operation (60 s capture, 31 distinct IDs):

| ID | Direction | Frequency | DLC | Notes |
|---|---|---|---|---|
| 0x091 | rx | 1.20 Hz | 8 | unknown — undocumented in upstream |
| 0x0D1 | rx | 1.18 Hz | 8 | unknown |
| 0x111 | rx | 1.20 Hz | 8 | unknown |
| 0x200, 0x202, 0x204, 0x206, 0x208 | rx | 40 Hz each | 8 | Cell voltage matrix (multiplexed) |
| 0x20C | rx | 40 Hz | 6 | VITM Status HV |
| 0x216 | rx | 40 Hz | 7 | Sensed voltage / current |
| 0x260, 0x262 | rx | 20 Hz | 8/3 | Diagnostic status |
| **0x270, 0x272, 0x274** | rx | 20 Hz | 8/8/2 | **BalancingSwitches diagnostic** — primary candidate for community decoding |
| 0x2C7 | rx | 40 Hz | 6 | Pack voltage |
| 0x302 | rx | 10 Hz | 8 | Module temperatures |
| 0x308 | rx | 10 Hz | 5 | unknown |
| 0x3E3 | rx | 10 Hz | 7 | min/max values |
| 0x460 | rx | 4 Hz | 4 | Coolant inlet/outlet temps |
| 0x7EF | rx | 10 Hz | 8 | UDS responses to 0x7E7 polls |

Full per-frame raw payloads in dataset 12. See `13_can_id_inventory_summary.csv` for first-seen payload bytes per ID.

## Open questions for the community

1. **Decode `0x270`, `0x272`, `0x274` BalancingSwitches frames** — these emit at 20 Hz but interpretation is unknown. Initial payload samples in dataset 13 (`0x270 = 4B4B4B484B4B4B48`, `0x272 = 4848484848484848`, `0x274 = 4848`) — appear to be repeating byte patterns suggesting some kind of cell-state bitfield. Decoding this would directly answer "is BICM signalling balancing intent."

2. **Decode `0x4323-0x4327` and `0x4340`** (HYBRID_CELL_BALANCING_ID 1-6) — Battery-Emulator declares them as poll-able PIDs but no public interpretation exists. Not yet polled in this dataset. Adding them is straightforward firmware change for any researcher with similar hardware.

3. **Decode unknown periodic frames** `0x091`, `0x0D1`, `0x111`, `0x308` — 1.2-10 Hz cadence, undocumented in Battery-Emulator. Could be sensor data, status flags, or BICM internal state.

3. **Resolve `0x40D4` interpretation** — what does it actually return if not instantaneous current? Multi-pack comparison would help.

4. **`0x438B` non-response** — wrong PID number, wrong service ID, or requires UDS Security Access (0x27) precondition?

5. **`0x428F` / `0x4290` scaling** — 3001/3020 mV doesn't fit "5V reference" semantics. What are these PIDs really?

6. **Capture BECM-to-BICM `0x20E` "Hybrid balancing request HV"** — Battery-Emulator TODO. Need a J2534 capture from a working in-vehicle Bolt's K16 X8 internal CAN tap (pins 4/5 or 14/15).

## Pack-level observations from `06_per_cell_rank_stability.csv`

Across 99,107 samples (49 days), some cell positions are persistently at the extremes:

- Cell position 12: at pack maximum 47.94 % of all samples (avg 3,630 mV when at maximum)
- Cell position 49: at pack minimum 19.30 % of all samples (avg 3,544 mV when at minimum)
- Cell position 1: at both extremes ~17 % each (edge-cell behavior)
- Most other positions are at extremes < 1 % of samples (i.e., never)

Without active balancing, these positional extremes persist — there is no mechanism to redistribute charge between cells.

## Pack-level observations from `09_cell_min_voltage_histogram.csv`

Pack minimum cell voltage distribution over 49 days, in 50 mV bins:

- ~60 % of operating time at cell min ≈ 3,500 mV
- ~19 % at cell min ≈ 3,550 mV
- ~2 % at cell min ≈ 3,250 mV (deep-discharge events)
- Cell delta is well-controlled (15–22 mV avg) when cell min is in the 3,500–3,650 mV band; widens to 30–135 mV at lower or higher voltages.

## What this dataset does NOT include

By design, this dataset is **raw observable measurements only**. It does not include:

- Estimated SOC values (SOC for the Bolt pack is not BMS-reported and any estimate is implementation-specific to the polling system)
- Algorithmic outputs (charge/discharge limits, throttle values, alarm flags, balancing decisions)
- Configuration values
- Source code, firmware version, internal architecture

Only physical battery measurements + raw UDS PID response bytes + raw CAN-frame-derived cell voltages and module temperatures. This is what GM hardware physically reports — anyone with comparable polling can replicate.

## Methodology

- Hardware: ESP32-S3 with Microchip MCP2518FD CAN-FD controller (commodity off-the-shelf parts)
- CAN bus: 500 kbps, listening + UDS polling on 0x7E7 (BICM target)
- UDS poll cadence: ~10 Hz cycling through ~15 PIDs
- Cell voltages received via periodic frames `0x200-0x208` (12-bit × 0.00125 V scale, multiplexed)
- Module temperatures received via periodic frame `0x302`
- Telemetry sampled at 30 s aggregation
- Sign convention for `pack_current_A`: positive = current flowing into pack (charging), negative = current flowing out (discharging)
- Cell positions are 1-indexed (1..96)
- All timestamps in UTC

## License

[CC0 1.0 Universal](LICENSE) — public domain dedication. No attribution required (welcomed but not enforceable).

Observed physical battery measurements and raw GM UDS PID responses are factual data, not copyrightable.

## Provenance

Data collected by polling and recording the BICM responses on a Bolt pack used in a stationary storage application. Pack hardware verified on physical labels: GM P/N 24297220, manufacturing week 2212 (USA), BEV2/VITM platform, LG NMC chemistry, 10 cell modules totaling 66.8 kWh nominal (rated 64 kWh usable).

PID names follow conventions established by the [Battery-Emulator](https://github.com/dalathegreat/Battery-Emulator) project's Bolt module.

## Related work

- [Battery-Emulator (dalathegreat)](https://github.com/dalathegreat/Battery-Emulator)
- [Battery-Emulator Bolt module source](https://github.com/dalathegreat/Battery-Emulator/blob/main/Software/src/battery/BOLT-AMPERA-BATTERY.cpp)
- [Discussion #1848 — Ampera-e doesn't balance](https://github.com/dalathegreat/Battery-Emulator/discussions/1848)
- [openinverter forum: Bolt VITM/BECM K16 CAN](https://openinverter.org/forum/viewtopic.php?t=4370)
- [openinverter forum: Reverse-engineering Chevy Bolt BMS](https://openinverter.org/forum/viewtopic.php?t=95)
- [allev.info Bolt PIDs catalog](https://allev.info/boltpids/)

## Contributing

Issues welcome. Particularly helpful:

- Decoding any of the PID responses listed above
- Replication data from your own Bolt pack (stronger evidence with multiple data points)
- Pointing out errors in decoding or interpretation
- A J2534 capture from a working in-vehicle Bolt's BECM↔BICM bus (this is the unblocking missing piece for the entire community)

## Changelog

- 2026-05-11 — Added datasets 12 (raw 60 s CAN bus capture, 31,800 frames) + 13 (per-CAN-ID inventory summary). Provides actual frame-level data including periodic BalancingSwitches frames `0x270`/`0x272`/`0x274`.
- 2026-05-11 — Cleanup release: stripped all derived/processed columns, retaining only raw battery observations and raw UDS PID responses.
- 2026-05-10 — Initial release.
