# Design Document: Passenger No-Show Classification

**Status:** Draft v1 | **Data:** Simulated (see Section 9)

## 1. Problem Statement

For each booking still active at a given snapshot before departure, predict whether the passenger will **board**, **no-show**, or **cancel before departure**. Outputs are calibrated probabilities used for operational planning (catering, staffing, seat inventory) and passenger communication (reminders and check-in nudges). Predictions are never used to deny boarding or to take any adverse action against an individual passenger.

## 2. Prediction Snapshots

Predictions are made at fixed points before departure. Each snapshot has its own feature set, because information becomes available over time.

| Snapshot | Hours to departure | Information available |
|---|---|---|
| S1 | 168 (T-7d) | Booking attributes, itinerary, passenger history. Online check-in is not yet open. |
| S2 | 24 (T-24h) | Everything in S1, plus check-in status and timing so far. |
| S3 | 3 (T-3h) | Everything in S2, plus latest check-in, seat selection and bag drop events. |

**Point-in-time rule:** a feature may use an event only if `event_time <= snapshot_time`. This rule is enforced by unit tests in Week 2.

## 3. Population and Target Definition

**Population at each snapshot:** bookings that are still active (not cancelled before that snapshot).

**Target (3 classes), measured after the snapshot:**
- `boarded`: the passenger boarded the flight.
- `no_show`: the passenger held an active booking at departure but did not board.
- `cancelled_after_snapshot`: the passenger cancelled between the snapshot and departure.

**Consequences:**
- The population shrinks as departure approaches, so the class mix differs by snapshot. At S3, cancellations are rare and no-shows are a larger share of non-boarders.
- Models are evaluated per snapshot, never pooled.

**Why not collapse cancellations into one class?** Cancellations made before the snapshot are already known at prediction time, so they are not something to predict. Only cancellations that happen after the snapshot are genuinely uncertain. Including earlier ones would leak known information into the target.

**Worked example (S2, T-24h):** a passenger booked 30 days ago, with no cancellation so far, appears in the S2 table. Their features use only events up to T-24h. If they cancel at T-10h, the label is `cancelled_after_snapshot`. If they cancel at T-30h, they do not appear in S2 at all. **(your call: replace with an example from your own simulated data once Day 3 is done.)**

## 4. Data Split

Temporal split by **flight departure date**, not booking date:

| Set | Departure dates |
|---|---|
| Train | 2023-01-01 to 2023-08-31 |
| Validation | 2023-09-01 to 2023-10-31 |
| Test | 2023-11-01 to 2023-12-31 |

Rules:
- The test set is not touched until model selection is final (end of Week 3).
- A passenger can appear in several periods. Historical features for a given row use only flights departing before that row's departure.
- Target encodings and any fitted preprocessing are fit on training data only, with time-aware folds.

## 5. Metrics

**Primary:** PR-AUC for the `no_show` class, reported per snapshot.

**Probability quality:** multiclass log loss, Brier score, reliability (calibration) plots before and after calibration.

**Business:**
- Expected cost at the chosen threshold (Section 6).
- Flight-level MAE between predicted and actual no-show counts.
- Simulated lift of model-targeted outreach versus random outreach.

**Not reported as headline metrics:** accuracy. With roughly 5% no-shows, a model that always predicts "boarded" scores about 85% accuracy while being useless.

**Baselines:** majority class, then logistic regression.

## 6. Cost Assumptions

Defined in `configs/costs.yaml`. All values are illustrative cost units, not real airline figures.

| Event | Assumed value |
|---|---|
| Unnecessary reminder message (false alarm) | cost of 1 |
| Empty seat from an unforecast no-show | cost of 50 |
| Seat saved after a successful outreach | benefit of 20 |
| Outreach response rate | 30% |

Decision thresholds are chosen by minimizing expected cost, not by defaulting to 0.5. A sensitivity analysis varies each assumption (ranges in `costs.yaml`) to show how conclusions change.

## 7. Responsible Use

**Intended use:**
- Flight-level no-show forecasts for capacity, catering and ground staffing planning.
- Reminder and check-in nudges to passengers with elevated risk.

**Prohibited use:**
- Automatic denial of boarding.
- Fare penalties, fees or any adverse action against an individual based on a prediction.

**Excluded features:** nationality, name, gender, age, and any feature that acts as a proxy for them.

**Output policy:** the scoring API returns calibrated probabilities and coarse risk bands only. It returns no binary "deny" or "no-show" flag. Any operational use requires a human in the loop.

**Fairness checks:** error rates and calibration are compared across fare class, region and booking channel, and reported in the model card (Week 4).

## 8. Out of Scope

- Overbooking optimization and inventory pricing.
- Real-time streaming inference.
- Decisions about individual passengers at the airport.
- Any claim of accuracy on real airline data.

## 9. Data and Limitations

Real passenger booking records are proprietary, so this project uses a **simulator** built on real flight schedules from the US BTS On-Time Performance dataset. Booking behavior, no-show and cancellation outcomes are generated from documented assumptions in `configs/sim.yaml` and the simulator code. A model trained on this data learns the simulator's rules, so results demonstrate the methodology (leakage control, calibration, cost-based decisions, responsible deployment), not real-world performance.

The Kaggle Hotel Booking Demand dataset is used as a secondary check that the pipeline behaves sensibly on real booking behavior.

## 10. Open Questions

- Should S1 include a model of fare-class-specific cancellation fees?
- How should group bookings be handled (one row per booking or per passenger)?
- Does the outreach simulation need a model of passengers who respond but still do not travel?
