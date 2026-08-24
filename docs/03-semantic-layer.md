# 03 — Semantic Layer

**Purpose:** Define the warehouse-second interval ledger and the Counterfactual Uptime Simulator — the single engine under almost every compute opportunity.

**Access tier assumption:** Tier 1 (`USAGE_VIEWER`) for events, metering, query history. Config snapshots need live `SHOW WAREHOUSES` (see Tier notes in [11](11-access-tiers-and-degradation.md)).

---

## Why this exists

Five catalog families look like five pipelines:

| Family need | Without a shared core | With this layer |
|-------------|----------------------|-----------------|
| Optimal auto-suspend (B13) | Custom gap cost function | Replay gaps with candidate `T` |
| Uptime-anchor shape (E7) | Ad-hoc “would warehouse sleep?” | Replay with one hash removed |
| Consolidation (C4) | Separate interval algebra | Replay union of members |
| Right-sizing (B14) | Different credit rate | Replay same intervals × new rate |
| Fair chargeback (A9) | Pro-rata join spaghetti | Allocate ledger credits to occupants |

They all ask: **which billed seconds would exist under hypothesis H?**

---

## Core objects

### 1. Warehouse (dimension)

- Identity: account, warehouse name/id  
- Config over time: size, `auto_suspend`, `auto_resume`, `min_cluster_count`, `max_cluster_count`, `scaling_policy`, type (standard / Snowpark / Gen2 / adaptive)  
- Source: **snapshot of `SHOW WAREHOUSES` every ingest** + reconstruction from `WAREHOUSE_EVENTS_HISTORY` (`RESIZE_WAREHOUSE`, cluster spin events)

There is **no** `ACCOUNT_USAGE.WAREHOUSES` config history. Without snapshots, past recommendations cannot be explained.

### 2. Uptime interval (fact)

One row (or segment) =

| Field | Meaning |
|-------|---------|
| `warehouse_id` | |
| `cluster_number` | 0 for single-cluster; else cluster id |
| `start_ts`, `end_ts` | Inclusive/exclusive convention fixed in DDL |
| `size_at_interval` | Credits/hour multiplier applicable |
| `credits_reconciled` | Share of metering attributed to this interval |
| `source` | `EVENTS` reconstructed, optionally adjusted |

**Grain:** warehouse × cluster × contiguous billed uptime segment.

### 3. Occupancy (fact / bridge)

Which query activity occupied which seconds:

| Field | Meaning |
|-------|---------|
| `interval_id` or time range | |
| `query_id` / `query_parameterized_hash` | Prefer hash for aggregation |
| `overlap_seconds` | Intersection of query exec window with uptime |
| `role_name`, `user_name` | Optional attribution |

Source: `QUERY_HISTORY` start/end (or `AGGREGATE_QUERY_HISTORY` at coarser grain for scale).

### 4. Busy bucket (derived)

From `WAREHOUSE_LOAD_HISTORY` 5-minute buckets where `AVG_RUNNING > 0` (and optionally queue metrics). Used for consolidation temporal overlap (C1/C3) — **busy ≠ billed uptime**, but both matter.

---

## Interval reconstruction algorithm

### Inputs

- `WAREHOUSE_EVENTS_HISTORY`: `RESUME_WAREHOUSE`, `SUSPEND_WAREHOUSE`, `SPINUP_CLUSTER`, `SPINDOWN_CLUSTER`, `RESIZE_WAREHOUSE` (and related)
- `WAREHOUSE_METERING_HISTORY`: hourly `credits_used_compute`, `credits_attributed_compute_queries`, warehouse size if present
- Optional: load history for busy sets

### Steps

1. **Order events** per warehouse (and cluster where applicable) by timestamp.
2. **Build open intervals** on RESUME / SPINUP; close on SUSPEND / SPINDOWN.
3. **Apply 60-second minimum:** for each resume→suspend span shorter than 60s of *billable* policy, extend billed end to `start + 60s` (or encode as billing rule in simulator — match Snowflake’s per-resume minimum).
4. **Split on RESIZE** so size (credit rate) is constant within a segment.
5. **Multi-cluster:** billable cost ∝ sum of active cluster-time × size rate. Never treat “warehouse hours” as single-cluster when `clusters_active > 1`.
6. **Reconcile to metering:**  
   `scale = SUM(metering.credits_used_compute) / SUM(interval_theoretical_credits)` over the same window.  
   Store both theoretical and reconciled credits. Prefer reconciled for external claims; keep theoretical for debugging.
7. **Handle open intervals** at window edges (warehouse still running at extract cutoff).
8. **Adaptive warehouses:** if `credits_attributed_compute_queries` is NULL, mark attribution path unavailable; still build uptime from events/metering; use `QUERY_METERING_HISTORY` where needed for query credits.

### Pseudocode

```text
for each warehouse W:
  state = SUSPENDED
  open = None
  for event in events(W) sorted by time:
    if RESUME and state == SUSPENDED:
      open = Interval(start=event.ts, size=current_size, clusters=...)
      state = RUNNING
    if SUSPEND and state == RUNNING:
      open.end = max(event.ts, open.start + 60s)  # billing floor
      emit(open); open = None; state = SUSPENDED
    if RESIZE:
      close_and_emit_partial(open, event.ts)
      open = Interval(start=event.ts, size=new_size, ...)
    if SPINUP/SPINDOWN:
      adjust cluster dimension; split intervals as needed
  reconcile(emitted, metering[W])
```

### Edge cases

| Case | Handling |
|------|----------|
| Missing SUSPEND | Close at window end; flag `open_ended` |
| Event gaps / latency | Do not invent events; widen uncertainty in J2 |
| Multi-cluster concurrent | Separate cluster intervals; sum credits |
| Size change mid-hour | Split; allocate metering by theoretical share |
| Cloud services | Out of scope of warehouse-second ledger (A3 separate) |

---

## Occupancy attribution

For each query (or shape aggregate):

```text
overlap = | [query_start, query_end) ∩ [interval_start, interval_end) |
```

Rules:

- Sub-~100ms / excluded from attribution views: still can occupy uptime for E7 logic if present in `QUERY_HISTORY`.
- Concurrent queries: seconds can be occupied by multiple shapes; for **chargeback (A9)** allocate interval credits **pro-rata by overlap_seconds** (or exclusive time if using a partition of the timeline).
- For **E7 uptime anchor**: a second is “caused by shape S” if removing all executions of S would allow suspend under current (or optimal) auto_suspend policy — computed via simulator, not by simple overlap.

---

## Counterfactual Uptime Simulator

### Interface

```text
simulate(
  warehouse_ids: set,
  hypothesis: Hypothesis,
  window: [t0, t1),
  policy: SimulationPolicy
) -> SimulationResult
```

`SimulationResult`:

- `baseline_credits`, `counterfactual_credits`
- `delta_credits` (p10/p50/p90 via bootstrap over days)
- `claimed_interval_slices` (for uptime budget)
- `diagnostics` (resume counts, idle seconds, queue risk flags)

### Hypothesis types (pluggable)

| Hypothesis | Parameters | Method |
|------------|------------|--------|
| `ChangeAutoSuspend` | `T` seconds | From query timeline, rebuild idle gaps; bill `min(g,T)` + `60×#{g>T}` + exec time (catalog §5.3) |
| `RemoveShape` | `query_parameterized_hash` | Drop those executions; recompute gaps / suspends |
| `ThinCadence` | hash, keep_ratio or min_interval | Thin executions; recompute |
| `MergeWarehouses` | member set → target config | Busy/uptime union; concurrency check external (C6) |
| `ChangeSize` | target size rank | Same intervals × new credit rate; latency risk **not** simulated here — Gate/Evidence from B11/B14 |
| `ChangeMaxClusters` | new max | Cap cluster-hours at observed need vs new max |
| `AllocateOccupants` | allocation rule | No delta; produces A9 shares |

### Auto-suspend cost function (exact)

Given sorted inter-query gaps `G` (and query execution seconds constant w.r.t. `T`):

```text
billed_idle_and_resume(T) =
    Σ_{g ∈ G} min(g, T)
  + 60 × |{g ∈ G : g > T}|
```

Minimize over `T ∈ {30, 60, …, 900}`.  
Snowflake suspension polling ~30s → non-multiples of 30 are not meaningfully different.

**Caveats encoded as Gates/Evidence, not in the sum:**

- Local SSD cache loss after suspend (high cache % → discourage aggressive T).
- Resume latency for interactive SLA (F1/F3).

### Consolidation arithmetic (exact)

```text
saving_seconds = Σ_i |uptime_i| − |⋃_i uptime_i|
saving_credits ≈ saving_seconds × rate(target_size)   # careful with heterogeneous sizes
```

Prefer **low** temporal overlap of **busy** sets with **high** workload affinity (C7). Overlapping busy periods save little and may queue.

### Bootstrap for ranges

1. Partition window into days (or DoW-aware strata).  
2. Resample days with replacement.  
3. Recompute delta each sample.  
4. Report p10–p90.

---

## Reconciliation identity (publish always)

```text
credits_used_compute
  ≈ credits_attributed_compute_queries  +  idle_credits
```

(From single view `WAREHOUSE_METERING_HISTORY` when columns present.)

```text
A8 coverage = SUM(attributed) / SUM(used_compute)
```

Idle is not an error. It is often the largest actionable pool.

---

## Mapping opportunities → simulator

| Opportunity | Hypothesis |
|-------------|------------|
| `IDLE_AUTOSUSPEND` | `ChangeAutoSuspend(T*)` |
| `ZOMBIE_WAREHOUSE` | Extreme: suspend always / remove warehouse uptime (with Gate) |
| `UPTIME_ANCHOR_SHAPE` | `RemoveShape(hash)` |
| `RELENTLESS_SHAPE` / cadence | `ThinCadence` |
| `CONSOLIDATE_WAREHOUSES` | `MergeWarehouses` |
| `RIGHTSIZE_DOWN` / `UP` | `ChangeSize` (+ Gates) |
| `MULTICLUSTER_MAX_REDUCTION` | `ChangeMaxClusters` |
| `RESUME_THRASH` | Often same as auto-suspend / cadence fix |
| Chargeback views | `AllocateOccupants` |

---

## Scale strategy (unknown estate size)

**Default:** pre-aggregate in Snowflake before extract.

| Object | Preferred Snowflake grain for large estates |
|--------|-----------------------------------------------|
| Query occupancy | `AGGREGATE_QUERY_HISTORY` by hash × hour + sampled raw for gaps |
| Gaps for B13 | Per-warehouse query start/end **only for warehouses above spend floor** |
| Load | Already 5-min in `WAREHOUSE_LOAD_HISTORY` |
| Events | Full event stream (usually manageable) |

Postgres holds normalized ledger + opportunities; not forever raw `QUERY_HISTORY` for every account day.

---

## Implementation notes

- Store **immutable run_id** on every ledger build (see [09](09-data-model.md)).
- Simulator must be **pure** given ledger snapshot + hypothesis (reproducible).
- Never claim dollar savings without A2 or explicit rate card.
- Never treat `AVG_RUNNING` as utilization % when interpreting Evidence.

---

## Non-goals of this layer

- Predicting query latency under resize (that is B14 evidence class: natural experiment / spill heuristic).
- Parsing SQL text.
- Storage bytes (H-family / Tech Debt Radar).
- Cloud services optimization beyond A3 honesty.
