# YamBMS - Release note about new YamBMS 1.7.1 Cut-Off charging logic

[![Badge License: GPLv3](https://img.shields.io/badge/License-GPLv3-brightgreen.svg)](https://www.gnu.org/licenses/gpl-3.0)
[![Badge Version](https://img.shields.io/github/v/release/Sleeper85/esphome-yambms?include_prereleases&color=yellow&logo=DocuSign&logoColor=white)](https://github.com/Sleeper85/esphome-yambms/releases/latest)
![GitHub stars](https://img.shields.io/github/stars/Sleeper85/esphome-yambms)
![GitHub forks](https://img.shields.io/github/forks/Sleeper85/esphome-yambms)
![GitHub watchers](https://img.shields.io/github/watchers/Sleeper85/esphome-yambms)

## Fix: false End-Of-Charge (EOC) detection on low PV production days

### The problem

The Cut-Off detection used a current-compensated voltage criterion (`max_cell_v ≥ cv_min + R·I`). At low charge current, the threshold converges toward `cv_min` (3.37 V/cell on LFP). The algorithm could therefore not distinguish between a **genuine current taper** (battery full, charger voltage-limited) and **PV starvation** (overcast weather, charger production-limited).

On grey days, a typical failure sequence was: a short sun burst raises cell voltages, a cloud passage drops the current to 1–2 A while a high runner cell or surface charge keeps `max_cell_v` above 3.37 V for a minute or two — the 10 s validation plus the 60 s cut-off timer (~70 s total) were enough to declare EOC at a real SoC of 85–92 %. The battery was then parked in Float/EOC (charging effectively stopped), the `rebulk_days` counter was wrongly reset, and recovery only occurred once SoC dropped back below the rebulk threshold — often losing the rest of the charging day.

### What was added

**1. "CVL reached" latch — the core fix** A current taper is only meaningful as an `EOC indicator` when the charger is voltage-limited (CV phase). Cut-Off can now only be armed once, per charge cycle, the pack has actually reached the requested bulk voltage (`total_voltage ≥ bulk_voltage − margin`), or alternatively once `max_cell_v` comes close to the cell max charge voltage — this second condition covers the runner-cell case where the pack voltage never reaches the bulk setpoint. On an overcast day the CVL is never reached, so no EOC can be declared regardless of how long the current stays low.

The latch is released at two points, each covering a distinct case:
- **When entering the Bulk phase** — prevents a stale latch *within* a cycle (e.g. CVL reached in the morning, heavy loads pull the pack down for hours, then a grey late afternoon with a residual trickle current would otherwise still validate a false EOC).
- **In the EOC script** — prevents a stale latch *between* cycles. Without this, a rebulk triggered by `rebulk_days` or a force-charge request could re-enter Cut-Off immediately (the leftover latch plus a trickle current would re-declare EOC within ~70 s), never actually recharging — defeating the very purpose of the periodic full charge (SoC resync and balancing).

**2. Minimum SoC gate** Cut-Off/EOC now additionally requires `SoC ≥ 95 %` (configurable). A simple safety net against premature EOC caused by a single imbalanced runner cell.

**3. Current deadband around 0 A** Previously, `current >= 0` treated exactly 0 A and sensor noise as "charging". Currents with `I` below 0.005C (configurable, ≈1.4 A on a 280 Ah pack) are now treated as a grey zone: no conclusion is drawn, both phase timers are reset, and the charge status is left unchanged. This neutralizes both current-sensor noise around zero and the residual trickle current of overcast days.

**4. Discharge cell hysteresis voltage** The discharge branch now uses an explicit `cell_hysteresis_v` independent of the capacity (20 mV/cell by default, corresponding to the previous behavior of a 280 Ah pack).

### New substitutions

| Substitution | Default | Purpose |
|---|---|---|
| `yambms_eoc_min_soc` | `95` | Minimum SoC (Max 98%) required to allow Cut-Off/EOC |
| `yambms_cvl_reached_margin_v` | `0.010` | Margin (V/cell) below bulk voltage to consider the CVL reached |
| `yambms_cell_knee_margin_v` | `0.10` | Alternative gate: `max_cell_v ≥ cv_max −` this margin (runner / Auto CVL case) |
| `yambms_current_deadband_crate` | `0.005` | C-Rate below which the current is treated as noise / PV starvation |
| `yambms_cell_hysteresis_v` | `0.020` | Discharge hysteresis margin (V/cell) |

### Expected behavior after the fix

- Overcast day, CVL never reached → Cut-Off never armed, status simply alternates between Bulk and grey zone. No false EOC possible.
- Normal sunny end of charge → unchanged behavior: pack reaches CVL, current tapers, Cut-Off → EOC as before.
- Periodic full charge (`rebulk_days`, force charge) → now guaranteed to perform an actual full charge cycle before re-declaring EOC.
