# Analysis & Interpretation Notes

Companion to the raw datasets. This document captures observations, hypotheses, and open questions the maintainer has noted while working with the data. **All claims here are interpretation and may be wrong** — this file is a starting point for community discussion, not authoritative.

## 1. The headline finding: BICM doesn't balance standalone

**Evidence**: 3,742 samples over 31 hours (Phase A monitoring window 2026-05-09 11:10 → 2026-05-10 18:26 UTC). Every single sample of UDS PID 0x431C returns `0` (idle). Conditions covered include:

| Condition | Sample | Reading |
|---|---|---|
| Cell delta peak | 2026-05-09 11:10:31 | 150 mV |
| Cell min absolute | 2026-05-09 11:14 | 3,265 mV |
| SOC at min | 2026-05-09 11:10 | 0.3% |
| Active discharge | continuous | up to -23 A |
| Active charge | continuous | up to +43 A |
| Idle (current ±0.5 A) | continuous | yes |

If the BICM had any internal "voltage-triggered emergency balance" routine, conditions on 2026-05-09 should have triggered it. They did not.

**Conclusion**: standalone BICM behaves as a pure sensor + reporter. All balancing decisions live in the missing BECM master.

This empirically refutes any "maybe BICM balances at very low SOC" speculation in prior community threads.

## 2. Persistent weak/strong cells

The 49-day cell rank stability data (`06_per_cell_rank_stability.csv`) shows extreme concentration:

- **Cell #12** is at maximum 48% of all 99,107 samples. Strongest cell.
- **Cell #49** is at minimum 19% of all samples. Weakest cell.
- **Cell #1** is at both extremes ~17% — pack edge cell, behaves erratically.
- 84 of 96 cells are essentially never at extremes (`AVERAGE` classification).

**Hypothesis**: cells #12 and #49 have fundamental capacity differences from manufacturing. Without active balancing, the strongest cell sees highest voltage during charge (full bucket) and the weakest sees lowest during discharge (empty bucket first). In NMC chemistry this gradient self-perpetuates: strong cell stays strong because it's at higher SOC; weak cell stays weak because at lower SOC.

**Implication for community**: any solution that does NOT preferentially drain cell #12 and/or charge cell #49 will not solve this pack's drift. The 0x20E "balance request" frame (if cracked) likely needs a 96-bit cell selection bitmask — and the community-suspected `balanceMask[12]` byte array (96 bits) in [PR #2294](https://github.com/dalathegreat/Battery-Emulator/pull/2294) matches this hypothesis.

## 3. PID anomalies

### 0x40D4 INTERNAL_CURRENT — never instantaneous current

Range across 31 h (including ±10 A operation): **-11823 to -11812** (8 distinct values out of 3,742 samples).

If this were instantaneous pack current as named, it would track the discharge/charge cycles visible in `pack_current_A`. It does not. **Likely interpretations**:

- Raw register value (16-bit signed) representing some internal sensor offset/baseline
- Possibly a "factory calibration zero" that the BICM reports as a constant
- Possibly the high-side or low-side leg of a differential current measurement

Decoding this requires comparing values across packs (multi-pack dataset would help).

### 0x438B HV_BUS_VOLTAGE — silence

Polled but BICM never sends a response. Multiple possible causes:

- PID number incorrect (typo in Battery-Emulator source?)
- Wrong service ID (we use 0x22 ReadDataByIdentifier; maybe needs different)
- Requires UDS Security Access (0x27 service) before responding
- Requires specific operating state (HV active, contactors closed) — but we polled during HV active too

### 0x428F / 0x4290 5V_REF_1 / 5V_REF_2 — wrong scaling?

Values 3001 / 3020 mV across 3,742 samples (only 2 distinct values). If these are 5V references they should center around 2,500 mV (5V × 50% with `(raw × 5000) / 65535` scaling we assume).

3000 mV = 60% of 5V. Either:

- These PIDs are NOT 5V references (mislabeled in Battery-Emulator)
- Different scaling formula (e.g., raw value is voltage × 1000 directly, no 65535 division)
- The two values 3001 vs 3020 represent some toggle state (operating vs idle?)

### 0x40D3 5V_REF_MAIN — looks correct

Value 4809 mV — close to expected 5V supply rail. So this PID is consistent with the name; only 0x428F/0x4290 are anomalous.

## 4. Operating regime characteristics

From `09_soc_histogram_with_delta.csv`:

| SOC band | % of time | Avg delta mV | Max delta mV |
|---|---|---|---|
| 0-5% | 2.11% | 127.7 | 150 |
| 5-15% | 0.74% | 38.9 | 150 |
| 15-30% | 70.67% | 21.3 | 93 |
| 30-90% | 19.51% | 22.5 | 51 |
| 90-100% | 3.83% | 38.9 | 50 |

**Key insight**: cell delta is well-behaved (15-25 mV) when pack operates in 15-90% SOC band. Both extremes (deep discharge or full charge) significantly elevate delta. This is consistent with NMC chemistry behavior — the SOC curve flattens in mid-range so cell-to-cell capacity differences manifest less in voltage; but at edges (where dV/dSOC is steep), small Ah differences become large mV differences.

**Operational implication**: keeping pack between 15-90% SOC keeps cell drift manageable EVEN WITHOUT balancing. The Bolt OEM operates 15-90% in the vehicle for the same reason.

## 5. The 2026-05-09 deep discharge event

`10_charge_recovery_cycle_2026-05-09.csv` captures 2 hours of recovery from this event. Highlights:

- **11:00:16** — Start: SOC 0.3%, cell min 3,270 mV (cell #49), delta 148 mV, no current flow
- **11:30:00** — Charging starts, cell voltages climbing
- **12:30:00** — SOC 5%, delta down to 80 mV
- **12:59:32** — End: SOC 10.1%, cell min 3,489 mV (cell #65), delta 35 mV

**Interesting observation**: the weakest cell identity changes during recovery. Cell #49 was lowest at start; by end of charge cycle, cell #65/68 became lowest. This suggests cells have different *charge acceptance rates* not just different capacities — likely related to internal resistance differences.

The pack recovered without any active balancing — purely through the inherent property that lower-voltage cells accept more charge current per unit time (they're at lower SOC, on the steeper part of the voltage curve). This is **passive equalization through charge curve geometry**, not active balancing.

**Why this matters for BECM emulation**: if a researcher is testing whether their candidate balance command "works", the test must control for this passive equalization. Just charging doesn't prove balancing happened.

## 6. Module thermal asymmetry (open question)

`07_module_temperatures_daily.csv` shows module #4 consistently 1-2°C hotter than neighbors over 49 days. Possible causes:

- Module #4 physical position closer to cooling loop inlet/outlet?
- Module #4 internal cell defect causing higher self-heating?
- Sensor calibration offset for module #4 specifically?

**Without raw cell-by-module mapping**, can't determine if this thermal hotspot correlates with the weak cell #49 or strong cell #12. If cells per module = 16 (96 cells / 6 modules — but that doesn't work for 96/10 modules from manufacturer data), the mapping is unclear.

Battery-Emulator may have decoded the cell-to-module mapping; cross-reference welcomed.

## 7. What this dataset cannot prove

- **It cannot rule out** that BICM balances at **even lower SOC** than 0.3% (we never went lower).
- **It cannot rule out** that BICM balances when receiving some specific command we don't send.
- **It cannot rule out** that PID 0x431C reports `0` even when balance IS happening (maybe the field is read-only and unimplemented).
- **It cannot identify** the BECM-to-BICM frame format — only a live in-vehicle CAN capture can.

## 8. Recommended community next steps

1. **Replicate** this dataset on different Bolt packs (different VINs, manufacturing dates) to confirm findings aren't pack-specific.

2. **Add 0x4323-0x4327 + 0x4340 (HYBRID_CELL_BALANCING_ID 1-6) polling** to existing data collection setups. Battery-Emulator users running the Bolt module can do this with minimal firmware change. Publish those PID values.

3. **Tap a working Bolt's K16 X8 internal CAN** (pins 4/5 or 14/15 per openinverter t=4370). Capture during a normal charge session at home. Look for traffic on 0x20E and any related IDs. **This is the unblocking missing piece.**

4. **Test hypothesis** that 0x270 BalancingSwitches frame changes when BECM is present and cells need balancing. Requires raw CAN logging during in-vehicle operation.

5. **Cross-reference Volt Gen1 protocol** (Swoozle's research) for architectural patterns even though chemistry/wire protocol differs. The "queue cells in 0x300/0x310, trigger via 0x200#020000" pattern from Volt may have a Bolt analog.

## 9. What to do if you can't crack BECM

For builders with surplus Bolt packs in stationary use today, the practical answer is **active balancer hardware** (NEEY/Heltec inductive equalizers, ~$200-400 for 96S coverage). This bypasses BECM emulation entirely. Combined with operating discipline (15-90% SOC band), pack drift stays manageable.

This dataset proves *why* you need it: 49 days of natural operation produced same-cell extremes that won't self-correct.
