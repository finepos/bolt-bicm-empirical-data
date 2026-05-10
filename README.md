# Empirical Bolt EV Pack Data — Standalone (No-BECM) Operation

**Public empirical dataset for Chevrolet Bolt EV battery (LG N2.2 NMC, 64 kWh, 96S3P) operated standalone — without vehicle BECM master.**

49 days of continuous monitoring including UDS PID polling, per-cell voltage tracking, and rare deep-discharge events. Goal: provide raw observable data to the open-source community working on Bolt BMS reverse engineering and BECM emulation, particularly for cell balancing.

## TL;DR for the BECM emulation researcher

1. **Standalone BICM never balances cells.** PID 0x431C (`HYBRID_BATTERY_CELL_BALANCE_STATUS`) reports `0` (idle) across **3,742 samples / 31 hours** including extreme conditions (cell delta 150 mV, cell min 3.265 V at SOC 0.3%). Confirms "BECM master required" hypothesis.

2. **Same cells stay extreme long-term.** Over 99,000 samples (49 days):
   - Cell #12: at maximum **48% of all samples** (avg 3,630 mV when max)
   - Cell #49: at minimum **19% of all samples** (avg 3,544 mV when min)
   - Cell #1: edge cell, both min and max equally
   - Without active balancing, these "weak/strong" identities persist indefinitely.

3. **Several documented PIDs return unexpected values:**
   - `0x438B` HV_BUS_VOLTAGE — **never responds** (expected: byte value)
   - `0x428F`, `0x4290` 5V_REF_1/2 — narrow range 3001-3020 mV (expected ~2500 mV at Uc/2)
   - `0x40D4` INTERNAL_CURRENT — narrow -11823..-11812 across all states including active discharge (NOT instantaneous current as named)
   - `0x40D3` 5V_REF_MAIN — returns 4809 mV (looks correct as ~5V supply)

4. **Pack operating distribution** (49-day SOC histogram): 63% of time at 20-30% SOC, 7% at 15% SOC (Deye cutoff), 2% at 0% SOC (deep discharge incidents). Delta tracks SOC: ~17 mV at 20-30% SOC, but jumps to 30-128 mV below 15% SOC and 38-48 mV above 90% SOC — both extremes stress balance.

## Pack identification

| Field | Value |
|---|---|
| Source vehicle | Chevrolet Bolt EV 2022 |
| Manufacturer P/N | GM 24297220 |
| Manufacturing date | Week 12, 2022 (USA) |
| Platform | BEV2 / VITM |
| Chemistry | LG N2.2 (NMC, LiMM-C.F) |
| Topology | 96S3P (96 series, 3 parallel) |
| Nominal capacity | 64 kWh / ~180 Ah |
| Nominal voltage | ~350 V |
| Cell module count | 10 modules × 6.68 kWh nominal |
| BMS | OEM BICM/VITM |

## Operating context

- **Standalone**: no BECM in circuit. BICM powered via external 12 V supply.
- **Application**: stationary inverter/storage, Deye SUN-80K hybrid
- **Internal CAN**: 500 kbps, listening + UDS polling on 0x7E7
- **No vehicle harness** — only HV+/HV-, BMS data CAN, BMS power 12 V, contactor control
- **Monitoring period**: 2026-03-23 → 2026-05-10 UTC (49 days)
- **Telemetry cadence**: 30 s aggregation
- **Deep monitoring cadence (Phase A PIDs)**: from 2026-05-09 11:10 UTC

## Datasets

### `01_bolt_bicm_balance_log.csv` — Headline finding (3,742 rows, 31 h)
Time series of PID 0x431C status alongside cell statistics, SOC, current, temperature.
Period: 2026-05-09 11:10 → 2026-05-10 18:26 UTC.
Includes the 2026-05-09 deep discharge event (SOC 0.3%, cell min 3.270 V, delta 150 mV) — proves standalone BICM doesn't react.

### `02_bolt_uds_pid_responses.csv` — UDS PID raw responses (3,742 rows)
Time series of all Phase A polled PIDs (0x431C, 0xC218, 0x40D4, 0x438B, 0x40D3, 0x428F, 0x4290, 0x8043) with cross-context fields. **Many PIDs return unexpected values — community decoding needed.** See ANALYSIS.md.

### `03_bolt_extreme_events.csv` — Worst-case events (1,980 rows, 49 days)
Every telemetry sample where `cell_delta_mV >= 100` OR `cell_min_mV <= 3300`. Most rows pre-date Phase A monitoring (PID 0x431C field absent). Useful for understanding what conditions produce the imbalance.

### `04_cell_voltage_snapshots.csv` — 96-cell array snapshots (5 events)
Full per-cell voltage arrays at five representative operating points: deep_discharge, peak_delta, idle_mid_soc, active_charge, active_discharge.

### `05_daily_cell_drift_evolution.csv` — Long-term drift (40 days)
Daily aggregate showing how `cell_voltage_delta` evolves over 49-day window. Includes peak/min/stdev delta per day, peak charge/discharge currents, average temperatures. Shows pack operating regime evolution.

### `06_per_cell_rank_stability.csv` — Per-cell statistics (96 rows)
For each of 96 cells: how often it was the minimum/maximum across all 99,000+ samples, average voltage when at extreme, and a classification (`STRONG`/`WEAK`/`MIXED`/`AVERAGE`). **Definitive empirical evidence that same cells stay weak/strong without active balancing.**

### `07_module_temperatures_daily.csv` — Module temperature trend (40 days)
Per-module temperature averages (6 module probes). Useful for thermal asymmetry analysis and confirming temperature is not the primary cause of cell drift.

### `08_bms_data_field_inventory.csv` — Field catalog (69 fields)
Inventory of every distinct field name seen in the BMS telemetry stream over 49 days, with sample counts. Reference for community to know what fields are populated and how often.

### `09_soc_histogram_with_delta.csv` — Operating range distribution (21 bins)
SOC histogram (5%-wide bins) with average and max cell delta per bin. Shows where pack actually operates and how cell imbalance correlates with SOC band.

### `10_charge_recovery_cycle_2026-05-09.csv` — Recovery event trace (242 rows, 2 h)
2-hour high-resolution trace of charge recovery from SOC 0.3% (cell min 3.270 V, delta 148 mV) to SOC 10.1% (cell min 3.488 V, delta 35 mV). Shows cell rank changes during recovery — cell #49 is weakest at start, cell #68 weakest at end.

### `11_hourly_trend_49days.csv` — Hourly aggregate (835 hours)
Hourly aggregates of SOC, current, cell voltages, delta, temperature for the full 49-day window. Plottable as time series for visualizing pack natural behavior.

## What the community can dig into

### High-priority unanswered questions

1. **0x4323-0x4327 + 0x4340 (HYBRID_CELL_BALANCING_ID 1-6)** — Battery-Emulator defines them as poll PIDs but no public interpretation. Our data doesn't poll these yet — adding them is straightforward firmware change for any researcher with similar hardware.

2. **0x270 BalancingSwitches diagnostic frame** — codebase comment says "Battery VoltageSensor BalancingSwitches diagnostic status" but never decoded. Did this frame change behavior during our extreme delta events? (Need raw CAN capture, not in this dataset.)

3. **0x40D4 internal_current** — narrow -11823..-11812 across all conditions including ±10 A discharge. **What does this PID actually return?** Likely a register value, not current.

4. **0x438B HV_BUS_VOLTAGE** — why never responds? Wrong PID number? Wrong service ID byte? Requires UDS Security Access (0x27) preconditions?

5. **0x428F / 0x4290 5V refs scaling** — values 3001-3020 mV don't match expected ~2500 mV (Uc/2). Either different scaling formula, or these are NOT 5V references but something else mislabeled.

6. **0x20E "Hybrid balancing request HV"** — Battery-Emulator TODO. Has anyone captured this from a working in-vehicle BECM? Can someone with a Bolt EUV (not the recalled pack) add a J2534 logger on the K16 X8 internal CAN tap (pins 4/5 or 14/15) and capture during charge?

### Cell-specific analysis opportunities

- **Cell #12** (47% of samples max) and **cell #49** (19% min) — what's the underlying cause? Module position? Cell-level manufacturing defect?
- **Cell #1** (mixed min/max) — likely edge cell with weakest thermal coupling.
- Per-cell drift rate over 49 days — calculable from `04_cell_voltage_snapshots.csv` if more snapshots are extracted.

### Cross-validation opportunities

- Compare our `bolt_cell_avg_mV` (PID 0xC218) vs computed average from cell array — confirms or refutes the BICM cell averaging algorithm.
- Sanity check `0x216` sensed_current vs Deye-reported pack current.
- Validate cell numbering convention (1-indexed, matches Battery-Emulator?)

## Methodology

- **Hardware**: ESP32-S3 with MCP2518FD CAN-FD controller
- **CAN bus**: 500 kbps, listening + UDS polling on 0x7E7 (BICM target), 0x7E4 not polled
- **UDS poll cadence**: ~10 Hz cycling through ~15 active PIDs
- **Periodic frames**: received at native rate (100 ms cadence for 0x200-0x216)
- **Telemetry storage**: PostgreSQL JSONB at 30 s aggregation
- **Cell voltage measurement source**: BICM-published mV via 0x200-0x208 frame multiplex (12-bit × 0.00125 V scale)
- **Sign convention**: pack_current > 0 = charging, < 0 = discharging
- **Pack utilization**: ~10% idle, ~70% active discharge, ~20% charging
- **Anonymization**: all device/server identifiers stripped (CSVs contain only physical measurements)

## License

[CC0 1.0 Universal](LICENSE) — public domain, no attribution required.

Observed physical measurements are factual data and generally not copyrightable. CC0 dedication provides legal certainty for jurisdictions with sui generis database protection.

## Provenance

Data extracted from production deployment of a custom BMS-to-inverter bridge running on ESP32-S3 hardware. Methodology and field naming conventions follow [Battery-Emulator](https://github.com/dalathegreat/Battery-Emulator)'s `BOLT-AMPERA-BATTERY.cpp` module.

Pack hardware verified on physical labels: GM P/N 24297220, manufacturing week 2212, BEV2/VITM platform, LG NMC LiMM-C.F chemistry. 10 cell modules × 6.68 kWh nominal = 66.8 kWh installed (rated 64 kWh usable).

The source firmware powering data collection is proprietary, but additional **observable data** can be extracted on request — open an issue.

## Related work and references

- [Battery-Emulator (dalathegreat)](https://github.com/dalathegreat/Battery-Emulator) — the canonical open-source BMS-to-inverter bridge
- [Battery-Emulator Bolt module source](https://github.com/dalathegreat/Battery-Emulator/blob/main/Software/src/battery/BOLT-AMPERA-BATTERY.cpp)
- [Discussion #1848 — Ampera-e doesn't balance](https://github.com/dalathegreat/Battery-Emulator/discussions/1848)
- [PR #2294 — Failed experimental balancing attempt (May 2026)](https://github.com/dalathegreat/Battery-Emulator/pull/2294)
- [openinverter forum: Bolt VITM/BECM K16 CAN](https://openinverter.org/forum/viewtopic.php?t=4370)
- [openinverter forum: Reverse-engineering Chevy Bolt BMS](https://openinverter.org/forum/viewtopic.php?t=95)
- [allev.info Bolt PIDs catalog (~240 PIDs)](https://allev.info/boltpids/)
- Volt Gen1 reference (different chemistry, similar architecture):
  - [Swoozle's BMS balance commands writeup](https://www.diyelectriccar.com/threads/1st-gen-chevy-volt-bms-balance-commands-found.200023/)
  - [Tom de Bree SimpBMS for Volt](https://github.com/tomdebree/SimpBMS)
  - [Damien Maguire AmperaBattery](https://github.com/damienmaguire/AmperaBattery)
- [chevybolt.org cell balancing thread (Telek's flyback inference)](https://www.chevybolt.org/threads/cell-balancing.35925/)

## Contributing

Issues and pull requests welcome. Specifically helpful:

- **Decoding ANY of the unknown PID responses** — open an issue with what you found
- **Pointing out errors** in this dataset (e.g., misinterpreted scaling, wrong field labels)
- **Replication data** from your own Bolt pack — additional data points strengthen findings
- **Anyone with vehicle access** — a J2534 capture of BECM↔BICM traffic during charge would unblock the entire community. Low-cost commercial bounty option: see [BECM_BOUNTY.md](BECM_BOUNTY.md) for ideas (TBD).

## Changelog

- **2026-05-10** — Initial release. 11 datasets covering 49 days of operation including the 2026-05-09 deep discharge event.
