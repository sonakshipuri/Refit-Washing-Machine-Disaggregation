# Recommendation.md

**Utility-facing summary: washing-machine disaggregation from smart-meter data**
Based on 19 UK homes from REFIT (2013-15). Full evidence is in `RESULTS.md`.

## The one-paragraph version

From the whole-home meter alone, our model can tell *roughly when and how much* a group of homes does laundry. It cannot reliably tell you *what one wash used* or *whether a specific home washed at a specific time*. On four homes it had never seen, it cut the error by about 37% compared with predicting "never washing" and clearly beat a usage-profile baseline, but only about 1 in 3 of the "washing" minutes it reports is real, and the energy of a detected wash is under-estimated by about half.

## Test results at a glance (4 unseen homes, 1-minute data)

| | Always off | Usage profile | Our model |
|---|---|---|---|
| Error vs always-off (NDE, lower is better) | 1.00 | 1.00 | **0.63** |
| "Machine on" F1 | 0.00 | 0.11 | **0.36** |
| Daily-energy correlation | - | 0.05 | **0.75** |
| Real washes found | 0% | 0% | **55%** (of 908) |
| Predicted washes that are real | - | - | **35%** (of 2,412) |
| Energy of a found wash | - | - | **about -53% (too low)** |

Home-to-home spread is large: F1 from 0.20 to 0.55 across the four test homes.

## Suitable uses

1. **Cohort-level laundry timing, after calibration.** "What share of washing energy in this feeder or tariff group falls in the evening peak?" Washing is morning-heavy (51% of washes start 06:00-12:00), so the evening share is small and shifting it is a modest, shared benefit. Treat these estimates as provisional until calibrated against a sub-metered or diary panel; the model overstated peak share in all four test homes.
2. **Customer segmentation and screening for outreach.** Flag homes that probably wash a lot in the evening, then confirm by survey or opt-in before any message goes out.
3. **Trend tracking across a group** (month over month), after calibrating the model against a small sub-metered or diary panel.
4. **Research and planning inputs** where errors average out across many homes.

## Not suitable (today)

- **Per-wash or per-home energy claims.** Detected washes read about half their true energy; pooled energy looked accurate (+1%) only because false alarms cancel the under-estimate.
- **Billing, rebates or demand-response settlement for an individual home.**
- **Telling a customer "your washing machine used X kWh".** Dishwashers, kettles, cookers and heaters trigger false alarms (a dishwasher is running in 25% of false-alarm minutes vs 1.4% of all minutes).
- **Real-time control.** The model looks 60 minutes ahead and behind, so it is after-the-fact only.
- **Homes with 15- or 30-minute data**, for any per-wash purpose (see Section 4.3 below).
- **Indian homes**, until re-trained and validated on local sub-metered data.

## Conditions before deployment

- Calibrate the evening-share estimate (the model overstates it by about 5 percentage points, in all four test homes) with a small plug-metered or diary panel.
- Do not treat missing or stale-sensor days as "no washing".
- Re-check timestamps: the cleaned REFIT clock is inconsistent in some months, which blurs hour-of-day results by up to an hour.
- Re-estimate sample sizes with a pilot; the numbers below rest on 19 homes.

---

# Section 4 answers

## 4.1 One practical opportunity: move evening washes out of the peak

**Opportunity.** Identify homes that wash during the local evening peak and nudge them to use delayed start or off-peak timing. It needs no new hardware: the signal comes from smart-meter data the utility already has.

**How big is it?** Using the sub-metered cycles (ground truth) of all 19 homes and a 16:00-19:00 peak window:

- The median home does about **10% of its washes** in the window (9% of wash energy).
- The median home has only about **12 kWh a year** of washing-machine energy consumption during this window (equivalent to roughly 11 W averaged across the 16:00–19:00 window); the top quarter of homes have about 28 kWh.
- The best targets are **H16, H10 and H9**, with 25-35% of washes in the window.

So the household bill saving is small. The value is lower evening demand summed across many homes, plus customer engagement. For India, replace the window with the local peak (often about 18:00-22:00) or the state's time-of-day tariff.

**How to measure whether it worked.**

- **Design:** randomised trial on opted-in smart-meter homes. Treatment gets a monthly laundry-timing message plus a delayed-start prompt; control gets nothing. Each household's outcome is compared with its own previous 8 weeks, with the concurrent control group helping account for seasonal changes, weather and holidays.
- **Main metric:** share of washing energy inside the peak window, 8 weeks after vs before (and peak-window kW).
- **Guardrails:** total washing energy and number of washes (no rebound, no suppressed laundry), and opt-outs.
- **Measurement error:** check the model's peak-share estimate against a small panel with plug meters or a laundry diary, because it overstates the peak share by about 5 percentage points (mean error 4.7 pp, above truth in 4 of 4 test homes).
- **Sample size:** a home's peak share moves by a standard deviation of 9.3 pp between two untreated 8-week blocks. At 80% power and 5% significance, detecting a **5 pp** shift needs about **55 homes per arm**; **3 pp** about **150**; **10 pp** about **14**. This rests on 19 UK homes, so run a pilot first and re-estimate.

## 4.2 Limitations

- **Overlapping appliance signatures.** Dishwashers, dryers, kettles and heaters look like the washer's heating burst. This is the main source of false alarms. Whole-cycle shapes, 1-second data or reactive power would help.
- **Missing data.** 6-25% of minutes per home have no reading, and flat-lined values hide dead sensors. We never train or score on those minutes, so a live system must not report "no washing" for a missing day.
- **Household differences.** F1 ranges from 0.20 to 0.55 across only four test homes, and three of those four carry data-quality flags. A few labelled weeks per home would probably help.
- **Clock.** The cleaned timestamps are inconsistent in some months (checked for House 1), so hour-of-day and peak-window numbers can be off by an hour or more. Nov-Mar-only versions give similar conclusions (median share of washes in peak 10.3% vs 10.0%). Dropping clock features lowers F1 from 0.36 to 0.32.
- **Evaluation size.** Four test homes, and all homes cover the same 2013-15 period, so train and test share seasons.
- **UK to India** (my assumptions, not tested). The model leans on the roughly 2 kW heating burst of UK front-loading machines. Many Indian homes use top-load or semi-automatic machines, often with cold water, which look like a few hundred watts of motor power and are much harder to separate. Indian homes also have large loads not in REFIT (air conditioners, geysers, pumps, inverters), different washing habits (hand washing, domestic help), different tariffs and peak times, and different voltage quality and meters. REFIT is 20 mostly detached homes in one English area. Re-train and validate on sub-metered Indian homes before any use.

## 4.3 What gets harder with 15- or 30-minute data

A typical wash lasts about 69 minutes and is recognised by its shape: a heating burst of about 13 minutes at about 2 kW, a long low-power tumble and a spin. At 15-minute resolution a wash is only about 5 samples and the heating burst is shorter than one sample; at 30 minutes the burst blends into everything else in the block. Using the same homes, split and model with only the resolution changed:

| Resolution | NDE | On-state F1 | Energy bias (SAE) | Daily-energy correlation |
|---|---|---|---|---|
| 1 min | 0.63 | 0.36 | +1% | 0.75 |
| 15 min | 0.77 | 0.31 | -11% | 0.66 |
| 30 min | 0.80 | 0.31 | -15% | 0.60 |

Timing is only known to the block, appliances with a similar average draw become hard to separate, and counting washes or measuring energy per wash becomes unreliable. Daily and weekly washing totals and time-of-day patterns stay roughly usable (weekly was not tested), but per-wash insight is lost.
