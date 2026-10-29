# REFIT: data cleaning and washing-machine disaggregation

Estimate a washing machine's power each minute from a home's aggregate smart-meter signal, using REFIT (20 UK homes, 2013-15).

| File | Content |
|---|---|
| `notebook.ipynb` | All code: cleaning, EDA, model, evaluation, Section 4 numbers |
| `DATA.md` | Raw-data coverage, quality, cleaning, concerns |
| `RESULTS.md` | EDA, plots, model results, validation, failure analysis |
| `Recommendation.md` | Utility-facing summary and Section 4 answers |

## What I built

1. **Cleaning (raw House 1):** join, sort, de-duplicate, flag impossible values, resample to 1 min, fill gaps of 5 min or less, flag longer outages. Before/after plots.
2. **EDA (19 homes, official cleaned data):** cycle rule, availability, hour/weekday patterns, cross-house comparison. 6,697 cycles.
3. **Model:** gradient boosting on features from the aggregate only, tested on 4 unseen homes against 3 baselines. Failure analysis and robustness checks.
4. **Practical implications:** evening-peak laundry timing, limitations, 15/30-minute data.

**Result:** on the 4 test homes NDE is 0.63 (always-off = 1.00) and on-state F1 is 0.36 (usage profile 0.11). About 1 in 3 predicted washing minutes is real, and energy of a found wash is about half the true value. Usable for group-level patterns, not per-wash or per-bill claims.

## Setup and run

```bash
pip install pandas numpy matplotlib scipy scikit-learn pyarrow jupyter
```

Reported numbers used pandas 2.3.3 and numpy 2.4.6.

Download and unpack into `DATA/` (not committed):

| Archive | Unpack to |
|---|---|
| [REFIT_RAW_081116.7z](https://pureportal.strath.ac.uk/files/62090177/REFIT_RAW_081116.7z) | `DATA/REFIT_RAW_081116/` (House 1 Part1 and Part2 needed) |
| [CLEAN_REFIT_081116.7z](https://pureportal.strath.ac.uk/files/62090184/CLEAN_REFIT_081116.7z) | `DATA/CLEAN_REFIT_081116/` |

Run all cells top to bottom in one kernel (the Task 2 clock check uses the Task 1 result):

```bash
jupyter nbconvert --to notebook --execute --inplace notebook.ipynb
```

Outputs go to `outputs/figures/`, `outputs/tables/`, and `outputs/house1_raw_cleaned_1min.parquet`. Per-house 1-minute parquet files are cached in `outputs/cache/`; the first run is the slow one. Random seeds are fixed.

## Assumptions

- Power is average real power in watts; a 1-minute value is the mean of the readings in that minute.
- Washing-machine channels come from the REFIT README and were not checked against the signals.
- The model sees only the aggregate. Other channels are used for labels and failure analysis only.
- It is after-the-fact: each prediction uses the 60 minutes before and after.
- A missing minute means unknown, not off.

## Preprocessing decisions

Details and counts in `DATA.md`.

- Duplicate seconds: keep the last non-missing value per channel.
- Aggregate of 0 W or above 23 kW set to missing; isolated spikes above 10 kW removed; appliance readings above 4,000 W set to 0 (official rule).
- Gaps of 5 min or less interpolated; longer ones flagged and left empty. Cycles are never joined across a gap.
- Cycle rule: active above 20 W; pauses up to 10 min merged; a cycle is 10+ min, peaks at 100+ W, and uses 50+ Wh. Count changes by under 5% when thresholds vary.
- Six of 19 homes are quality-flagged, none dropped. House 4 (two machines on one channel) is out of the model.
- Main model numbers use `Issues` = 0 minutes; a check scores all valid minutes.

## Validation design

- **Model:** `HistGradientBoostingRegressor`, 52 features from a -60 to +60 min window of the aggregate (load above local base load at 24 offsets, rolling mean/max/std, step counts, heavy-load counts, hour, weekday). Seq2Point-style window, hand-built features.
- **Training sample:** all active minutes, minutes within 15 min of one, and 10% of the rest at weight 10 (the machine is off about 96% of the time). Predictions under 30 W set to 0.
- **Split by household,** not by minute: neighbouring minutes are near-identical, so a random split leaks, and a household split tests transfer to unseen homes. Homes were sorted by washes per week and every fifth assigned to test or validation before any fitting.

| Split | Homes | Cycles |
|---|---|---|
| Train (11) | 11, 18, 15, 9, 19, 1, 16, 8, 10, 5, 7 | 4,270 |
| Validation (3) | 20, 17, 3 | 1,058 |
| Test (4) | 6, 21, 2, 13 | 1,113 |

- Validation homes set tree count (214), size (15 leaves) and the 30 W cut-off. Test homes were scored once.
- **Baselines:** always off; hour-of-week profile; simple heavy-load rule.
- **Metrics:** NDE, MAE/RMSE on active minutes, SAE, on-state precision/recall/F1, daily energy error and correlation, whole-wash recall and precision. Plain MAE is not used for ranking because always-off has the lowest MAE.
- **Checks:** validation homes, day-block bootstrap, all valid minutes, no clock features.

## Trade-offs

- Gradient boosting with hand-built features over a neural network: easier to explain and enough to show skill. A sequence model would likely cut false alarms.
- Household split over a time split: tests new homes, but all homes share 2013-15, so seasons leak. No forward-in-time test was run.
- Four test homes, three of them flagged: harder test, poorly estimated spread (F1 0.20 to 0.55).
- Flagged homes kept: more realistic, noisier.
- Official cleaned data for modelling, as the brief asks; my cleaning is applied to House 1 only.
- Looking 60 min ahead helps accuracy and rules out real-time use.

## Limitations

- Cleaned-file clock is inconsistent in some months (checked for House 1 only); hour-of-day results can be off by about an hour.
- The model finds "a heavy load" more than "the washing machine": 2,412 predicted cycles vs 908 real, 35% correct. Dishwashers, kettles and cookers cause most false alarms.
- Cold washes are mostly missed (24% found vs 57% of heated washes).
- Energy per found wash is about 53% too low. The +1% pooled energy bias comes from false alarms cancelling this.
- UK to India transfer is untested.

## Possible improvements

1. Leave-houses-out cross-validation over all 18 modelled homes.
2. Forward-in-time check within homes.
3. Verify the clock for every home against the raw files.
4. Reduce false alarms: sequence model (Seq2Point or small CNN), stricter on/off rule, whole-cycle features.
5. Reactive power or 1-second data.
6. Per-home calibration with a few labelled weeks.
7. Calibrate the evening-peak share against a plug-metered or diary panel.
8. Validate on sub-metered Indian homes.

## References

- Zhang et al., *Sequence-to-point learning with neural networks for non-intrusive load monitoring*, AAAI 2018: window-in, midpoint-out.
- Kelly & Knottenbelt, *Neural NILM*, BuildSys 2015.
- Batra et al., *NILMTK*, e-Energy 2014: evaluation framing.
- Pereira & Nunes, *Performance evaluation in NILM*, WIREs DMKD 2018: NDE, SAE, F1.
- Klemenjak et al., *On metrics to assess the transferability of ML models in NILM*, arXiv:1912.06200: unseen-house testing.
- Ke et al., *LightGBM*, NeurIPS 2017.
- Murray et al., *An electrical load measurements dataset of United Kingdom households*, Scientific Data 4:160122, 2017.

Adopted from the literature: the window input, the unseen-house test and the metrics. The code is my own.
