# RESULTS.md

Washing-machine EDA (Task 2) and appliance-disaggregation model (Task 3), using the **official cleaned REFIT data** at **1-minute resolution**. All numbers come from `notebook.ipynb`; figures are in `outputs/figures/` and tables in `outputs/tables/`.

**Headline.** A gradient-boosting model that sees only the household aggregate beats every baseline on four unseen homes (NDE 0.63 vs 1.00 for always-off, F1 0.36 vs 0.11 for the usage profile), but absolute accuracy is modest: only about 1 in 3 predicted "washing" minutes is real, and the energy of a wash that *is* found is under-estimated by about half. It is useful for cohort-level patterns, not for per-wash or per-bill claims (see `Recommendation.md`).

---

## 1. Data used

- **Source:** cleaned REFIT (University of Strathclyde), 20 UK homes, 2013-09 to 2015-07, 8-second readings resampled to 1-minute means, power in watts.
- **Homes with a washing-machine channel:** 19 of 20 (House 12 has none). All 19 passed the minimum-data gate (at least 180 usable days and 50 usable cycles). Weaker homes are **flagged, not dropped**.
- **House 4** has two washing machines summed in one channel (A4+A5). It is used in EDA but **left out of the model**.
- Period per home is in the table below (first to last day, usable days = days with enough valid data).

### Availability and data quality per home

`no data` = share of minutes with no reading. `off` = share of valid minutes at or below 20 W. Cycle statistics use usable cycles (not truncated, not over 240 min).

| House | Span (days) | Usable days | No data % | WM off % | Cycles | Usable cycles | Cycles / week | Median duration (min) | Median energy (kWh) | Heated % | Flags |
|---|---|---|---|---|---|---|---|---|---|---|---|
| 1 | 639 | 548 | 12.6 | 98.1 | 396 | 376 | 5.0 | 33 | 0.32 | 85 | - |
| 2 | 617 | 441 | 24.6 | 95.3 | 336 | 303 | 5.1 | 97 | 0.63 | 99 | gappy |
| 3 | 615 | 401 | 11.4 | 94.3 | 580 | 499 | 8.8 | 66 | 0.55 | 100 | stale |
| 4* | 634 | 510 | 13.7 | 98.5 | 256 | 253 | 3.4 | 52 | 0.71 | 99 | - |
| 5 | 648 | 573 | 9.3 | 92.5 | 747 | 655 | 8.9 | 44 | 0.29 | 72 | long, issues |
| 6 | 577 | 468 | 15.2 | 98.1 | 124 | 106 | 1.8 | 130 | 0.68 | 92 | - |
| 7 | 613 | 522 | 12.6 | 91.6 | 865 | 826 | 11.3 | 60 | 0.54 | 100 | - |
| 8 | 555 | 478 | 10.3 | 96.7 | 366 | 355 | 5.2 | 60 | 0.80 | 98 | - |
| 9 | 568 | 444 | 15.4 | 97.2 | 247 | 238 | 3.8 | 74 | 0.66 | 99 | - |
| 10 | 587 | 455 | 11.2 | 90.4 | 600 | 559 | 8.8 | 120 | 0.71 | 97 | - |
| 11 | 392 | 337 | 11.0 | 98.6 | 76 | 71 | 1.5 | 103 | 0.52 | 93 | issues |
| 13 | 431 | 288 | 21.7 | 94.6 | 419 | 277 | 9.0 | 68 | 0.44 | 90 | gappy, truncated |
| 15 | 567 | 472 | 13.7 | 97.1 | 261 | 247 | 3.8 | 91 | 0.70 | 83 | - |
| 16 | 544 | 423 | 17.3 | 96.2 | 317 | 300 | 5.0 | 73 | 0.63 | 97 | - |
| 17 | 470 | 420 | 6.2 | 97.6 | 284 | 275 | 4.6 | 61 | 0.38 | 85 | - |
| 18 | 441 | 384 | 5.9 | 98.8 | 137 | 128 | 2.4 | 48.5 | 0.32 | 84 | - |
| 19 | 470 | 429 | 6.5 | 96.1 | 258 | 244 | 4.1 | 102 | 0.24 | 9 | - |
| 20 | 460 | 408 | 6.0 | 97.5 | 194 | 189 | 3.3 | 74 | 0.38 | 86 | - |
| 21 | 487 | 394 | 9.4 | 96.6 | 234 | 222 | 4.0 | 86 | 0.63 | 99 | issues |

\* House 4 sums two machines; treat its cycle statistics with care.

**Pooled over all valid minutes:** the machine draws exactly 0 W for **89.1%** of the time, 0 to 20 W (standby/noise) for 6.9%, and more than 20 W for only **4.0%**. Off time per home ranges from 90.4% (H10) to 98.8% (H18). Missing minutes range from 5.9% (H18) to 24.6% (H2).

**Houses with a data-quality flag (6 of 19):** H2 (gappy), H3 (stale readings 13.3% of the time), H5 (long cycles, many `Issues` minutes), H11 and H21 (many minutes where the sub-meters sum to more than the aggregate), H13 (gappy, 142 of 419 cycles touch a data gap).

---

## 2. Cycle-detection rule

In plain language: a minute is **active** when the machine draws more than **20 W**. Active stretches separated by pauses of **up to 10 minutes** are joined into one event, because a wash has idle moments between heating, tumbling and spinning. An event counts as a **cycle** if it lasts at least **10 min**, peaks at **100 W or more** and uses at least **50 Wh**. Cycles longer than 240 min, or touching a data gap, are flagged and left out of duration and energy statistics. A cycle is **heated** if it has 5 or more minutes above 1,000 W.

Why this rule: there is no clean gap in the power distribution (`task2_wm_power_distribution.png`), so the threshold was checked by sensitivity instead. Changing the threshold (10 to 50 W) or the merge gap (5 to 20 min) moves the total cycle count by **under 5%** (6,445 to 7,015 across all homes, baseline 6,697).

Total: **6,697 cycles** detected over 19 homes; 6,123 usable in the deep-dive homes; 5,902 on days with enough data for timing analysis.

The `Issues` flag (sum of sub-meters above the aggregate) touches 36.6% of cycles by at least one minute, but only 5.7% by more than a quarter of their minutes. Dropping the affected cycles changes a home's median cycle energy by at most 1.7% (H7), so they are kept.

---

## 3. Usage patterns

### Hour of day and day of week (`task2_hour_and_weekday_patterns.png`)

- Most washes start in the morning: **51%** of cycles start between 06:00 and 12:00, peaking at **08:00** (11.1% of cycles). 27% start 12:00-18:00, 14% start 18:00-24:00, 8% start 00:00-06:00.
- Weekends are slightly busier: **0.79 vs 0.65 cycles per day**, Saturday highest, but the weekend is higher in only **10 of 19** homes, so this is a weak effect.
- Homes differ a lot. H18 starts 44% of its washes at 09:00 and H6 32% at 10:00; H8 (65%) and H19 (52%) mostly wash between 22:00 and 06:00, which fits an overnight or cheap-tariff habit.

**Clock caveat (unresolved).** The cleaned timestamps are not on one consistent clock. Comparing House 1's cleaned file with the raw file shows the cleaned series sits 60 minutes earlier in Oct 2013 and Apr-Sep 2014, and matches the raw file otherwise. Two daylight-saving checks disagree (spring 2014 says UTC, autumn 2014 says local). I did not settle this. As a clock-proof check, the hour pattern was repeated on **Nov-Mar only**, when UK time equals UTC (`task2_hour_clock_robustness.png`):

| Version | Busiest hour | Share 06-12 h | Share 16-19 h | Share 22-06 h |
|---|---|---|---|---|
| All year, stamps read as UK local | 08:00 | 50.6% | 11.5% | 11.3% |
| All year, stamps read as UTC, converted | 09:00 | 47.6% | 11.1% | 11.1% |
| Nov-Mar only (clock-proof) | 08:00 | 47.1% | 11.9% | 13.1% |

The largest hourly gap between the clock-proof curve and either all-year curve is 2.3 percentage points. So "mostly a morning activity" holds, but treat the exact peak hour as good to about one hour.

### Cross-house comparison (`task2_cross_house_comparison.png`)

- **Energy per cycle:** median from **0.24 kWh (H19) to 0.80 kWh (H8)**, a factor of 3.3.
- **Duration:** median from 33 min (H1) to 130 min (H6).
- **Frequency:** from 1.5 washes a week (H11) to 11.3 (H7). The Spearman correlation between washes per week and household size is 0.62, but with 19 homes read it as a hint.
- **Why energy differs:** H19 rarely heats the water (9% of its washes vs 72-100% elsewhere), which explains its low energy. Across all cycles, energy tracks **minutes spent heating** (Spearman 0.96) and barely tracks peak power (0.20).
- Seven usable cycles in H5, eight in H8, six in H11 and one each in H15 and H21 exceed 2 kWh, which is unusual for one wash and may be a washer-dryer or a mixed channel.

---

## 4. Required plots, with interpretation

**(i) Representative 24 hours, aggregate + washing machine** (`task2_representative_day.png`). House 7, Thursday 23 April 2015, chosen as the day closest to that home's median wash duration, energy and daily total. One 58-minute wash (07:50 to 08:47, 0.55 kWh, peak 2,068 W) is only **4% of the day's 13.1 kWh**. The aggregate peaks at 5.8 kW from other loads, and 112 minutes of the day exceed 1 kW with the washer off. This one picture explains why the problem is hard: the washer's heating burst is not special in the aggregate.

**(ii) Detected cycles** (`task2_detected_cycles.png`, `task2_cycle_overlay.png`). Four homes at the low-to-high end of cycle energy (H19, H13, H16, H8), plus 300 overlaid cycles from H7. A wash looks like a **heating burst of about 2 kW, a long low-power tumble, then a spin**. The rule finds whole washes rather than fragments. In H7 the median cycle is 60 min (10th to 90th percentile 55 to 113 min) and every cycle has a heating burst. H19's plotted cycle spends only 2% of its time above 1 kW: a cold or eco wash.

**(iii) Cross-house comparison** (`task2_cross_house_comparison.png`). The three-panel plot shows the spread in energy, duration and frequency described above. The takeaway: a single "average washing machine" would be a poor description of these homes, which is why a per-household-agnostic model struggles.

---

## 5. Model and validation design

**Task:** estimate the washing machine's power each minute from the household aggregate alone, which is what a utility has.

**Model:** gradient boosting (`HistGradientBoostingRegressor`) on 52 hand-built features computed from a window of the aggregate (-60 to +60 minutes): net load above the local base load at 24 offsets, rolling mean, max and standard deviation, counts of big steps, counts of heating-type load (above 1 kW), plus hour and weekday. This is the Seq2Point idea (window in, midpoint out) with hand-built features. It looks at future readings, so it suits after-the-fact reporting, not live control. No feature uses the washing-machine channel.

**Training sample:** every active minute, every minute within 15 minutes of one, and a random 10% of the far-off minutes weighted by 10, since the machine is off about 96% of the time. Model size (15 leaves) and number of trees (214) were chosen on the validation homes. Predictions below **30 W** are set to zero (cut-off chosen on validation to maximise on-state F1).

**Baselines:**
1. *Always off* (predict 0 W).
2. *Hour-of-week usage profile* (mean washing-machine power for each hour of the week, from the training homes).
3. *Simple rule on the aggregate* (predict 459 W when net load exceeds 1 kW in at least 5 of the surrounding 15 minutes; no tuning).

**Split by household, not by random minutes.** Neighbouring minutes look nearly identical, so a random split would leak. Splitting by home also tests the real question: does the model work in a home it has never seen? Homes were sorted by washes per week and every fifth went to test (or validation), fixed *before* any model was fitted, so no one picked an easy test set. Validation homes chose the model size, tree count and cut-off; test homes were scored once.

| Split | Homes | Usable minutes | Cycles |
|---|---|---|---|
| Train (11) | 11, 18, 15, 9, 19, 1, 16, 8, 10, 5, 7 | 7,428,759 | 4,270 |
| Validation (3) | 20, 17, 3 | 1,855,160 | 1,058 |
| Test (4) | 6, 21, 2, 13 | 2,380,937 | 1,113 |

Only valid minutes with `Issues` = 0 are used in the main numbers. A robustness check scores **all valid minutes** (see section 6).

**Leakage caveat.** All homes cover the same 2013-15 period, so train and test share seasons and calendar events. I treat this as a small leak and did not test it. **Three of the four test homes carry data-quality flags** (H2, H13, H21), so this test set is on the harder side.

**Why these metrics.** The machine is off about 96% of the time, so plain MAE rewards a model that predicts nothing (always-off has the *lowest* MAE at 13.6 W vs 17.2 W for the model). I therefore report:

- **NDE** (normalised disaggregation error; 1.0 = no better than always-off) for overall fit.
- **MAE on active minutes** and RMSE for error when the machine is actually running.
- **SAE** for total-energy bias.
- **Precision, recall and F1 of "machine on"** for detection quality.
- **Daily energy error and correlation**, and **whole-wash recall/precision**, which match how a utility would use the output.

---

## 6. Model results

### Test homes (6, 21, 2, 13), pooled, `Issues` = 0 minutes

| Model | MAE (W) | MAE on (W) | RMSE (W) | NDE | SAE % | Precision | Recall | F1 | Daily MAE (Wh) | Daily corr |
|---|---|---|---|---|---|---|---|---|---|---|
| Always off | 13.6 | 378.7 | 146.1 | 1.00 | -100.0 | - | 0.00 | 0.00 | 312.0 | - |
| Hour-of-week profile | 19.2 | 367.8 | 145.9 | 1.00 | -52.9 | 0.06 | 0.30 | 0.11 | 343.7 | 0.05 |
| Simple rule (heavy load) | 50.2 | 375.7 | 175.9 | 1.45 | +218.6 | 0.15 | 0.40 | 0.22 | 773.6 | 0.26 |
| **Gradient boosting** | 17.2 | **261.5** | **115.6** | **0.63** | +1.3 | **0.28** | **0.49** | **0.36** | **219.7** | **0.75** |

The model wins on every metric except plain MAE. The simple rule is *worse than always-off* (NDE 1.45) because it fires on every heavy load. The usage profile is no better than always-off: when you wash is not predictable enough from the clock.

Validation homes (20, 17, 3) give similar numbers (NDE 0.64, F1 0.41, daily correlation 0.80), so the test result is not a lucky draw.

### Per test home (gradient boosting)

| House | F1 | NDE | SAE % | Wash recall | Wash precision |
|---|---|---|---|---|---|
| 2 | 0.29 | 0.70 | -3.0 | 0.31 | 0.31 |
| 6 | 0.42 | 0.25 | +8.3 | 0.88 | 0.42 |
| 13 | 0.55 | 0.67 | -7.7 | 0.86 | 0.54 |
| 21 | 0.20 | 0.69 | +22.9 | 0.35 | 0.22 |

F1 ranges from 0.20 to 0.55 across only four homes: **home-to-home variation is the main uncertainty**.

### Whole-wash detection (Task 2 cycle rule applied to the predictions)

A real wash counts as *found* if at least half of it is covered by a predicted cycle; a predicted cycle is *correct* if at least half overlaps a real wash.

- Found **503 of 908** real washes (**55%**).
- Predicted **2,412** cycles, of which **852 (35%)** are real.
- Energy per found wash is **under-estimated by a median of 53%** (median absolute error 54%).

### The +1% energy bias is misleading

Pooled SAE is +1.3%, which looks excellent. It is not: found washes are about half their true energy, and the false alarms add that energy back. Per home the bias runs from -7.7% to +22.9%. Do not quote the pooled bias as evidence of accuracy.

### Actual vs predicted (`task3_actual_vs_predicted_day.png`, `task3_test_overview.png`)

- **Day plot** (median-F1 test home, a day with median error): the model puts the wash about in the right place but far too low, 34 Wh predicted vs 190 Wh actual, day NDE 0.77, F1 0.32. It is clearly better than the baselines but nowhere near a faithful reconstruction.
- **Scatter of daily energy:** points follow the diagonal loosely (correlation 0.75).
- **Bars:** the model beats both baselines in every test home; F1 spans 0.20 to 0.55.

### Robustness checks

| Check | Result | Reading |
|---|---|---|
| Score on **all valid minutes**, not just `Issues` = 0 (the flag is built from sub-meters, which a utility would not have) | F1 0.36 to 0.36, NDE 0.63 to 0.67, SAE +1.3% to -9.4%, daily corr 0.75 to 0.71 | The `Issues` filter is not flattering the model much; the energy bias moves, so treat it as roughly -10% to +1%. |
| **Day-block bootstrap**, 1,000 draws over 1,779 test house-days | F1 0.358 (95% CI 0.341 to 0.375); NDE 0.669 (0.643 to 0.694) | The gain over always-off is not day-to-day noise. This captures variation **within** these four homes only; between-home variation is larger (see per-home F1). |
| **Drop clock features** (hour, weekday) and refit | F1 0.36 to 0.32, NDE 0.63 to 0.65, daily corr 0.75 to 0.73 | The model leans a little on the clock (about 0.04 F1). Given the unresolved clock problem above, part of that gain may rest on an inconsistent clock. Without it the model still beats all baselines, so the core result stands. |

---

## 7. Failure analysis

The other appliance channels are used here for **explanation only**; the model never sees them. Cases are plotted in `task3_failure_cases.png` (two biggest misses, two biggest false alarms).

**1. It detects "a heavy load", not specifically "the washing machine".** False-alarm minutes make up 4.4% of usable test minutes (105,158 minutes). During those:

| Other load running (> 1 kW) | Share of false-alarm minutes | Share of all minutes | Enrichment |
|---|---|---|---|
| Dishwasher | 24.8% | 1.4% | 17.8x |
| Kettle / microwave / toaster | 7.4% | 0.8% | 9.2x |
| Other washer / dryer | 3.8% | 1.0% | 3.7x |
| Unmonitored load (cooker, shower, etc.) | 40.9% | 5.5% | 7.4x |

The result is over-triggering: 2,412 predicted cycles vs 908 real, only 35% correct. Washes are also found *more* often when other heavy loads run during them (73% found with heavy load over more than 25% of the wash vs 27% with none), a sign the model partly keys on "heavy load on".

**2. Cold or eco washes are mostly missed.** Washes with no heating burst are found 24% of the time vs 57% for heated ones. (Only 42 such washes in the test set, so this is indicative.)

**3. Energy per wash is badly under-estimated (about -53%)** even when the wash is found, because the long low-power tumble and spin phases sit close to the noise of the aggregate.

**4. Home differences.** Recall ranges from 0.31 (H2) to 0.88 (H6); F1 from 0.20 (H21) to 0.55 (H13). H2 and H21 are the flagged, harder homes, with many false alarms from other loads.

**Where it is likely to struggle beyond this test:** homes with a washer-dryer or other heavy loads that overlap the wash; machines that never heat water; dead or stale sensors and long data gaps (the model must not report "no washing" for a missing day); and any home very unlike these UK houses.

---

## 8. Interpretation

- Washing is rare (about 4% of minutes), morning-heavy and highly household-specific. A clock-based profile alone has no skill.
- Learning the *shape* of a wash from the aggregate gives real but modest skill: it removes about 37% of the squared error versus always-off and about triples the F1 of the profile baseline.
- The honest summary: **good enough to say roughly when and how much washing a group of homes does; not good enough to say what one wash used.**

## 9. Limits of this evaluation and next steps

- Four test homes, three of them flagged, so per-home spread is poorly estimated. Leave-houses-out cross-validation is the first fix.
- Same calendar period in train and test; a forward-in-time check within homes is still to do.
- Clock inconsistency in the cleaned files is not fully resolved (verified for House 1 only).
- Next: cut false alarms with a stricter on/off rule or a sequence model (Seq2Point or a small CNN); add reactive power or 1-second data; calibrate per home with a few labelled weeks; and test on sub-metered Indian homes, which is the biggest unknown.

## References

- Zhang et al., *Sequence-to-point learning with neural networks for non-intrusive load monitoring*, AAAI 2018: window-in, midpoint-out idea.
- Kelly & Knottenbelt, *Neural NILM*, BuildSys 2015: deep learning for NILM.
- Batra et al., *NILMTK*, e-Energy 2014: evaluation framing and metrics.
- Pereira & Nunes, *Performance evaluation in NILM*, WIREs Data Mining and Knowledge Discovery 2018: metrics (NDE, SAE, F1).
- Klemenjak et al., *On metrics to assess the transferability of ML models in NILM*, arXiv:1912.06200: unseen-house testing.
- Ke et al., *LightGBM*, NeurIPS 2017: gradient boosting background.
- Murray et al., *An electrical load measurements dataset of United Kingdom households*, Scientific Data 4:160122, 2017: the REFIT dataset.

The ideas adopted are the window input, the unseen-house test and the metrics; the code is my own.
