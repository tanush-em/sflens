# SnowLens — KPI Catalog & Data Foundation

**Purpose:** the complete list of measurable signals available from Snowflake telemetry, what each one actually means, how to compute it, where the data comes from, which statistical treatment turns it into a decision, and where it will mislead you.

**How to use this document:** read Part 1 first. Almost every wrong Snowflake cost KPI comes from not internalising the billing model. Then treat Parts 4–13 as a menu — pick the KPI families that match the product you want, not all of them.

**Verified against Snowflake documentation, August 2026.** Where a view is new or its semantics are commonly misread, the caveat is called out inline.

---

## Part 1 — The unit economics you must encode first

Every KPI below is downstream of five facts. If the model gets these wrong, the numbers will be confidently incorrect.

### 1.1 You are billed for warehouse *uptime*, not for queries

A warehouse bills per second while it is running, at a rate set by its size, with a **60-second minimum every time it resumes**. Queries do not have a price. They have an *allocated share* of the uptime they happened to occupy.

| Size | Credits / hour | Size | Credits / hour |
|---|---|---|---|
| X-Small | 1 | 2X-Large | 32 |
| Small | 2 | 3X-Large | 64 |
| Medium | 4 | 4X-Large | 128 |
| Large | 8 | 5X-Large | 256 |
| X-Large | 16 | 6X-Large | 512 |

Snowpark-optimized, Gen2, and adaptive warehouses use different multipliers. Do not hardcode this table — derive rates from `ORGANIZATION_USAGE.RATE_SHEET_DAILY` where you have it, and treat the table above as a fallback.

**The consequence that drives the whole product:** the marginal cost of running a query on a warehouse that is *already awake* is approximately zero. This single fact means

- "expensive query" is a misleading frame; the right frame is "which queries cause warehouse-seconds to exist"
- consolidation can be a large win, because merging two warehouses whose busy periods overlap eliminates the duplicated uptime
- shaving 20% off a query's runtime saves nothing if the warehouse stays up anyway for the next query

### 1.2 Multi-cluster warehouses bill per running cluster

A Large warehouse with 3 active clusters bills 24 credits/hour, not 8. Any sizing or idle calculation that ignores cluster count will understate cost by an integer factor.

### 1.3 Cloud services are free below a threshold

Cloud services credits are only billed to the extent they exceed **10% of the day's compute credits**. `METERING_DAILY_HISTORY.CREDITS_ADJUSTMENT_CLOUD_SERVICES` carries this adjustment. Flagging cloud services spend that is inside the free allowance is a fast way to lose credibility with a platform team.

### 1.4 Attributed credits are an allocation, not a measurement

`QUERY_ATTRIBUTION_HISTORY.CREDITS_ATTRIBUTED_COMPUTE` deliberately **excludes warehouse idle time**, excludes queries under ~100ms, excludes cloud services, storage, data transfer and serverless, is NULL for adaptive warehouses, and has up to ~6–8h latency. The sum of attributed credits will never equal the bill. That gap is not an error — it is the idle time, and it is one of your most valuable findings.

### 1.5 Storage is roughly an order of magnitude cheaper than people assume

On-demand storage runs in the region of $23–45 per TB per month depending on cloud, region and contract. A 40 TB redundant dataset is on the order of $1K/month. Real, but a single oversized X-Large running 24/7 is ~$25K/month. **Size your engineering effort to the money.**

---

## Part 2 — Data source map

### 2.1 Access layers

| Layer | Schema | What it gives you | Role needed |
|---|---|---|---|
| Organization | `SNOWFLAKE.ORGANIZATION_USAGE` | Cross-account rollups, **actual currency**, contract rates | `ORGANIZATION_USAGE_VIEWER` / `ORGANIZATION_BILLING_VIEWER` |
| Account — usage | `SNOWFLAKE.ACCOUNT_USAGE` | Queries, warehouses, metering, storage | `USAGE_VIEWER` |
| Account — objects | `SNOWFLAKE.ACCOUNT_USAGE` | Tables, columns, dependencies, tags | `OBJECT_VIEWER` |
| Account — governance | `SNOWFLAKE.ACCOUNT_USAGE.ACCESS_HISTORY` | Who read/wrote which object and column | `GOVERNANCE_VIEWER` (Enterprise Edition) |
| Live config | `SHOW WAREHOUSES` + `RESULT_SCAN` | auto_suspend, min/max clusters, scaling policy | `MONITOR` on warehouse, or admin-run snapshot |
| Near-real-time | `INFORMATION_SCHEMA` table functions | Last 7 days, no latency | Warehouse `MONITOR` |

### 2.2 The gap that will bite you

**There is no `ACCOUNT_USAGE.WAREHOUSES` view.** Warehouse *configuration* — `auto_suspend`, `min_cluster_count`, `max_cluster_count`, `scaling_policy`, `enable_query_acceleration` — is **not in ACCOUNT_USAGE at all**. It is only available from `SHOW WAREHOUSES`.

This has real architectural consequences:

- You cannot compute "current auto_suspend vs optimal auto_suspend" from historical telemetry alone.
- `SHOW WAREHOUSES` is a point-in-time snapshot with no history, so you must **snapshot it on every ingestion run and store your own config history** — otherwise you can never explain why cost changed on the day someone resized a warehouse.
- `WAREHOUSE_EVENTS_HISTORY` partially compensates: it records `RESIZE_WAREHOUSE`, `SUSPEND_WAREHOUSE`, `RESUME_WAREHOUSE`, `SPINUP_CLUSTER`, `SPINDOWN_CLUSTER` events, so you can reconstruct size and cluster timelines retroactively. Use both.

### 2.3 Core view inventory

Latency and retention are the operative constraints on what your product can promise.

| View | Grain | Latency | Retention | Primary use |
|---|---|---|---|---|
| `WAREHOUSE_METERING_HISTORY` | warehouse × hour | 3h (6h for cloud services) | 365d | **Billing anchor.** Includes `CREDITS_ATTRIBUTED_COMPUTE_QUERIES` |
| `WAREHOUSE_LOAD_HISTORY` | warehouse × **5 min** | 3h | 365d | Concurrency, queuing, consolidation |
| `WAREHOUSE_EVENTS_HISTORY` | event | 3h | 365d | Uptime intervals, resize natural experiments |
| `METERING_HISTORY` | service × hour | ~2h | 365d | Total bill by service type |
| `METERING_DAILY_HISTORY` | day | ~2h | 365d | Cloud services adjustment, billed credits |
| `QUERY_HISTORY` | query | 45 min | 365d | Query features, the workhorse |
| `AGGREGATE_QUERY_HISTORY` | query shape × hour | — | 365d | Same signal at 1/1000 the row count |
| `QUERY_ATTRIBUTION_HISTORY` | query | 6–8h | 365d | Credits per query (excl. idle) |
| `QUERY_METERING_HISTORY` | query × hour | 1h | 365d | Per-query credits **for adaptive warehouses** |
| `QUERY_INSIGHTS` | query × insight | — | 365d | **Snowflake's own anti-pattern detection** |
| `QUERY_ACCELERATION_ELIGIBLE` | query | — | 365d | QAS opportunity sizing |
| `ACCESS_HISTORY` | query | ~3h | 365d | Object and column read/write lineage |
| `AGGREGATE_ACCESS_HISTORY` | aggregated | — | 365d | Scalable version of the above |
| `TABLE_STORAGE_METRICS` | table | 90 min | current | Active / time-travel / fail-safe / clone bytes |
| `TABLE_PRUNING_HISTORY` | table × hour | 6h | 365d | Partition pruning efficiency per table |
| `TABLE_QUERY_PRUNING_HISTORY` | table × query | 4h | 365d | Pruning attributed to specific queries |
| `COLUMN_QUERY_PRUNING_HISTORY` | column × query | 4h | 365d | **Which column would make a good clustering key** |
| `TABLE_DML_HISTORY` | table × hour | 6h | 365d | Write churn |
| `AUTOMATIC_CLUSTERING_HISTORY` | table × hour | 3h | 365d | Reclustering credits |
| `MATERIALIZED_VIEW_REFRESH_HISTORY` | MV × hour | — | 365d | MV maintenance credits |
| `SEARCH_OPTIMIZATION_HISTORY` | table × hour | — | 365d | SO maintenance credits |
| `SEARCH_OPTIMIZATION_BENEFITS` | table × hour | — | 365d | **Partitions pruned *because of* SO** |
| `DYNAMIC_TABLE_REFRESH_HISTORY` | DT × refresh | — | 365d | Dynamic table cost |
| `SERVERLESS_TASK_HISTORY` | task × hour | — | 365d | Serverless task credits |
| `PIPE_USAGE_HISTORY` | pipe × hour | — | 365d | Snowpipe credits and bytes |
| `DATA_TRANSFER_HISTORY` | transfer × hour | — | 365d | Egress cost |
| `TABLES`, `COLUMNS`, `OBJECT_DEPENDENCIES`, `TAG_REFERENCES` | object | 90 min – 3h | current | Metadata, lineage, ownership |
| `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY` | account × day × service | 24h | — | **Real money** |
| `ORGANIZATION_USAGE.RATE_SHEET_DAILY` | account × day × type | 24h | — | Contract effective rate |

### 2.4 Two views that change the build/buy calculus

**`QUERY_INSIGHTS`** — Snowflake now ships per-query anti-pattern detection with structured `insight_type_id` values including `QUERY_INSIGHT_NO_FILTER_ON_TOP_OF_TABLE_SCAN`, `QUERY_INSIGHT_UNSELECTIVE_FILTER`, `QUERY_INSIGHT_EXPLODING_JOIN`, `QUERY_INSIGHT_LIKE_WITH_LEADING_WILDCARD`, and spillage insights. Each row carries a `message` VARIANT and a `suggestions` array, plus an `is_opportunity` boolean.

This is generated from the actual query plan, which is strictly better than anything you can infer from parsing SQL text. **Consume it rather than reimplementing it.** Your differentiation moves from "detect the anti-pattern" to "price the anti-pattern and rank it against everything else" — which Snowflake does *not* do.

**`SEARCH_OPTIMIZATION_BENEFITS` and `QUERY_ACCELERATION_ELIGIBLE`** — these give you the benefit side of two feature-ROI equations for free. Pair either with its cost view and you have a defensible ROI number with no modelling at all.

---

## Part 3 — Statistical toolkit

Match the method to the distribution. Snowflake cost data is heavily right-skewed and strongly weekly-seasonal; naive mean/stddev methods fail on both counts.

| Problem | Wrong method | Right method | Why |
|---|---|---|---|
| Rank spend | Mean, stddev | **Percentile rank, Gini/Lorenz** | Spend is Pareto; the mean sits above the 80th percentile |
| Spot a cost spike | `today > 1.5 × avg` | **Robust z on day-of-week strata**: `(x − median) / (1.4826 × MAD)`, flag \|z\| > 3.5 | Median/MAD resist the outliers you are hunting; DoW strata remove weekend seasonality |
| Find *when* cost changed | Threshold alert | **Changepoint detection (PELT / BOCPD)** | Returns a date, which is what makes it actionable |
| Forecast month-end | Linear extrapolation | **Holt-Winters, weekly seasonality** | Weekday/weekend split dominates the series |
| Optimal auto_suspend | Rule of thumb | **Empirical CDF of inter-query gaps, 1-D cost minimisation** | Deterministic and exactly correct; see 5.3 |
| Predict resize impact | Guess an elasticity | **Difference-in-differences on historical resizes**; fall back to spill-based bound | You have real natural experiments in `WAREHOUSE_EVENTS_HISTORY` |
| Consolidation savings | Add up idle | **Interval union vs interval sum** | Exact, no model risk |
| Will a cold table be read again | "0 reads = dead" | **Kaplan-Meier on historical read-gap distribution** | Gives a probability with a confidence bound instead of a false certainty |
| Savings confidence | Point estimate | **Bootstrap over daily observations, report p10–p90** | Enterprises trust a range more than a suspiciously precise number |
| Verify a change worked | Before vs after | **Difference-in-differences vs unchanged control warehouses** | Strips out org-wide trend and seasonality that would otherwise be credited to you |
| Group similar queries | Custom SQL fingerprint | **`QUERY_PARAMETERIZED_HASH`** | Native, plan-aware, free |
| Find duplicate tables | Name matching | **MinHash / Jaccard on column signature + co-access clustering** | Names lie; schemas and access patterns do not |

**Normalization convention for composite scores.** Use percentile rank within the account, not min-max. Min-max lets one 6XL warehouse compress every other warehouse to ~0 and makes scores incomparable across accounts and across time. Percentile rank is stable and lets you say "this is in the worst 5% of your estate."

---

## Part 4 — Family A: Foundation & spend attribution

These are not optimizations. They are the substrate: if the totals do not reconcile to the invoice, nothing downstream is believed.

| # | KPI | Formula | Source | Statistical treatment |
|---|---|---|---|---|
| A1 | **Total billed credits** | `SUM(credits_used)` by `service_type` | `METERING_HISTORY` | None — this is ground truth |
| A2 | **Effective credit price** | `USAGE_IN_CURRENCY / USAGE` where `RATING_TYPE='compute'` | `ORG_USAGE.USAGE_IN_CURRENCY_DAILY`, `RATE_SHEET_DAILY` | Take the trailing 30d weighted mean; rates change mid-contract |
| A3 | **Cloud services billable** | `credits_used_cloud_services − credits_adjustment_cloud_services` | `METERING_DAILY_HISTORY` | Report only the positive remainder |
| A4 | **Spend by warehouse** | `SUM(credits_used)` grouped | `WAREHOUSE_METERING_HISTORY` | Percentile rank + cumulative share |
| A5 | **Spend by user / role** | `SUM(credits_attributed_compute)` grouped | `QUERY_ATTRIBUTION_HISTORY` | Allocation only — see A8 |
| A6 | **Spend by team / cost centre** | Group by `QUERY_TAG` or object tag | `QUERY_ATTRIBUTION_HISTORY` ⋈ `TAG_REFERENCES` | Requires a tagging convention to exist; measure tag coverage % and report it |
| A7 | **Spend concentration (Gini)** | Gini coefficient over per-shape credits | `QUERY_ATTRIBUTION_HISTORY` | Gini > 0.8 means a handful of shapes own the bill — good news, it means few fixes needed |
| A8 | **Attribution coverage** | `SUM(credits_attributed_compute_queries) / SUM(credits_used_compute)` | `WAREHOUSE_METERING_HISTORY` (single view — both columns present) | **Publish this number.** It is your honesty metric |
| A9 | **Fully-loaded query cost** | `attributed + (idle × attributed_share)` | derived from A8 | Distributes idle back onto queries pro-rata — the only fair per-team chargeback |

**A8 deserves emphasis.** `WAREHOUSE_METERING_HISTORY` now carries `CREDITS_ATTRIBUTED_COMPUTE_QUERIES` alongside `CREDITS_USED_COMPUTE`, so idle cost is a single subtraction within one view — you no longer need to join to `QUERY_ATTRIBUTION_HISTORY` to get it. Note the column is NULL for adaptive warehouses.

**A9 is the politically important one.** If you charge a team only for attributed credits, nobody owns the idle time, and idle time is often 30–50% of the bill. Pro-rata redistribution makes idle somebody's problem.

---

## Part 5 — Family B: Warehouse efficiency

The largest single pool of recoverable money in most estates.

| # | KPI | Formula | Source | Signal |
|---|---|---|---|---|
| B1 | **Idle credits** | `credits_used_compute − credits_attributed_compute_queries` | `WAREHOUSE_METERING_HISTORY` | Money spent with nothing running |
| B2 | **Idle share** | `B1 / credits_used_compute` | derived | >30% with meaningful absolute spend → act |
| B3 | **Query load** | `AVG_RUNNING` | `WAREHOUSE_LOAD_HISTORY` | **Not a percentage.** See caveat below |
| B4 | **Queue load** | `AVG_QUEUED_LOAD` | `WAREHOUSE_LOAD_HISTORY` | >0 sustained → undersized or needs more clusters |
| B5 | **Provisioning queue** | `AVG_QUEUED_PROVISIONING` | `WAREHOUSE_LOAD_HISTORY` | High → cold-start thrash, auto_suspend too aggressive |
| B6 | **Blocked load** | `AVG_BLOCKED` | `WAREHOUSE_LOAD_HISTORY` | Lock contention, not a sizing problem — do not resize |
| B7 | **Uptime hours** | Union of `RESUME`→`SUSPEND` intervals | `WAREHOUSE_EVENTS_HISTORY` | Denominator for everything |
| B8 | **Resume count** | `COUNT(RESUME_WAREHOUSE)` | `WAREHOUSE_EVENTS_HISTORY` | × 60s minimum = floor on billing waste |
| B9 | **Cluster-hours** | Union of `SPINUP`→`SPINDOWN` per cluster | `WAREHOUSE_EVENTS_HISTORY` | Real multi-cluster cost |
| B10 | **Max cluster headroom** | `max_cluster_count − p99(clusters_active)` | `SHOW WAREHOUSES` + B9 | Headroom never used |
| B11 | **Spill rate** | `SUM(bytes_spilled_to_remote_storage) / SUM(bytes_scanned)` | `QUERY_HISTORY` | **The single best undersizing signal** |
| B12 | **Local spill rate** | `bytes_spilled_to_local_storage` share | `QUERY_HISTORY` | Mild pressure; remote spill is the severe one |
| B13 | **Optimal auto_suspend** | argmin of the cost function in 5.3 | `QUERY_HISTORY` gaps + `SHOW WAREHOUSES` | Deterministic |
| B14 | **Right-size verdict** | see 5.2 | multiple | Direction + confidence, not a bare number |

### 5.1 B3 is the most misread metric in Snowflake

`AVG_RUNNING` is a **load ratio**, not utilization: total query execution seconds in the interval divided by interval length, over 5-minute buckets. 4 queries totalling 276 seconds in a 300-second interval gives 0.92.

Therefore:

- it can exceed 1.0 (concurrency, or multiple clusters)
- 0.92 does **not** mean "92% utilized" — one query running the whole interval on a 4XL also gives 1.0 while using a fraction of the hardware
- it says nothing about whether the warehouse is the right *size*, only whether it is *busy*

Use B3 for consolidation and concurrency reasoning. Use **B11 (spill)** for sizing. Conflating them produces recommendations that make queries slower and save nothing.

### 5.2 Right-sizing without pretending to have a latency model

The honest hierarchy, best evidence first:

1. **Natural experiment.** Find `RESIZE_WAREHOUSE` events in `WAREHOUSE_EVENTS_HISTORY`. For query shapes (`QUERY_PARAMETERIZED_HASH`) that ran both before and after, compute difference-in-differences on runtime and on warehouse-seconds. This is measured elasticity for *your* workload, and it is the strongest claim you can make.
2. **Cross-warehouse observation.** The same `QUERY_PARAMETERIZED_HASH` often runs on several warehouses of different `WAREHOUSE_SIZE` (visible directly in `QUERY_HISTORY`). Regress log(runtime) on log(size_rank) per shape. Pooling these gives a workload-specific elasticity distribution.
3. **Spill-bounded heuristic.** Zero remote spill, zero queue load, and p95 load ratio well under 1 means a size reduction is unlikely to hurt. Non-zero remote spill means a size reduction will hurt badly — memory halves with each step down and spill-to-remote is catastrophic for runtime.

**Emit direction and confidence, never a naked percentage.** "Down one size, expected −45% credits, latency impact bounded by +15% based on 340 observed executions across two sizes, confidence 0.82" is defensible. "Saves $7,100/month" alone is not.

### 5.3 Optimal auto-suspend — fully deterministic, no ML

This is one of the cleanest wins in the whole product because the answer is exactly computable.

Extract the sorted sequence of query start/end times per warehouse from `QUERY_HISTORY`. Derive the set of inter-query gaps `G`. For a candidate auto_suspend value `T` seconds:

```
billed_seconds(T) = Σ_{g ∈ G} min(g, T)                    # idle seconds paid before suspending
                  + 60 × |{g ∈ G : g > T}|                 # 60s minimum on each subsequent resume
                  + Σ query execution seconds              # constant, independent of T
```

Minimise over `T ∈ {30, 60, 90, ..., 900}`. Snowflake's suspension poller runs roughly every 30 seconds, so values that are not multiples of 30 are not meaningfully different — restrict the search space accordingly.

The shape of this function is why "just set it to 60 seconds" is bad advice: as `T` falls, the first term shrinks but the second grows. For a warehouse with many 2–3 minute gaps, aggressive suspension **increases** the bill. Your engine should be able to output "keep the current setting" and be right.

Two caveats to encode:

- Suspending discards the warehouse's local SSD cache, so the next queries re-read from remote storage. This does not show in the formula but it is real. Weight against aggressive suspension on warehouses with high `PERCENTAGE_SCANNED_FROM_CACHE`.
- Resume latency (~1–2s, longer for large warehouses) matters for interactive SLA-bearing warehouses. Gate the recommendation on workload class (Family F).

---

## Part 6 — Family C: Warehouse consolidation

Your stated priority, and the family with the least existing coverage. It is also the one where the arithmetic is exact rather than modelled, which makes it unusually defensible.

### 6.1 The core insight

Two warehouses each running 4 hours a day cost 8 warehouse-hours. If their busy periods **overlap**, merging them costs the *union* of their busy intervals, not the sum. Savings equal the overlap, and the overlap is directly observable.

| # | KPI | Formula | Source |
|---|---|---|---|
| C1 | **Busy interval set** | 5-min buckets where `AVG_RUNNING > 0` | `WAREHOUSE_LOAD_HISTORY` |
| C2 | **Uptime interval set** | `RESUME`→`SUSPEND` spans | `WAREHOUSE_EVENTS_HISTORY` |
| C3 | **Pairwise temporal overlap** | `\|busy(A) ∩ busy(B)\| / \|busy(A) ∪ busy(B)\|` | derived from C1 |
| C4 | **Consolidation saving** | `(Σ uptime_i − \|∪ uptime_i\|) × credits_per_hour` | C2 + size |
| C5 | **Combined peak concurrency** | `p99(Σ AVG_RUNNING over members)` | `WAREHOUSE_LOAD_HISTORY` |
| C6 | **Concurrency feasibility** | `C5 ≤ target_size_capacity × max_clusters` | C5 + config |
| C7 | **Workload affinity** | Jaccard over sets of tables touched | `ACCESS_HISTORY` |
| C8 | **Role/security affinity** | Overlap of roles executing on each | `QUERY_HISTORY.ROLE_NAME` |
| C9 | **SLA class conflict** | Do members mix interactive and batch? | Family F classification |
| C10 | **Size compatibility** | Spread of member sizes | `QUERY_HISTORY.WAREHOUSE_SIZE` |

### 6.2 The counterintuitive part

**Low temporal overlap makes consolidation *more* attractive, not less.** Two warehouses that are busy at the same time save nothing by merging — you still need the same total compute simultaneously, and you may now introduce queuing. Two warehouses busy at *different* times of day merge almost for free.

So C3 is inverted relative to intuition: you are looking for **low C3 with high C7** — different times, similar data. Similar data matters because merging warehouses that touch the same tables preserves local cache warmth, which is a second-order saving on top of the uptime saving.

### 6.3 Consolidation candidate score

```
consolidation_score =
      w1 · (1 − temporal_overlap)          # different busy windows
    + w2 · workload_affinity               # same tables → shared cache
    + w3 · size_compatibility              # similar sizes merge cleanly
    − w4 · sla_conflict                    # never merge interactive with batch
    − w5 · security_conflict               # disjoint role sets are a red flag
    − w6 · concurrency_infeasibility       # would the merged peak queue?
```

Gate the whole thing on C6. If the combined p99 concurrency exceeds what the target configuration can absorb without queuing, the recommendation is invalid regardless of how good the score looks.

### 6.4 What to be careful about

- **Resource monitors and quotas** are attached per warehouse. Merging changes the blast radius of a spend cap.
- **Query-level isolation** disappears. One runaway batch query can now starve a BI dashboard.
- **Chargeback breaks.** If finance allocates by warehouse name, consolidation destroys their model unless you have query tags in place first. In practice this, not the technical risk, is what kills consolidation proposals — surface it as a prerequisite rather than discovering it at approval time.

---

## Part 7 — Family D: Query efficiency

Remember 1.1: query-level savings are only real if they reduce warehouse uptime or prevent an upsize. Price accordingly.

| # | KPI | Formula | Source |
|---|---|---|---|
| D1 | **Credits per execution** | `credits_attributed_compute` | `QUERY_ATTRIBUTION_HISTORY` |
| D2 | **Scan-to-output ratio** | `bytes_scanned / NULLIF(bytes_written_to_result, 0)` | `QUERY_HISTORY` |
| D3 | **Partition pruning ratio** | `partitions_scanned / NULLIF(partitions_total, 0)` | `QUERY_HISTORY` |
| D4 | **Remote spill ratio** | `bytes_spilled_to_remote_storage / bytes_scanned` | `QUERY_HISTORY` |
| D5 | **Cache hit** | `percentage_scanned_from_cache` | `QUERY_HISTORY` |
| D6 | **Queue fraction** | `(queued_overload + queued_provisioning) / total_elapsed_time` | `QUERY_HISTORY` |
| D7 | **Compilation fraction** | `compilation_time / total_elapsed_time` | `QUERY_HISTORY` |
| D8 | **Row explosion** | `rows_produced / NULLIF(rows_inserted + rows_updated, 0)` | `QUERY_HISTORY` |
| D9 | **Failed-query spend** | credits where `execution_status = 'FAIL'` | join `QUERY_HISTORY` ⋈ attribution |
| D10 | **Retry waste** | `query_retry_time` share | `QUERY_HISTORY` |
| D11 | **Native insight flags** | `insight_type_id`, `suggestions` | **`QUERY_INSIGHTS`** |
| D12 | **Table pruning efficiency** | `partitions_pruned / (partitions_pruned + partitions_scanned)` | `TABLE_PRUNING_HISTORY` |
| D13 | **Clustering key candidate** | column with highest pruning contribution | **`COLUMN_QUERY_PRUNING_HISTORY`** |

### 7.1 D2 needs care

The intuitive form is `bytes_scanned / rows_produced`, but rows are not comparable across schemas — a 4-column row and a 400-column row are not the same unit. Use `BYTES_WRITTEN_TO_RESULT` as the denominator instead, and treat `ROWS_PRODUCED` as a secondary explanatory field. Also exclude `QUERY_TYPE` values where a large scan with small output is the entire point: aggregations, `CREATE TABLE AS`, and `MERGE` will otherwise dominate your worst-offenders list with correct behaviour.

### 7.2 D9 and D10 are underrated

Failed queries still consume the warehouse-seconds they ran for. A retry loop on a broken pipeline can burn real credits indefinitely while producing nothing, and no one is watching because there is no output to notice. This is easy to compute, easy to explain, and almost always finds something.

### 7.3 D13 is a genuinely differentiated feature

`COLUMN_QUERY_PRUNING_HISTORY` tells you, per column, how much pruning that column achieved across real queries. That converts "pick a clustering key" from an expert judgement call into a ranked list backed by your actual workload. Pair with `AUTOMATIC_CLUSTERING_HISTORY` for the cost side and you have full-loop clustering ROI. Very few tools do this.

### 7.4 Composite query cost score

```
query_cost_score = 100 × Σ wᵢ · percentile_rank(metricᵢ)
```

over D2, D3 (inverted), D4, D6, and execution frequency. **Percentile rank, not min-max** — one pathological query would otherwise flatten the entire distribution.

Critically, multiply the score by an **addressability factor**: a terrible query that runs twice a month on a warehouse that is busy anyway is worth nothing. A mediocre query that runs 40,000 times and is the sole reason a warehouse never suspends is worth a lot. Score × frequency × marginal-uptime-contribution is the number that should drive ranking.

---

## Part 8 — Family E: Recurring & repeated workload

Usually the largest single addressable pool, and the most commonly missed.

| # | KPI | Formula | Source |
|---|---|---|---|
| E1 | **Executions per shape** | `COUNT(*)` by `query_parameterized_hash` | `AGGREGATE_QUERY_HISTORY` |
| E2 | **Credits per shape** | `SUM(credits_attributed_compute)` by hash | `QUERY_ATTRIBUTION_HISTORY` |
| E3 | **Per-execution cost** | `E2 / E1` | derived |
| E4 | **Waste archetype** | see below | derived |
| E5 | **Cadence regularity** | stddev of inter-execution interval | `QUERY_HISTORY` |
| E6 | **Result-identical repeats** | repeats over unchanged base tables | `QUERY_HISTORY` ⋈ `TABLE_DML_HISTORY` |
| E7 | **Uptime attributable to shape** | uptime that exists *only* because of this shape | E1 + `WAREHOUSE_EVENTS_HISTORY` |

Split E4 into archetypes, because each has a different fix:

| Archetype | Condition | Fix |
|---|---|---|
| **Heavy** | high E3, low E1 | Tune the query |
| **Relentless** | low E3, very high E1 | Reduce cadence, or cache |
| **Both** | high E3, high E1 | Escalate immediately |
| **Polling** | low E5 (very regular) + E6 high | The data has not changed — this is pure waste |
| **Uptime anchor** | high E7 | This shape alone is preventing auto-suspend |

**E6 and E7 are the differentiated ones.** E6 catches the dashboard refreshing every 60 seconds against a table that updates daily — you can prove the result was identical by comparing execution time against `TABLE_DML_HISTORY` on the base tables. E7 catches the low-cost heartbeat query that runs every 4 minutes and, entirely on its own, keeps a 2X-Large from ever suspending. E7 is frequently the single most valuable finding in an estate and it is invisible to every cost-per-query view, because the query itself is nearly free.

Filter thresholds: `E1 ≥ 10` and `E2 ≥ 1.0` credit. Below that you are surfacing human ad-hoc noise.

---

## Part 9 — Family F: Workload classification

Not a savings source directly. It is the **risk gate** that makes every other recommendation safe.

| # | KPI | Formula | Source |
|---|---|---|---|
| F1 | **Workload class** | classify ETL / BI / ad-hoc / ML / batch | `QUERY_HISTORY` |
| F2 | **Class purity** | share of the dominant class per warehouse | derived |
| F3 | **Interactivity index** | fraction of queries with a human-like arrival pattern | `SESSIONS`, `QUERY_HISTORY` |
| F4 | **Diurnal profile** | credits by hour-of-day × day-of-week | `WAREHOUSE_METERING_HISTORY` |
| F5 | **Burstiness** | `p99(load) / median(load)` | `WAREHOUSE_LOAD_HISTORY` |

Classify from observable features rather than names: `QUERY_TYPE` distribution, arrival regularity (cron-like vs Poisson), client application from `SESSIONS`, read/write ratio, `ROLE_NAME` patterns, and whether the session is `IS_CLIENT_GENERATED_STATEMENT`. A k-means or simple rule cascade over these features is sufficient — this does not need to be sophisticated.

**Why it gates everything:** a 40% latency increase on a nightly ETL job that finishes at 3am is free. The same increase on a customer-facing dashboard is an incident. F1 is what lets your risk score distinguish them, and without it you cannot responsibly recommend any downsize.

F2 also generates its own recommendation: a warehouse that is 70% BI and 30% ETL has an isolation problem, because one heavy ETL query will block interactive users. That is the *opposite* of consolidation and the two families must be reconciled before either emits an opportunity.

---

## Part 10 — Family G: Anomaly & regression

| # | KPI | Formula | Source | Method |
|---|---|---|---|---|
| G1 | **Daily cost anomaly** | robust z on DoW strata | `WAREHOUSE_METERING_HISTORY` | MAD-based, \|z\|>3.5 |
| G2 | **Shape-level regression** | changepoint in daily credits per hash | `QUERY_ATTRIBUTION_HISTORY` | PELT |
| G3 | **Pruning regression** | drop in `partitions_pruned` ratio | `TABLE_PRUNING_HISTORY` | Changepoint |
| G4 | **Scan volume regression** | rise in `bytes_scanned` per shape | `AGGREGATE_QUERY_HISTORY` | Changepoint |
| G5 | **Frequency regression** | rise in executions per shape | `AGGREGATE_QUERY_HISTORY` | Changepoint |
| G6 | **Config change correlation** | resize/suspend events near a changepoint | `WAREHOUSE_EVENTS_HISTORY` | Temporal join |
| G7 | **Data volume growth** | table byte growth rate | `TABLE_STORAGE_METRICS` over time | Trend |
| G8 | **New-object cost** | credits from objects first seen in window | `ACCESS_HISTORY` + attribution | — |
| G9 | **Forecast variance** | projected vs budget, decomposed by driver | `METERING_DAILY_HISTORY` | Holt-Winters |

### 10.1 Attribute the cause, do not just detect the spike

Detection is easy and low value. The product value is in the decomposition. When G1 fires, walk the hierarchy automatically:

```
account spike
  → which warehouse contributed most of the delta      (metering)
    → which query shape contributed most of that delta (attribution by hash)
      → which of these changed:                        (decompose)
           frequency ↑        (G5)  → someone scheduled it more often
           bytes scanned ↑    (G4)  → the data grew, or pruning broke
           pruning ↓          (G3)  → predicate or clustering changed
           warehouse size ↑   (G6)  → someone resized
           new object         (G8)  → a new pipeline launched
```

Each leaf is a different owner and a different fix. **G4 must be normalized against G7** — if a table doubled in size, a doubling in bytes scanned is expected behaviour, not a regression. Failing to separate organic data growth from genuine efficiency loss is the most common false-positive source in this family.

### 10.2 Normalize regressions correctly

Do not detect regressions on raw credits. Credits are confounded by warehouse size, which someone may have changed for unrelated reasons. Detect on **warehouse-seconds normalized by warehouse size** (that is, size-adjusted compute-seconds), or on `bytes_scanned per row returned`. Then translate to credits only for reporting.

---

## Part 11 — Family H: Storage

Lower priority per Part 1.5, but the existing codebase already covers most of it well.

| # | KPI | Formula | Source |
|---|---|---|---|
| H1 | **True reclaimable bytes** | `max(0, active + time_travel + failsafe − retained_for_clone)` | `TABLE_STORAGE_METRICS` |
| H2 | **Time-travel overhead** | `time_travel_bytes / active_bytes` | `TABLE_STORAGE_METRICS` |
| H3 | **Fail-safe overhead** | `failsafe_bytes / active_bytes` | `TABLE_STORAGE_METRICS` |
| H4 | **Retention appropriateness** | `retention_time` vs table's actual churn | `TABLES` + `TABLE_DML_HISTORY` |
| H5 | **Coldness** | days since last read | `ACCESS_HISTORY` |
| H6 | **Write-hot / read-cold** | writes > 0, reads = 0 | `ACCESS_HISTORY` |
| H7 | **Orphan degree** | count of dependents | `OBJECT_DEPENDENCIES` |
| H8 | **Schema-similarity family** | MinHash over column signature | `COLUMNS` |
| H9 | **Access-pattern similarity** | Jaccard over querying-role sets | `ACCESS_HISTORY` |
| H10 | **Churn-amplified storage** | high DML on a high-retention table | `TABLE_DML_HISTORY` + `TABLES` |
| H11 | **Stage bytes** | staged files not cleaned up | `STAGE_STORAGE_USAGE_HISTORY` |
| H12 | **Unused column ratio** | never-referenced columns / total | `ACCESS_HISTORY` column flatten |
| H13 | **Read-probability** | Kaplan-Meier survival estimate | `ACCESS_HISTORY` history |

### 11.1 Three storage corrections worth encoding

**H3 — fail-safe is not reclaimable on demand.** Permanent tables carry a fixed 7-day fail-safe period that cannot be configured or shortened. Dropping a table does *not* immediately free those bytes. Transient and temporary tables have no fail-safe at all. So "drop this to save X TB" must be phrased as "saves X TB after 7 days," and the transient/permanent distinction changes the number materially.

**H4 and H10 are the underused lever.** Time-travel storage cost scales with *churn*, not table size. A 100 GB table rewritten fully every day with 90-day retention costs vastly more than a 5 TB table that never changes. Ranking tables by `retention_time × daily_churn_bytes` finds waste that a pure size ranking completely misses — and reducing a retention period is a far easier approval than deleting data.

**H13 replaces a false claim with an honest one.** Rather than "0 reads in 365 days, therefore dead," fit a survival curve on observed read-gap distributions across the estate and report "based on tables with similar access patterns, probability of a read in the next 90 days is 3%." That is defensible in a way that a bare zero is not, and it survives the objection that `ACCESS_HISTORY` retention is finite.

---

## Part 12 — Family I: Feature ROI

Optimization features cost money. A tool that can say "turn this off" is more credible than one that only ever recommends adding things.

| Feature | Cost source | Benefit source | ROI verdict |
|---|---|---|---|
| **Automatic clustering** | `AUTOMATIC_CLUSTERING_HISTORY.credits_used` | `TABLE_PRUNING_HISTORY` pruning improvement | Credits spent reclustering vs credits saved scanning |
| **Search optimization** | `SEARCH_OPTIMIZATION_HISTORY.credits_used` | **`SEARCH_OPTIMIZATION_BENEFITS.partitions_pruned_additional`** | Direct, measured — no modelling needed |
| **Query acceleration** | `QUERY_ACCELERATION_HISTORY.credits_used` | **`QUERY_ACCELERATION_ELIGIBLE.eligible_query_acceleration_time`** | Eligible seconds vs credits consumed |
| **Materialized views** | `MATERIALIZED_VIEW_REFRESH_HISTORY.credits_used` | Query credits avoided by MV rewrite | Often negative on high-churn base tables |
| **Dynamic tables** | `DYNAMIC_TABLE_REFRESH_HISTORY` | Downstream compute avoided | Compare against the ETL it replaced |
| **Snowpipe** | `PIPE_USAGE_HISTORY.credits_used / bytes_inserted` | — | Credits per TB ingested vs `COPY` on a warehouse |

Two composite KPIs:

- **I1 — Feature ROI ratio:** `(credits_saved − credits_spent) / credits_spent`. Negative means turn it off. A clustered table that no longer receives selective queries reclusters forever for nothing, and this is a common, easy, uncontroversial win.
- **I2 — Dead-but-maintained objects:** the intersection of H5 (cold) with any nonzero maintenance credits. A table nobody reads that still reclusters daily is a pure, recurring, zero-risk saving. This is the cleanest opportunity type in the entire catalog and should be in your demo.

---

## Part 13 — Family J: Opportunity & verification meta-KPIs

These make it a product rather than a report.

| # | KPI | Formula | Notes |
|---|---|---|---|
| J1 | **Estimated saving** | family-specific | Always a p10–p90 range via bootstrap |
| J2 | **Confidence** | `f(observation_days, variance, n_executions, evidence_class)` | Natural experiment > observational > modelled |
| J3 | **Risk** | `g(workload_class, blast_radius, reversibility)` | Gated on F1 |
| J4 | **Effort** | config change / query rewrite / architecture change | Determines who can approve |
| J5 | **Addressability** | is anyone able and willing to own this? | The gap between theoretical and realistic |
| J6 | **Priority** | `J1 × J2 / (J3 × J4)` | The backlog sort key |
| J7 | **Realization rate** | `verified / estimated`, tracked over time | **Calibrates J1** |
| J8 | **Verified saving** | difference-in-differences vs control | The only number labelled "verified" |
| J9 | **Coverage** | share of the bill your KPIs can explain | Same spirit as A8 |

### 13.1 J8 is the hard one and the one that matters

Naive before/after is wrong. If you resize a warehouse on the 1st and the business has a quiet month, you will claim savings you did not create — and eventually someone will check.

Use difference-in-differences:

```
verified = (control_after − control_before) − (treated_after − treated_before)
```

where the control group is a set of warehouses with similar workload class and diurnal profile (F1, F4) that were **not** changed in the window. This strips out org-wide seasonality and trend. Report a bootstrapped confidence interval, and report it as zero when the interval spans zero.

### 13.2 J7 is what earns trust over time

Track the ratio of verified to estimated savings per opportunity type. After a few dozen verified changes you can tell a customer "our auto-suspend estimates historically realize at 0.91; our right-sizing estimates realize at 0.68" — and then apply those factors to future estimates. That feedback loop is the product's real moat, and it is not something a dashboard can copy.

---

## Part 14 — Traps

Each of these has burned a real cost tool.

| Trap | Reality |
|---|---|
| `AVG_RUNNING` is utilization % | It is a load ratio and can exceed 1.0 |
| Bytes scanned measures cost | Cost is warehouse-seconds × size × clusters. A 4 TB scan on an XS is cheap |
| Σ attributed credits = the bill | Excludes idle, sub-100ms queries, cloud services, serverless; NULL on adaptive warehouses |
| Lower auto_suspend is always better | 60s resume minimums and cache loss can make it more expensive |
| 0 reads = safe to delete | Latency, 365-day retention limit, and quarterly/annual jobs all break this |
| High cache hit % is a win to chase | It is a symptom, not a lever; you cannot directly act on it |
| Cloud services spend is waste | Free below 10% of daily compute |
| Fail-safe frees on DROP | 7-day fixed delay for permanent tables; transient tables have none |
| Bigger warehouse = proportionally faster | Only for parallelizable and spilling work; small queries see nothing |
| Consolidation always saves | Only when busy windows do not overlap; otherwise it adds queuing |
| Query cost regression detected on credits | Confounded by warehouse resizes; normalize to size-adjusted compute-seconds |
| Scan growth = regression | Could be organic data growth; always normalize against H7 |
| Storage cleanup is where the money is | Typically 5–10% of the bill |
| Query tags will be there | Tag coverage is usually poor; measure and report it before relying on chargeback |
| One `SHOW WAREHOUSES` snapshot is enough | Config has no history in ACCOUNT_USAGE — snapshot every run or you cannot explain past cost |

---

## Part 15 — Suggested MVP subset

If everything above is the menu, this is the plate. Roughly ordered by expected recovered dollars per unit of engineering effort.

| Priority | KPIs | Why first |
|---|---|---|
| 1 | A1, A2, A8, B1, B2 | The billing anchor plus idle cost. Small, exact, and unarguable |
| 2 | E1–E4, E7 | Repeated-query spend and uptime anchors. Usually the biggest single pool |
| 3 | B13 (auto-suspend) | Deterministic, low risk, easy approval, immediate money |
| 4 | I2 (dead-but-maintained) | Trivially computed, zero risk, always finds something |
| 5 | C1–C6 (consolidation) | Your stated differentiator; exact arithmetic, no model risk |
| 6 | D11 (`QUERY_INSIGHTS`) + D1 | Free anti-pattern detection, priced and ranked by you |
| 7 | G1, G2 (anomaly, regression) | High perceived value, moderate effort |
| 8 | B11, B14 (right-sizing) | Highest savings but hardest to defend — needs J2 mature first |
| 9 | H-family | Already largely built; keep, do not lead with it |
| 10 | J7, J8 (verification) | Ships last but is what makes it a product |

Note that priorities 1–5 involve **no machine learning and no LLM**, and every number in them is either directly measured or exactly computed. That is deliberate. Build the deterministic core first; the model-based layers (right-sizing elasticity, latency prediction, survival analysis) should sit on top of a foundation that is already correct and already trusted.

---

## Part 16 — Open questions to resolve before building

1. **Organization access.** Without `ORGANIZATION_USAGE`, you have credits but not dollars, and no cross-account view. Is per-account deployment with a manual rate card an acceptable fallback for v1?
2. **`GOVERNANCE_VIEWER` / Enterprise Edition.** `ACCESS_HISTORY` is Enterprise-only and often the last role approved. Families H and C7 degrade heavily without it. Which KPIs must still work when it is refused?
3. **Query tag coverage.** A6, A9, and all chargeback depend on tags existing. If coverage is under ~30%, team attribution is not a v1 feature.
4. **Adaptive warehouses.** `CREDITS_ATTRIBUTED_COMPUTE_QUERIES` is NULL and attribution is unavailable for them; `QUERY_METERING_HISTORY` is the substitute. Does the target estate use them?
5. **`SHOW WAREHOUSES` execution.** It needs a live session and appropriate privileges. Confirm the service role can run it, since without config data the warehouse optimizer cannot function.
6. **Positioning against `QUERY_INSIGHTS`.** Snowflake now ships plan-based anti-pattern detection natively. The defensible position is pricing, ranking, and verification — not detection. This should shape the pitch.
