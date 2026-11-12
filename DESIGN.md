# Design Document: Passenger No-Show Classification

**Status:** Draft v1 | **Data:** Simulated (see Section 9)

## 1. Problem Statement

For each booking still active at a given snapshot before departure, predict whether the passenger will **board**, **no-show**, or **cancel before departure**. Outputs are calibrated probabilities used for operational planning (catering, staffing, seat inventory) and passenger communication (reminders and check-in nudges). Predictions are never used to deny boarding or to take any adverse action against an individual passenger.

## 2. Prediction Snapshots

| Snapshot | Time before arrival/departure | Track A (hotel, real) | Track B (airline, simulated) |
|---|---|---|---|
| S1 | 7 days | Booking attributes, customer history | Same, plus booking and itinerary data |
| S2 | 1 day | Same as S1 (daily granularity only) | Adds check-in status and timing |
| S3 | 3 hours | Not available (no timestamps) | Adds latest check-in, seat selection, bag drop |

**Point-in-time rule:** a feature may use an event only if `event_time <= snapshot_time`. Enforced by unit tests.

The hotel data has daily granularity and no check-in signal, so Track A uses S1 and S2 only. Check-in snapshots exist only in the simulated airline track.

## 3. Population and Target Definition

**Population at snapshot k:** bookings still active at k days before arrival, meaning `lead_time >= k` and not cancelled earlier than k days before arrival.

**Target (3 classes), measured after the snapshot:**
- `showed` (hotel: Check-Out; airline: boarded)
- `no_show`
- `cancelled_after_snapshot`

Cancellation timing is reconstructed from `reservation_status_date`. Bookings cancelled before the snapshot are excluded because that outcome is already known.

## Leakage Audit (Track A)

| Column | Decision | Reason |
|---|---|---|
| `is_canceled`, `reservation_status` | Drop | They are the target |
| `reservation_status_date` | Use only to build labels and snapshots | Records the outcome date |
| `assigned_room_type` | Drop | Assigned at or near check-in |
| `booking_changes` | Drop | Final count includes changes after the snapshot |
| `required_car_parking_spaces`, `total_of_special_requests` | Drop or test as sensitivity | Final state, may be updated after the snapshot |
| `days_in_waiting_list` | Review | Possible post-booking information |
| `deposit_type` | Keep, with a sensitivity run | Legitimate at booking time, but Non Refund is near-deterministic with cancellation, which looks like a data artifact |
| `country` | Drop | Nationality proxy, excluded by responsible-use policy |
| `agent`, `company` | Review | High cardinality, use time-aware encoding |

## 4. Data Split

Temporal split by **arrival date** (the dataset spans roughly July 2015 to August 2017; verify in the audit):

| Set | Arrival dates |
|---|---|
| Train | 2015-07-01 to 2016-09-30 |
| Validation | 2016-10-01 to 2017-02-28 |
| Test | 2017-03-01 to 2017-08-31 |

The test window covers a full spring and summer season, so it also measures seasonal shift. Test data is untouched until model selection is final.

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

**Track A (real):** Hotel Booking Demand dataset (Antonio, Almeida & Nunes, 2019; CC BY 4.0), about 119,000 bookings from two Portuguese hotels. It is a different domain from airlines and has no check-in signal. No-shows are about 1% of bookings, so imbalance handling matters.

**Track B (simulated):** Real BTS flight schedules with simulated bookings and check-in events, built from documented assumptions in `configs/sim.yaml`. Used only to demonstrate flight-level aggregation, check-in snapshots and an overbooking simulation. Results from Track B illustrate method, not real performance.

**At an airline,** both tracks would be replaced by PNR and DCS data. Involuntary non-boardings (missed connections, disruption rebooks, denied boarding) would be excluded from the no-show label.

## 10. Open Questions

- Should S1 include a model of fare-class-specific cancellation fees?
- How should group bookings be handled (one row per booking or per passenger)?
- Does the outreach simulation need a model of passengers who respond but still do not travel?
