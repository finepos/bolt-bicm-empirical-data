# Analysis & Open Questions

Companion notes to the raw datasets. **All claims here are interpretation and may be wrong** — this file is a starting point for community discussion, not authoritative.

## 1. Headline finding: standalone BICM does not balance cells

Across 4,113 polled responses to UDS PID `0x431C` (HYBRID_BATTERY_CELL_BALANCE_STATUS), every single sample returned `0` (idle). The polling window covered:

| Condition | Sample | Reading |
|---|---|---|
| Cell delta peak | 2026-05-09 11:10:31 | 150 mV |
| Cell min absolute | 2026-05-09 11:14 | 3,265 mV |
| Pack at active discharge | continuous | up to -23 A |
| Pack at active charge | continuous | up to +43 A |
| Pack at rest (current ±0.5 A) | continuous | yes |

If the BICM had any internal "voltage-triggered emergency balance" routine, the conditions on 2026-05-09 should have triggered it. They did not.

Conclusion: standalone BICM behaves as a sensor + reporter only. Balancing decisions live in the missing BECM master.

This empirically refutes any "maybe BICM balances at very low SOC" speculation in prior community threads.

## 2. Persistent positional cell extremes

The 49-day per-cell rank statistics in `06_per_cell_rank_stability.csv` show extreme concentration at a few cell positions:

- Position 12: at pack maximum 47.94 % of all 99,107 samples (avg 3,630 mV when at maximum)
- Position 49: at pack minimum 19.30 % of all samples (avg 3,544 mV when at minimum)
- Position 1: at both extremes ~17 % each (likely an edge cell with weakest thermal coupling)
- 84 of 96 cell positions are essentially never at extremes (< 1 %)

Hypothesis: positions 12 and 49 have fundamental capacity differences from manufacturing. Without active balancing, the highest-capacity cell sees the highest voltage during charge (full bucket) and the lowest-capacity cell sees the lowest during discharge (empty bucket first). In NMC chemistry this gradient self-perpetuates.

Implication for community: any solution that does not preferentially drain position 12 and/or charge position 49 will not address this pack's drift. The 0x20E "balance request" frame (per [Battery-Emulator TODO](https://github.com/dalathegreat/Battery-Emulator/blob/main/Software/src/battery/BOLT-AMPERA-BATTERY.cpp)) likely needs a 96-bit cell selection bitmask. The community-suspected `balanceMask[12]` byte array (96 bits) in [PR #2294](https://github.com/dalathegreat/Battery-Emulator/pull/2294) matches this hypothesis.

## 3. PID response anomalies

### `0x40D4` INTERNAL_CURRENT — does not track current

Range across 31 hours (including ±10 A operating cycles): `-11823` to `-11812` (only 8 distinct values out of 4,113 samples).

If this were instantaneous pack current as named, it would track the discharge/charge cycles visible in `pack_current_A`. It does not. Likely interpretations:

- Raw register value (16-bit signed) representing some internal sensor offset/baseline
- A "factory calibration zero" the BICM reports as a near-constant
- High-side or low-side leg of a differential current measurement
- An accumulator or counter rather than instantaneous reading

Decoding requires comparing values across packs (multi-pack dataset would help).

### `0x438B` HV_BUS_VOLTAGE — never responds

Polled at the same cadence as other PIDs. BICM never sends a response. Possible causes:

- PID number incorrect (typo in upstream documentation)
- Wrong service ID (we use 0x22 ReadDataByIdentifier; maybe needs different)
- Requires UDS Security Access (0x27 service) before responding
- Requires specific operating state we have not produced

### `0x428F` / `0x4290` 5V_REF_1 / 5V_REF_2 — wrong scaling?

Values 3001 / 3020 mV across 4,113 samples (only 2 distinct values). If these are 5 V references they should center around 2,500 mV (Uc/2 with `(raw × 5000) / 65535` scaling assumed). 3000 mV ≈ 60 % of 5 V which doesn't fit either.

Either:

- These PIDs are NOT 5 V references (mislabeled in upstream sources)
- Different scaling formula (raw value is voltage × 1000 directly, no 65535 division)
- The two values represent some toggle state (operating vs idle?)

### `0x40D3` 5V_REF_MAIN — looks correct

Value 4,809 mV — consistent with a 5 V supply rail measurement. So this PID semantics matches its name; only `0x428F` / `0x4290` are anomalous.

## 4. Operating regime characteristics from `09_cell_min_voltage_histogram.csv`

| cell_min bin | % of time | Avg delta | Max delta |
|---|---|---|---|
| 3,200–3,350 mV | 2.07 % | 102–135 mV | 150 mV |
| 3,350–3,500 mV | 3.81 % | 22–55 mV | 60–100 mV |
| 3,500–3,650 mV | 81 % | 16–22 mV | 29–42 mV |
| 3,650–3,950 mV | 9.4 % | 32–37 mV | 39–51 mV |

Key observation: cell delta is well-controlled (15–22 mV) when pack operates with cell min in the 3,500–3,650 mV band. Both extremes (lower or higher cell min) significantly elevate delta. This is consistent with NMC chemistry — the OCV curve flattens in mid-range so cell-to-cell capacity differences manifest less in voltage; but at edges (where dV/dSOC is steep), small Ah differences become large mV differences.

Operational implication: keeping pack between cell min 3,500–3,650 mV keeps cell drift manageable even without balancing.

## 5. The 2026-05-09 deep-discharge recovery event

`10_recovery_event_2026-05-09_trace.csv` captures 2 hours of recovery. Highlights:

- 11:00:16 — Start: cell min 3,270 mV (position 49), delta 148 mV, pack idle
- 11:30:00 — Charging starts, cell voltages climbing
- 12:30:00 — Cell min ~3,420 mV, delta down to ~80 mV
- 12:59:32 — End: cell min 3,489 mV (position 65), delta 35 mV

Notable observation: the lowest-voltage cell position **changes during recovery**. Position 49 was lowest at start; by end of charge cycle, position 65 (or 68 in nearby samples) became lowest. This suggests cells have different *charge acceptance rates*, not just different capacities — likely related to internal resistance differences.

The pack recovered without any active balancing — purely through the inherent property that lower-voltage cells accept more charge current per unit time (they're at lower SOC, on the steeper part of the OCV curve). This is **passive equalization through charge curve geometry**, not active balancing.

Why this matters: if a researcher is testing whether their candidate balance command "works", the test must control for this passive equalization. Just charging doesn't prove balancing happened.

## 6. Module thermal asymmetry (open question)

`07_module_temperatures_daily.csv` shows module 4 consistently 1–2 °C hotter than neighbors over 49 days. Possible causes:

- Module 4 physical position closer to cooling loop inlet/outlet
- Module 4 internal cell defect causing higher self-heating
- Sensor calibration offset for module 4 specifically

Without raw cell-by-module mapping, cannot determine if this thermal hotspot correlates with the consistently-lower position 49 or consistently-higher position 12.

## 7. What this dataset cannot prove

- Cannot rule out that BICM balances at even lower cell voltages than 3,265 mV (we never went lower).
- Cannot rule out that BICM balances when receiving some specific command we don't send.
- Cannot rule out that PID `0x431C` reports `0` even when balance IS happening (maybe the field is unimplemented in standalone mode).
- Cannot identify the BECM-to-BICM frame format — only a live in-vehicle CAN capture can.

## 8. Recommended community next steps

1. **Replicate** this dataset on different Bolt packs (different VINs, manufacturing dates) to confirm findings aren't pack-specific.

2. **Add `0x4323`-`0x4327` and `0x4340` (HYBRID_CELL_BALANCING_ID 1–6) polling** to existing data collection setups. Battery-Emulator users with the Bolt module can do this with minimal change. Publish those PID values.

3. **Tap a working Bolt's K16 X8 internal CAN** (pins 4/5 or 14/15 per [openinverter t=4370](https://openinverter.org/forum/viewtopic.php?t=4370)). Capture during a normal charge session at home. Look for traffic on `0x20E` and any related IDs. **This is the unblocking missing piece.**

4. **Test hypothesis** that frame `0x270` (BalancingSwitches) changes when BECM is present and cells need balancing. Requires raw CAN logging during in-vehicle operation.

5. **Cross-reference** the [Volt Gen1 protocol](https://www.diyelectriccar.com/threads/1st-gen-chevy-volt-bms-balance-commands-found.200023/) (Swoozle's research) for architectural patterns even though chemistry/wire protocol differs. The "queue cells in 0x300/0x310, trigger via 0x200#020000" pattern from Volt may have a Bolt analog.

## 9. What to do if BECM emulation cannot be cracked

For builders with surplus Bolt packs in stationary use today, the practical answer is **active balancer hardware** (NEEY/Heltec inductive equalizers). This bypasses BECM emulation entirely. Combined with operating in the 3,500–3,650 mV cell-min band, pack drift stays manageable.

This dataset proves *why* you need it: 49 days of natural operation produced positional cell extremes that won't self-correct.
