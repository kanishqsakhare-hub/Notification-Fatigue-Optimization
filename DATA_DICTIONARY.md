# Data Dictionary

## Source files

| File | Grain | Known size | Role |
|---|---|---:|---|
| `users.csv` | One row per user | 5,000 rows; 8 columns | User profile and baseline characteristics |
| `notification_events.csv` | One row per candidate notification decision | 85,019 rows; 25 columns | Delivery decision, response, and related event outcomes |

The executed audit covers January 1–March 31, 2026. Notebook 01 identifies both source files with SHA-256 digests. The mapping to the earlier filenames cannot be verified because those files are not present.

## Known fields in the event data

| Field | Initial interpretation | Validation required |
|---|---|---|
| `clicked_24h` | Whether a delivered event was clicked within 24 hours | Binary; use delivered events as the denominator |
| `engaged_24h` | Engagement within 24 hours | Define engagement and confirm event-time window |
| `app_opened_24h` | Source-reported app open within 24 hours | Every recorded open is also marked engaged, but 977 engaged events have no recorded app open; qualifying engagement actions remain undefined. |
| `converted_7d` | Source-reported conversion within seven days | Delivered-event denominator only. The final seven calendar days contain 4,396 delivered flags with incomplete windows relative to April 1; extraction time is unavailable. |
| `churn_target` | Validated view of 30-day future inactivity; raw version is invalid | In the raw source, observed values are all `1` and contradict future opens. The corrected view retains only evidence-backed `0` values and missing outcomes. Population modeling remains blocked. |
| `delivered` | Delivery indicator | Reconciles exactly with `suppression_reason` |
| `suppression_reason` | Reason event was suppressed | Missing for delivered events; `quiet_hours` or `daily_cap` for suppressed events |
| `notifications_prior_24h` | Delivered notifications in the preceding **one complete local calendar day**, `[event_date - 1 day, event_date)`; the suffix is historical naming, not a rolling 24-hour interval | Exactly reconstructs on all 85,019 rows |
| `notifications_prior_7d` | Delivered notifications in `[event_date - 7 days, event_date)` | Exactly reconstructs on all 85,019 rows |
| `notifications_prior_30d` | Delivered notifications in `[event_date - 30 days, event_date)` | Exactly reconstructs on all 85,019 rows |
| `fatigue_before_send` | Labeled pre-decision fatigue score | Construction cannot be reconstructed; allowed as a feature under an explicit timing assumption |
| `fatigue_after_event` | Post-event fatigue score | Excluded from predictive and policy features |
| `user_segment` | User segment | Confirm whether assigned before observation |
| `app_active_on_event_day` | Same-day activity flag | Timing within the day is unresolved; excluded from predictive and policy features |

The raw files do not contain the previously documented session-count, consecutive-ignore, previous-click-rate, delivery-status, or activity-frequency fields.

The exposure calculation excludes current-day deliveries and suppressed candidates. The timestamps are used as supplied local clock values; no timezone name, UTC offset or daylight-saving rule is present. The source's `daily_cap` suppression reason corresponds to a prior-day count of one in this dataset. Its intended business rule is unavailable, so this does not validate a conventional same-day cap. The notebook simulator uses the same calendar-day count contract.

## Phase 2: processed data lineage (F04)

The notebooks use a keyed projection of the verifiable engineered/profile fields from the two raw CSVs and validate `notification_features.csv` against raw fields and the canonical churn view. Hour sine/cosine use `2π × hour_local / 24`; 7- and 30-day rates divide their corresponding calendar-day counts by 7 and 30. `fatigue_high` is all zero here, but its threshold and generator are absent, so it remains unverified. `fatigue_before_send`, `fatigue_after_event`, inactive streak and baseline synthetic propensities are copied source inputs, not independently regenerated histories.

## Phase 2: 24-hour outcome maturity (F05)

The March 1 local-clock split is an as-of cutoff. Training requires `event_timestamp + 24 hours <= 2026-03-01 00:00`; evaluation in Notebook 03 requires `event_timestamp >= 2026-03-01` and `event_timestamp + 24 hours <= 2026-04-01 00:00`. Parsing errors and missing timestamps fail before assignment. No ingestion timestamp is supplied, so completion of a calendar window cannot prove the response was ingested by the cutoff. Opt-out timing is unknown; its fit uses the same conservative 24-hour eligibility gate. Simulated March opportunities remain the historical delivered candidates, including late-March records, and are not treated as measured mature outcomes.

Notebook 02 retains the all-delivered source conversion flag as a labeled source summary, then separately reports the subset whose seven-day window ends by the April 1 calendar boundary. Calendar completion is only a necessary check: source extraction time and individual conversion timestamps are absent. The 24-hour engagement/open flags on 607 delivered March 31 events have the same calendar limitation.

For interpretation, `channel` equals `preferred_channel` in every modeled row; `baseline_activity_probability` has one value per `user_segment`; `fatigue_high` is constant zero. The original policy models retain their validated inputs, while Notebook 03 reports a reduced-redundancy refit and a sensitivity model limited to channel, type, personalization, local hour and independently reconstructible prior-delivery counts. These are synthetic holdout checks, not proof that the other simulator fields were available online.

## Required validation in Notebook 01

- Print `head()`, `shape`, `info()`, `describe(include="all")`, missing-value counts, and duplicate counts.
- Check uniqueness of user IDs in the user table and `(user_id, event_timestamp)` event keys in the event table.
- Confirm all event user IDs map to the user table.
- Inspect timestamp minimum/maximum, ordering, and per-user event counts.
- Enumerate categories and target values.
- Inspect actual churn target values and missingness. This supplied source has no `-1` sentinel; positive labels are invalid and unsupported labels remain missing in the corrected view.
- Check whether feature fields are computed before or after the event/outcome.
- Record all confirmed definitions and unresolved assumptions here.

## Phase 1: canonical churn handling (F01)

Raw files retain their original bytes and must not be used directly for churn
modeling. `phase1.load_events()` constructs the canonical event view, and
`phase1.build_churn_artifacts()` writes `churn_label_validation.csv` and updates
only the churn columns/provenance in `notification_features.csv`.

| Field | Definition |
|---|---|
| `churn_target`, `no_app_open_next_30d` | Both are `0` only for supported negatives; otherwise missing. No positive inactivity label is inferred from absent activity. |
| `source_churn_target`, `source_no_app_open_next_30d` | Original values retained for audit provenance only; forbidden as predictive features or valid business outcomes. |
| `churn_status` | `verified_future_open_negative`, `unverified_incomplete_activity_history`, `source_missing_preserved`, or `censored_window`. |
| `churn_witness_event_timestamp` | Same-user event with `app_opened_24h=True`, strictly more than 24 hours after the candidate and at most 29 days later. |
| `churn_witness_latest_open` | Witness event plus 24 hours; the entire possible open window is inside the candidate's next 30 days. |
| `churn_window_complete_calendar` | Candidate plus 30 days is on/before the exclusive April 1 source calendar boundary. This does not assert complete activity logging. |
| `churn_model_eligible` | Always false for the supplied incomplete history; evidence-selected negatives do not form a representative modeling population. |

Source-missing values remain missing even if a witness exists. A current event's
own response is not used to infer a later return. Same-day activity flags are not
used for corrections because their intraday timing is unresolved. The separate
`user_retention.csv` is not substituted for a complete activity timeline.

## Phase 1: stateful simulation outputs (F02)

- `phase1_simulation_events.csv`: one terminal record per fixed March opportunity,
  including candidate/terminal/delivery times, initial action, final status,
  delivered/immediate counts, response flags and fatigue at the terminal attempt.
- `phase1_simulation_transitions.csv`: each chronological attempt, consent before
  and after, dynamic exposure, fatigue before and after, rescored probabilities,
  delivery/delay/suppression/expiry/cancellation and simulated responses.
- `phase1_simulation_users.csv`: one row per randomized user, retaining zero-send
  users. `engagement_rate` retains the original mean-user metric with an explicit
  zero-delivery-to-zero convention; it is a composite score in this case.
- `phase1_simulation_results.csv`: arm means, differences, user-bootstrap intervals,
  Cohen's d and two-sided Welch tests, conditional on the declared simulator.
- `phase1_policy_*.csv`: paired all-baseline/all-treatment stateful scenarios and
  their aggregate comparison, generated by Notebook 04 using the same engine.

Simulation exposure counters use the same previous-complete-calendar-day contract
as the fitted inputs, and update only on simulated delivery. This is necessary for
F02 state consistency and does not resolve the separate F03 source naming issue.
The fatigue transition approximation and scheduling assumptions are documented in
Notebooks 04 and 05.

The simulator requires complete binary initial opt-in, delivery and historical opt-out fields. Missing or string consent is rejected, not coerced to a truth value. Historical event keys must be unique, nonmissing and map to known profiles; historical notification opt-out requires a delivered record, and initial fatigue history must be finite and nonnegative. These input guards do not reconstruct unknown consent history or change the all-opted-in supplied profiles.

## Phase 3: common KPI contract and opt-out validation

Notebooks 04 and 05 use **event-weighted engagement efficiency** as the primary efficiency KPI: total simulated engagements divided by total delivered notifications. The unit resampled for uncertainty is the assigned user; the ratio is recomputed inside each bootstrap. Notebook 04 uses paired resamples of the same users under both policies; Notebook 05 resamples users independently within assigned arms.

**User-weighted efficiency among recipients** is the mean of each recipient's engagements/notifications. It is a secondary, selected-population diagnostic because policy changes who receives a notification. Individual ratios for zero-send users are missing, not zero. All assigned users remain in total and per-assigned-user outcomes, including engagements, notifications and opt-out risk. A whole group or bootstrap sample with no notifications has undefined efficiency, and exports count finite bootstrap replicates.

Fields ending in `expected_*_on_path` sum the pre-draw engagement probabilities at actual simulated deliveries and use the same denominators as their simulated counterparts. They are probability-sum summaries along one realized trajectory, not marginal expectations over alternative consent/fatigue paths or a conditional expectation given the entire realized future trajectory. Their intervals resample users conditional on those paths.

`phase3_*` files contain the current KPI definitions. The older `phase1_simulation_users.csv` `engagement_rate` (zero-filled composite) and `phase1_simulation_results.csv` are retained solely for Phase 1–2 regression compatibility; they are not the Phase 3 primary KPI.

`phase3_optout_*` files record mature temporal predictions, ROC AUC, average precision, Brier loss, observed/predicted rates, calibration gap, five calibration bins and precision-recall thresholds. Intervals use 500 paired user-cluster resamples with the fitted models held fixed. Baselines are pre-split prevalence and always-zero risk. The user risk product `1 - product(1-p_event)` is explicitly a conditional-independence approximation on historical exposures. The four opt-out simulation mechanisms use the same assigned users and seed: fitted logistic, constant training prevalence, half logistic and double logistic clipped at one. The multipliers are stress assumptions, not fitted estimates.

The historical user-risk product is retrospective: an unsubscribe can reduce the number of later observed exposures, so that full-period exposure count is outcome-dependent. It is not a deployable user-risk forecast or evidence of prospective user-model calibration. Direct randomized-user opt-out comparisons use all assigned users and report risk differences and Fisher's exact equality test.
