# 07 — Verification (Passive)

**Purpose:** Prove savings **without write privileges**. Detect that humans acted on recommendations; measure with difference-in-differences; maintain a savings ledger and ignored-recommendation cost.

---

## Design constraint

SnowLens v1 is **strictly advisory**. No `MODIFY` / `OPERATE` / ALTER execution.

Verification still works because:

1. Every ingest snapshots `SHOW WAREHOUSES` (config history you own).  
2. `WAREHOUSE_EVENTS_HISTORY` records RESIZE / SUSPEND / RESUME / cluster events.  
3. Query-shape cadence changes show up in `QUERY_HISTORY` / aggregates.  
4. Storage drops show up in object + storage metrics (via Tech Debt Radar signals).

---

## Opportunity status machine

```text
recommended
    │
    ├─► dismissed (user) / expired (TTL)
    │
    ├─► observed          ← passive detector matched expected change
    │       │
    │       ├─► verifying (waiting for post window)
    │       │
    │       ├─► verified_positive
    │       ├─► verified_zero      ← CI spans 0
    │       └─► verified_negative  ← cost got worse (rare; investigate)
    │
    └─► stale              ← underlying metrics no longer support card
```

---

## Passive change detectors

Match **expected fingerprint** stored on the opportunity at emit time.

| Opportunity type | Detector |
|------------------|----------|
| `IDLE_AUTOSUSPEND` | Snapshot: `auto_suspend` equals recommended `T*` (or within 30s grid) |
| `RIGHTSIZE_DOWN/UP` | Snapshot size **or** `RESIZE_WAREHOUSE` event to target size |
| `MULTICLUSTER_MAX_REDUCTION` | Snapshot `max_cluster_count` ≤ recommended |
| `ZOMBIE_WAREHOUSE` | Sustained suspend / no resumes / warehouse dropped |
| `CONSOLIDATE_WAREHOUSES` | Member warehouses dropped or zero traffic; target absorbs roles/queries |
| `UPTIME_ANCHOR` / `RELENTLESS` | Shape E1 drops ≥ threshold % or hash disappears |
| `FAILED_RETRY_WASTE` | Fail credits for shape → 0 |
| `FEATURE_NEGATIVE_ROI` / `DEAD_BUT_MAINTAINED` | Maintenance credits → 0; feature disabled |
| Storage types | Object absent / bytes reclaim path started |

Store on opportunity:

```text
expected_change = {
  kind: "config" | "event" | "metric",
  match: {...},
  observed_at: null
}
```

**Ambiguity:** If change happens but doesn’t match recommendation (e.g. resize to different size), mark `observed_unrelated` — do not claim verification credit for SnowLens.

---

## Observation window

| Parameter | Default | Notes |
|-----------|---------|-------|
| `pre_days` | 14–28 | Before observed_at |
| `post_days` | 14–28 | After; respect ACCOUNT_USAGE latency |
| `min_post_days` | 7 | Before computing verified |
| Exclude | Change day ±1 | Transition noise |

---

## Control group selection

Naive before/after is wrong (quiet month, org-wide seasonality).

**Controls:** warehouses (or shapes) that:

1. Same **F1** workload class (or closest).  
2. Similar **F4** diurnal profile (cosine similarity / bucket corr).  
3. **Not** changed in the window (config snapshot stable; no RESIZE).  
4. Not in the same consolidation member set.  
5. Prefer same size tier.

Need ≥ N controls (default 3); else fall back to DoW-stratified robust z before/after with **lower confidence** and label `verified_weak`.

---

## Difference-in-differences (J8)

Metric `Y` = daily compute credits (or size-adjusted compute-seconds for query regressions).

```text
verified =
  (Y_treated_post − Y_treated_pre)
  − (Y_control_post − Y_control_pre)
```

Sign convention: **savings positive** when treated credits fall more than controls.

```text
verified_savings = − verified_delta_credits   # if delta is post−pre change in spend
```

Bootstrap over days for CI.  

- If 90% CI spans 0 → record **verified_zero** (honest).  
- Never label estimated as verified.

For consolidation, `Y` = sum of member credits vs synthetic control of similar multi-WH groups when possible.

---

## Savings ledger (logical schema)

| Column | Meaning |
|--------|---------|
| `ledger_id` | |
| `opportunity_id` | |
| `opportunity_type` | |
| `estimated_p10/p50/p90` | At emit |
| `verified_point` | DiD point |
| `verified_p10/p90` | |
| `status` | verified_* |
| `pre_start/pre_end` | |
| `post_start/post_end` | |
| `control_ids` | |
| `run_id` | Computation provenance |

**Credibility rule:** Exec materials only cite **verified** and **portfolio estimated (de-duplicated)** as separate lines.

---

## Realization rate (J7)

Per `opportunity_type`:

```text
realization_rate = sum(verified_point) / sum(estimated_p50)
```

only over opportunities that reached a verified_* terminal state (exclude dismissed).

Use to calibrate future estimates ([05](05-savings-accounting.md)).

---

## Ignored recommendation cost

Politically useful under advisory-only constraint:

```text
ignored_cost(t) =
  sum( estimated_p50 of opportunities
       where status = recommended
         and age_days > ignore_ttl
         and not observed )
```

Also track **realized ignore**: opportunities that expired while underlying waste continued (re-estimated each run).

Surface on FinOps home: “Open recommendations older than 30d imply ~X credits/month still on the table.”

---

## What we do **not** claim

| Claim | Why forbidden |
|-------|----------------|
| “We saved $X” without observed change | No causality |
| Before/after without controls | Seasonality |
| Verified from attributed query credits alone after resize | Confounded by size |
| Storage free on DROP day | Fail-safe delay |

---

## Implementation sketch

1. On each analyze run, load open opportunities with `expected_change`.  
2. Run detectors against latest snapshots/events/metrics.  
3. Transition to `observed`; schedule verification job when `post_days` satisfied.  
4. Select controls; compute DiD; write ledger; update J7.  
5. Recompute ignored_cost.

No Snowflake writes required — only reads + SnowLens Postgres writes.
