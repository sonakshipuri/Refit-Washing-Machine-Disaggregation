# DATA.md

Raw-data coverage, quality findings, cleaning steps, gap/outlier decisions and remaining concerns. All numbers are from `notebook.ipynb`. Washing-machine results are in `RESULTS.md`.

## 1. Data used

| Task | Source | Files | Resolution |
|---|---|---|---|
| 1. Cleaning | Raw REFIT | House 1: `RAW_House1_Part1.csv`, `RAW_House1_Part2.csv` | ~8 s, resampled to 1 min |
| 2-3. EDA, model | Official cleaned REFIT | `CLEAN_House{h}.csv`, 19 houses | 8 s, resampled to 1 min means |

- REFIT, University of Strathclyde: 20 UK homes (numbered 1-21, no House 14), aggregate plus 9 appliance monitors each. Units: watts, average real power.
- Washing-machine channel exists in 19 of 20 homes (House 12 has none). House 4 sums two machines (A4+A5): used in EDA, excluded from the model. House 13's channel changes on 25 Mar 2015, so its data is cut at 2015-03-24.
- Raw `Time` equals `Unix` read as UTC in 100% of rows (47.8% as UK local). The cleaned files have a clock problem (section 6).

### Coverage per home (cleaned data, 1-min)

| House | First | Last | House | First | Last |
|---|---|---|---|---|---|
| 1 | 2013-10-09 | 2015-07-10 | 10 | 2013-11-20 | 2015-06-30 |
| 2 | 2013-09-17 | 2015-05-28 | 11 | 2014-06-03 | 2015-06-30 |
| 3 | 2013-09-25 | 2015-06-02 | 13 | 2014-01-17 | 2015-03-24 |
| 4 | 2013-10-11 | 2015-07-07 | 15 | 2013-12-17 | 2015-07-08 |
| 5 | 2013-09-26 | 2015-07-06 | 16 | 2014-01-10 | 2015-07-08 |
| 6 | 2013-11-28 | 2015-06-28 | 17 | 2014-03-06 | 2015-06-19 |
| 7 | 2013-11-01 | 2015-07-08 | 18 | 2014-03-07 | 2015-05-22 |
| 8 | 2013-11-01 | 2015-05-10 | 19 | 2014-03-06 | 2015-06-20 |
| 9 | 2013-12-17 | 2015-07-08 | 20 | 2014-03-20 | 2015-06-23 |
| | | | 21 | 2014-03-07 | 2015-07-07 |

Per-home no-data %, usable days, off time, cycle counts and flags are in `RESULTS.md` section 1.

## 2. Raw House 1: findings

- **Two files overlap** from 2014-06-27 to 2014-10-01 and agree (mean aggregate 354.6 vs 354.9 W; median absolute difference 1.3 W). Part1 marks a missing sensor as 0, Part2 as NaN. All 5,284,083 NaN cells are in Part2. In Part1 a zero can mean "off" or "offline".
- **Ordering:** 1 out-of-order row. All timestamps parse.
- **Duplicates:** 597,182 rows share a `Unix` second; 508,829 are exact copies and 88,353 conflict. For conflicting seconds the aggregate differs by a median of 3 W (95th percentile 68 W); the washing machine by a median of 38 W (95th percentile 213 W).
- **Intervals:** median 7 s. Before July 2014, 30-47% of intervals exceed 10 s each month; from July 2014, 0-3%.
- **Gaps:** 25 gaps over 1 hour, 1,905 h in total. Largest: 998 h (2014-01-24 to 2014-03-06).
- **Impossible/extreme values:** no negatives. Aggregate: 12 exact zeros, maximum 29,159 W, 69 readings above 23 kW (about 100 A at 230 V, the largest normal UK supply), 18,019 above 10 kW. Appliance maxima reach 98,301 W and 65,535 W (the heater; 2^16 - 1), and 2,889 readings exceed 4,000 W.
- **Appliances summing above the aggregate:** 55,700 raw rows.

## 3. Cleaning pipeline (House 1)

| Step | Rule | Effect |
|---|---|---|
| Join | Part1 before the first Part2 timestamp, Part2 after | 7,260,898 rows (1,257,865 Part1 rows dropped) |
| Sort | By `Unix`, stable | |
| Duplicates | One row per second; keep last non-missing value per channel | 597,182 rows removed |
| All-channel zeros | Set to NaN | 0 rows |
| Aggregate = 0 | Set to NaN | 8 rows |
| Aggregate > 23 kW | Set to NaN | 62 rows |
| Isolated aggregate spike > 10 kW | Both neighbours below 30% of it: NaN | 8 rows |
| Aggregate 10-23 kW, not isolated | Kept, flagged `agg_extreme` | 16,513 rows (1,689 min) |
| Appliance > 4,000 W | Set to 0 (official REFIT rule) | 2,591 readings |
| Resample | 1-min mean | 920,031 min |
| Short gaps | 5 min or less: time interpolation, between valid readings only | 888 aggregate min filled |
| Long gaps | Over 5 min: aggregate flagged `outage`, left NaN | 382 outages |
| `Issues` flag | Appliances sum exceeds aggregate by more than 10 W | 5,893 min (1.0%) |

Why:
- **Last non-missing value per channel**, not last row: Part2 rows are partial, so keeping the last row loses readings.
- **5-minute fill limit:** longer gaps would invent events. The limit is my choice. Only 374 gaps fall between 5 minutes and 1 hour.
- **Long gaps are masked back to NaN.** `interpolate(limit=5)` alone would fill the first 5 minutes of every long gap.
- **Appliance zeros are kept:** off and offline cannot be told apart.
- **10 kW and 23 kW ceilings:** 23 kW has a physical basis; the 10 kW isolated-spike rule is my own.
- **Tasks 2-4 use the official cleaned data,** as the brief asks.

Result: 382 outages, 1,980 h, 12.9% of minutes. Three outages (Jan-Mar 2014, May 2015, Jun 2015) account for 1,386 h. The electric heater has the most missing minutes (36.7%); other channels 13.0-14.1%. The `Issues` rate falls from 2.2% (2013Q4) to 0.2-0.6% after mid-2014.

Plots (`outputs/figures/`): `task1_before_after_spike_day.png`, `task1_before_after_wm_day.png`, `task1_before_after_full_period.png` (aggregate, fridge, washing machine). The 4,000 W rule removes the spikes; 1-minute averaging alone would not. Raw and cleaned daily means agree except on spike days.

## 4. Cleaned data quality (19 homes)

- 5.9% (House 18) to 24.6% (House 2) of minutes have no reading.
- Washing machine: exactly 0 W for 89.1% of valid minutes, 0-20 W for 6.9%, above 20 W for 4.0%.
- Minimum-data gate: 180 usable days and 50 usable cycles (a day is usable with at least 90% valid minutes). All 19 pass.

| Flag | Rule | Homes |
|---|---|---|
| gappy | over 20% of minutes without a reading | 2, 13 |
| stale | aggregate unchanged 30+ min, over 10% of minutes | 3 (13.3%) |
| truncated | over 25% of cycles touch a gap or data edge | 13 |
| long | over 5% of cycles exceed 240 min | 5 |
| issues | over 10% of cycles overlap `Issues` minutes | 5, 11, 21 |

Homes are flagged, not dropped. Three of the four test homes (2, 13, 21) are flagged.

## 5. Decisions on gaps and outliers

- A minute is **valid** only if the aggregate and washing-machine channel both have a reading. Training and scoring use valid minutes only.
- **Cycles are never bridged over a gap.** Pauses up to 10 min are merged only if fully observed. Cycles touching a gap or the first/last day are `truncated` and excluded from duration and energy statistics.
- **A missing day is not "no washing".**
- **`Issues` minutes:** the main model results exclude them. A utility has no sub-meters to build this flag, so a check scores all valid minutes: F1 stays 0.36, energy bias moves from +1.3% to -9.4%. The flag touches 36.6% of cycles by at least one minute but only 5.7% by over a quarter of their minutes; dropping them changes a home's median cycle energy by at most 1.7% (House 7), so they are kept.
- **No extra outlier removal on the cleaned data.** Cycles above 2 kWh (7 in House 5, 8 in House 8, 6 in House 11, 1 each in Houses 15 and 21) may be a washer-dryer or mixed channel. They are kept.
- **Cycle threshold (20 W):** the power distribution has no clean gap, so it was checked by sensitivity. Thresholds of 10-50 W and merge gaps of 5-20 min change the total cycle count by under 5%.

## 6. Remaining concerns

1. **Clock in the cleaned files.** For House 1, the cleaned series is 60 minutes earlier than raw (UTC) in Oct 2013 and Apr-Sep 2014, and matches it otherwise. Two daylight-saving checks disagree: spring 2014 says UTC (14 of 14 homes), autumn 2014 says local (16 of 16); spring 2015 had too little data. I used the stamps as local time. Hour-of-day and peak-window results can be off by an hour or more. Only House 1 was checked against raw. A Nov-Mar-only check gives similar conclusions, and dropping clock features lowers model F1 from 0.36 to 0.32.
2. **Raw cleaning covers House 1 only.** Other homes were not checked for the same duplicate and Part1/Part2 patterns.
3. **Judgement thresholds are untuned:** 5 min fill limit, 10 kW and 23 kW ceilings, 4,000 W cap, 20 W / 10 min cycle rule, 30 min stale rule, flag cut-offs.
4. **Uneven early sampling:** before July 2014, a 1-minute mean is built from fewer, unevenly spaced readings.
5. **Appliance labels** come from the README and were not checked against the signals.
6. **Narrow sample:** 20 mostly detached homes in one English area, all covering about the same 2013-15 window.
