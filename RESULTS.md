# Results Log

Running log of experiments and findings. Every result is tagged **Real (hotel)** or **Simulated (airline)**.

---

## Day 2: Data audit

**Data:** Real (hotel). Hotel Booking Demand dataset (Antonio, Almeida & Nunes, 2019; CC BY 4.0).
**Notebook:** `notebooks/01_data_audit.ipynb`
**Note:** all numbers below are computed on the raw file, **before deduplication**, unless stated otherwise.

### 1. Basics

- Shape: **119,390 rows x 32 columns** (36 after the derived date columns).
- Missing values: `company` 112,593 (94.3%), `agent` 16,340 (13.7%), `country` 488 (0.4%), `children` 4.
- Exact duplicate rows: **31,994 (26.8%)**.
- `is_canceled` is stored as text (`yes`/`no`) in this copy of the file, not 0/1.

> Finding: more than a quarter of rows are exact duplicates. The file has no booking ID, so some may be legitimately separate bookings with identical attributes (for example, group bookings), but duplicates that land in both train and test would inflate scores. Decision recorded in the table at the end.

### 2. Classes

| Class | Count | Rate |
|---|---|---|
| Check-Out (showed) | 75,166 | 62.96% |
| Canceled | 43,017 | 36.03% |
| No-Show | 1,207 | 1.01% |

| Hotel | Canceled | Check-Out | No-Show |
|---|---|---|---|
| City Hotel | 40.57% | 58.27% | 1.15% |
| Resort Hotel | 27.04% | 72.24% | 0.73% |

> Finding: no-shows are about 1% overall, a much harder imbalance than the 5% airline assumption. City Hotel has about 1.6x the no-show rate of Resort Hotel, so `hotel` is a real signal.

### 3. Date sanity

- Arrival dates span **2015-07-01 to 2017-08-31**, which matches the planned split windows.
- Canceled rows with `status_date` after arrival: **0**.
- Rows with `status_date` before `booking_date`: **0**.

> Finding: `reservation_status_date` is internally consistent, so snapshot logic built on it is safe. No anomaly rule is needed.

### 4. Cancellation timing

Share of the 43,017 cancellations, by when they were recorded:

| Recorded | Share |
|---|---|
| More than 7 days before arrival | 88.1% |
| 1 to 7 days before arrival | 9.9% |
| Less than 1 day before arrival (same day) | 2.0% |

> Finding: nearly 9 in 10 cancellations happen more than a week out, so they are already known at both snapshots and removed from the population. Only about 12% of cancellations remain as a prediction target at the 7-day snapshot.

### 5. No-show timing

- No-shows with `status_date == arrival_date`: **99.9%** (about 1 row differs, max gap 111 days).

> Finding: a no-show is recorded on the arrival date, as expected. The single outlier should be inspected before training and either dropped or explained.

### 6. Leakage probes

Outcome by `deposit_type`:

| Deposit type | Canceled | Check-Out | No-Show |
|---|---|---|---|
| No Deposit | 27.3% | 71.6% | 1.1% |
| Non Refund | **99.1%** | 0.6% | 0.2% |
| Refundable | 21.6% | 77.8% | 0.6% |

> Finding: `Non Refund` bookings are 99.1% cancelled, which is implausible for real behavior and looks like a data artifact. Keep `deposit_type` only with a sensitivity run that drops it, and report both.

**Not yet run:** the `booking_changes` and `assigned_room_type == reserved_room_type` probes. The cell failed with `TypeError` because `is_canceled` is text. Fix and rerun before finalizing the leakage table in `DESIGN.md`.

### 7. Snapshot sizes

Snapshot = bookings still active k days before arrival (`lead_time >= k` and not cancelled earlier than k days out).

| Snapshot | Rows | Showed | Cancelled after | No-show | No-shows (approx.) |
|---|---|---|---|---|---|
| S1 (7 days) | 63,093 | 92.69% | 5.93% | 1.38% | ~870 |
| S2 (1 day) | 71,539 | 96.80% | 1.67% | 1.53% | ~1,090 |

> Findings:
> - Post-snapshot cancellations fall from 5.9% to 1.7% between S1 and S2, as designed.
> - S2 has **more** rows than S1, because bookings made 1 to 6 days before arrival only enter the population at S2. The population is not a shrinking subset, which corrects the wording in `DESIGN.md` Section 3.
> - With only about 870 to 1,090 no-shows per snapshot across all dates, the validation and test windows will contain a few hundred each. Expect noisy PR-AUC, so report confidence intervals (bootstrap) and avoid over-reading small differences.

### Decisions and open items

| # | Item | Decision | Status |
|---|---|---|---|
| 1 | Duplicates | Drop exact duplicates (119,390 to 87,396 rows). Cancel rate falls 36.03% to 26.33%, no-show rises 1.01% to 1.16% | **Final** |
| 2 | Date anomalies | None found; no rule needed. Inspect the one no-show with a 111-day gap | Open (minor) |
| 3 | Snapshot sizes | Recompute on deduplicated data | Pending cell 4 above |
| 4 | `deposit_type` | Keep with sensitivity run (with and without) | Planned |
| 5 | `booking_changes`, `assigned_room_type` | Drop. Cancel rate 39.8% vs ~10-27% (changes) and 40.6% vs 4.0% (room match) shows they leak the outcome | **Final** |
| 6 | Metric noise | About 1,010 no-shows after dedup; bootstrap confidence intervals on no-show PR-AUC | Planned |
