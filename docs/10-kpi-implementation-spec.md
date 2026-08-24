# 10 — KPI Implementation Spec

**Purpose:** How to compute each KPI that survives into the product: sources, grain, window, formulas, NULLs, edge cases.

Canonical definitions remain in [`kpi-catalog.md`](../kpi-catalog.md). This doc is the **engineering contract**.

**Default windows:** compute 90d; access/storage coldness up to 365d; configurable in stack YAML.

---

## Global rules

1. Prefer columns from a **single view** when possible (e.g. idle from metering).  
2. Percentile ranks: **within account**, not min-max.  
3. Adaptive warehouses: `CREDITS_ATTRIBUTED_COMPUTE_QUERIES` may be NULL → set `attribution_available=false`; use `QUERY_METERING_HISTORY` for query credits where needed.  
4. Cloud services: never flag spend inside free allowance (A3).  
5. Sub-100ms / excluded queries: attributed sum ≠ bill (expected).  
6. Every KPI row stores `run_id`, `period_start`, `period_end`.

---

## Family A — Foundation

### A1 Total billed credits

| | |
|--|--|
| **Source** | `SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY` |
| **Columns** | `SERVICE_TYPE`, `CREDITS_USED`, `START_TIME` |
| **Grain** | account × day × service_type |
| **Formula** | `SUM(credits_used)` |
| **NULL** | None expected |
| **Edges** | Include all service types for reconciliation |

### A2 Effective credit price

| | |
|--|--|
| **Source** | `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY`, `RATE_SHEET_DAILY` — else YAML rate card |
| **Formula** | Trailing 30d weighted mean of currency/usage for compute |
| **Degrade** | Manual `usd_per_credit`; mark `price_source=manual` |

### A3 Cloud services billable

| | |
|--|--|
| **Source** | `METERING_DAILY_HISTORY` |
| **Columns** | `CREDITS_USED_CLOUD_SERVICES`, `CREDITS_ADJUSTMENT_CLOUD_SERVICES` |
| **Formula** | `credits_used_cloud_services − credits_adjustment_cloud_services` (report ≥ 0 remainder) |

### A4 Spend by warehouse

| | |
|--|--|
| **Source** | `WAREHOUSE_METERING_HISTORY` |
| **Columns** | `WAREHOUSE_NAME`, `CREDITS_USED`, `CREDITS_USED_COMPUTE`, `START_TIME` |
| **Grain** | warehouse × day |

### A8 Attribution coverage

| | |
|--|--|
| **Source** | `WAREHOUSE_METERING_HISTORY` |
| **Columns** | `CREDITS_USED_COMPUTE`, `CREDITS_ATTRIBUTED_COMPUTE_QUERIES` |
| **Formula** | `SUM(attributed) / NULLIF(SUM(used_compute),0)` |
| **NULL** | If attributed NULL (adaptive): coverage NULL; publish `attribution_available=false` |
| **Always** | Surface on summary UI |

### A9 Fully-loaded query cost

| | |
|--|--|
| **Inputs** | Attributed credits per shape/user + idle from A8 gap |
| **Formula** | `attributed + idle × (attributed / SUM(attributed))` within warehouse×period |
| **Gate** | Tag coverage for team rollup |

---

## Family B — Warehouse efficiency

### B1 Idle credits / B2 Idle share

| | |
|--|--|
| **Source** | `WAREHOUSE_METERING_HISTORY` |
| **Formula** | `B1 = used_compute − attributed_compute_queries`; `B2 = B1 / used_compute` |
| **Signal** | B2 ≥ 0.30 and B1 ≥ floor |

### B3–B6 Load metrics

| | |
|--|--|
| **Source** | `WAREHOUSE_LOAD_HISTORY` |
| **Columns** | `AVG_RUNNING`, `AVG_QUEUED_LOAD`, `AVG_QUEUED_PROVISIONING`, `AVG_BLOCKED`, `START_TIME` |
| **Grain** | warehouse × 5 min |
| **Caveat** | `AVG_RUNNING` is load ratio, can be > 1; **not** CPU % |

### B7 Uptime hours / B8 Resume count / B9 Cluster-hours

| | |
|--|--|
| **Source** | `WAREHOUSE_EVENTS_HISTORY` + ledger |
| **B8** | `COUNT(*)` where event = `RESUME_WAREHOUSE` |
| **B7/B9** | From `fact.warehouse_uptime_interval` |

### B10 Max cluster headroom

| | |
|--|--|
| **Sources** | Config snapshot `max_cluster_count` + observed p99 clusters from events/load |
| **Formula** | `max_cluster_count − p99(clusters_active)` |

### B11 / B12 Spill rates

| | |
|--|--|
| **Source** | `QUERY_HISTORY` (or agg) |
| **Columns** | `BYTES_SPILLED_TO_REMOTE_STORAGE`, `BYTES_SPILLED_TO_LOCAL_STORAGE`, `BYTES_SCANNED` |
| **Formula** | `SUM(remote_spill) / NULLIF(SUM(bytes_scanned),0)` |
| **Gate** | Remote spill > ε blocks RIGHTSIZE_DOWN |

### B13 Optimal auto_suspend

| | |
|--|--|
| **Inputs** | Per-warehouse query start/end times; current auto_suspend from snapshot |
| **Method** | Gap set `G`; minimize billed idle+resume over T∈{30…900}; see [03](03-semantic-layer.md) |
| **Output** | `T_star`, `delta_credits` vs current |

### B14 Right-size verdict

| | |
|--|--|
| **Evidence hierarchy** | (1) RESIZE natural experiments DiD (2) same hash across sizes (3) spill/queue heuristic |
| **Output** | direction, expected credit Δ band, latency bound, confidence_class |
| **Never** | Naked “saves $X” without confidence |

---

## Family C — Consolidation

### C1 Busy interval set

| | |
|--|--|
| **Source** | `WAREHOUSE_LOAD_HISTORY` |
| **Rule** | Buckets with `AVG_RUNNING > 0` → `is_busy` |

### C2 Uptime interval set

From ledger (`fact.warehouse_uptime_interval`).

### C3 Temporal overlap

| | |
|--|--|
| **Formula** | `\|busy(A) ∩ busy(B)\| / \|busy(A) ∪ busy(B)\|` (Jaccard on busy time) |
| **Signal** | Low overlap preferred for merge |

### C4 Consolidation saving

| | |
|--|--|
| **Formula** | `(Σ \|uptime_i\| − \|⋃ uptime_i\|) × credits_per_hour(target)` |
| **Compose** | Prefer post-autosuspend intervals when enabled |

### C5 / C6 Concurrency

| | |
|--|--|
| **C5** | `p99(Σ AVG_RUNNING over members aligned in time)` |
| **C6** | `C5 ≤ capacity(target_size, max_clusters)` — capacity model in YAML (conservative) |
| **Hard gate** | C6 false → suppress |

### C7 Workload affinity

| | |
|--|--|
| **Source** | `ACCESS_HISTORY` table sets per WH (needs GOVERNANCE) |
| **Degrade** | Jaccard on query_tag / database names from QUERY_HISTORY |
| **Formula** | Jaccard(tables_A, tables_B) |

### C8 Role affinity / C9 SLA / C10 Size

| | |
|--|--|
| **C8** | Overlap of `ROLE_NAME` sets from QUERY_HISTORY |
| **C9** | F1 classes mix interactive+batch → conflict |
| **C10** | Size rank spread among members |

---

## Family D — Query efficiency

### D1 Credits per execution

`QUERY_ATTRIBUTION_HISTORY.CREDITS_ATTRIBUTED_COMPUTE` (or query metering for adaptive).

### D2 Scan-to-output

`BYTES_SCANNED / NULLIF(BYTES_WRITTEN_TO_RESULT,0)` — exclude intentional heavy `QUERY_TYPE`s.

### D3 Pruning ratio

`PARTITIONS_SCANNED / NULLIF(PARTITIONS_TOTAL,0)`.

### D4 Remote spill ratio

Per query or shape aggregate.

### D9 / D10 Fail / retry

Join attribution to `QUERY_HISTORY` where `EXECUTION_STATUS = 'FAIL'`; include `QUERY_RETRY_TIME` share.

### D11 Native insights

| | |
|--|--|
| **Source** | `QUERY_INSIGHTS` |
| **Columns** | `INSIGHT_TYPE_ID`, `MESSAGE`, `SUGGESTIONS`, `IS_OPPORTUNITY`, `QUERY_ID` |
| **Use** | Attach to HEAVY_SHAPE; price externally — do not re-detect |

### D13 Clustering key candidate

| | |
|--|--|
| **Source** | `COLUMN_QUERY_PRUNING_HISTORY` |
| **Logic** | Rank columns by pruning contribution; join AC cost |

---

## Family E — Recurring

### E1–E4

| | |
|--|--|
| **Sources** | `AGGREGATE_QUERY_HISTORY`, `QUERY_ATTRIBUTION_HISTORY` grouped by `QUERY_PARAMETERIZED_HASH` |
| **Filters** | E1 ≥ 10, E2 ≥ 1.0 credit |
| **Archetypes** | Heavy / Relentless / Both / Polling / Uptime anchor |

### E5 Cadence regularity

Stddev of inter-execution intervals (low → cron-like).

### E6 Result-identical repeats

Join shape runs to `TABLE_DML_HISTORY` on accessed tables; no DML between runs → candidate.

### E7 Uptime attributable to shape

Simulator `RemoveShape(hash)` Δ credits — **primary Driver** for UPTIME_ANCHOR.

---

## Family F — Classification

### F1 Workload class

Rule cascade / k-means on: `QUERY_TYPE` mix, arrival regularity, client app (`SESSIONS` if available), R/W ratio, roles, `IS_CLIENT_GENERATED_STATEMENT`.

Classes: `ETL`, `BI`, `ADHOC`, `ML`, `BATCH`, `MIXED`.

### F2 Class purity

Share of dominant class per warehouse.

### F4 Diurnal profile

Credits by hour×DoW from metering — used for controls + consolidation.

### F5 Burstiness

`p99(load)/median(load)` on busy buckets.

---

## Family G — Anomaly

### G1 Daily cost anomaly

Robust z on DoW strata: `(x − median) / (1.4826 × MAD)`; flag \|z\| > 3.5.

### G2–G5

Changepoint (PELT) on shape credits / bytes / frequency; normalize for size and table growth (G7).

### G6

Temporal join changepoints to `WAREHOUSE_EVENTS_HISTORY`.

---

## Family H — Storage (compute in Radar or light mirrors)

### H1 True reclaimable bytes

`max(0, active + time_travel + failsafe − retained_for_clone)` from `TABLE_STORAGE_METRICS`.

### H5 Coldness

Days since last read from `ACCESS_HISTORY` (governance).

### H13 Read-probability

Kaplan-Meier on read-gap distributions — prefer over binary dead.

---

## Family I — Feature ROI

| Feature | Cost view | Benefit view |
|---------|-----------|--------------|
| Automatic clustering | `AUTOMATIC_CLUSTERING_HISTORY` | Pruning improvement vs baseline |
| Search optimization | `SEARCH_OPTIMIZATION_HISTORY` | `SEARCH_OPTIMIZATION_BENEFITS` |
| Query acceleration | `QUERY_ACCELERATION_HISTORY` | `QUERY_ACCELERATION_ELIGIBLE` |
| Materialized views | `MATERIALIZED_VIEW_REFRESH_HISTORY` | Avoided query credits (modelled) |

**I1** `(saved − spent) / spent`  
**I2** cold ∩ maintenance > 0  

---

## Family J — Meta (computed in app)

J1–J9 as in catalog; implementation in arbitrate / verify / rank modules — not Snowflake extracts.

---

## Minimum extract column sets (phase 1)

**Must have for idle + consolidation:**

- `WAREHOUSE_METERING_HISTORY`: warehouse, start, credits_used, credits_used_compute, credits_attributed_compute_queries  
- `WAREHOUSE_LOAD_HISTORY`: warehouse, start, avg_running, avg_queued_* , avg_blocked  
- `WAREHOUSE_EVENTS_HISTORY`: timestamp, warehouse, event_name, and size/cluster fields as available  
- `QUERY_HISTORY` slim (for top warehouses): warehouse, start, end, hash, role, bytes spill/scan, status  
- `SHOW WAREHOUSES` snapshot  

**Strongly preferred:** attribution by hash, aggregate query history.

**Later:** ACCESS_HISTORY, org currency, insights, pruning, feature histories.

Exact role mapping: [16](16-kpi-source-access-matrix.md).
