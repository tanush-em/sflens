# 04 — Opportunity Catalog

**Purpose:** Closed set of opportunity types. Users see these; KPIs are machinery underneath.

**Convention for each type:** detection, counterfactual, savings, gates, evidence, confidence class, risk, effort, owner, verification, failure modes, phase.

**Confidence classes:** `exact` > `natural_experiment` > `observational` > `modelled`.

**Effort:** `config` | `query_change` | `architecture` | `governance`.

---

## A. Compute / warehouse

### `IDLE_AUTOSUSPEND`

| Field | Spec |
|-------|------|
| **Detection** | B1 ≥ floor (default 100 credits / lookback) AND B2 ≥ 0.30; config snapshot has `auto_suspend`; B13 `T*` ≠ current (material Δ credits). |
| **Counterfactual** | `ChangeAutoSuspend(T*)` on interval/gap model. |
| **Savings** | Δ billed credits from simulator; bootstrap p10–p90 over days. |
| **Gates** | F1/F3: interactive SLA may block aggressive T; high cache hit → risk↑ or T floor; config known. |
| **Evidence** | Gap CDF, B8 resume count, B5 provisioning queue, cache %, current vs T*. |
| **Confidence** | `exact` |
| **Risk** | Low–Med (resume latency, cache) |
| **Effort** | `config` |
| **Owner** | Platform / warehouse owner |
| **Verification** | Snapshot shows new auto_suspend; Diff-in-diff on idle credits / uptime. |
| **Failure modes** | Ignoring 60s resume minimum; recommending T that increases bill; treating cache as free. |
| **Phase** | 1 (non-negotiable) |

---

### `ZOMBIE_WAREHOUSE`

| Field | Spec |
|-------|------|
| **Detection** | Sustained credits with near-zero busy (C1 empty / B3≈0) over lookback; or zero queries with metering > 0. |
| **Counterfactual** | Eliminate uptime (suspend / remove schedule / stop resume triggers). |
| **Savings** | ≈ all compute credits on warehouse in window (range for residual). |
| **Gates** | Confirm not DR / failover / intentional warm pool (tag or name patterns); F1. |
| **Evidence** | Metering, events, load empty, last query time. |
| **Confidence** | `observational` → `exact` if truly no queries |
| **Risk** | Med (hidden dependency) |
| **Effort** | `config` / `architecture` |
| **Owner** | Platform |
| **Verification** | Warehouse suspended/dropped or credits→0; DiD. |
| **Failure modes** | Warm pools for SLA; seasonal warehouses. |
| **Phase** | 1 |

---

### `UPTIME_ANCHOR_SHAPE`

| Field | Spec |
|-------|------|
| **Detection** | E7 high: removing shape S allows material suspend under current or T*; often low E3, high E1. |
| **Counterfactual** | `RemoveShape(S)` or move S to serverless / different WH / reduce cadence. |
| **Savings** | Simulator Δ credits (often >> attributed credits of S). |
| **Gates** | Business need for heartbeat; F1 if moving to shared WH. |
| **Evidence** | E1–E3, E7, gap plot with/without S. |
| **Confidence** | `exact` (simulator) |
| **Risk** | Low–Med |
| **Effort** | `query_change` / `config` |
| **Owner** | App / dashboard owner |
| **Verification** | Shape cadence change or gone; uptime/idle DiD. |
| **Failure modes** | Cost-per-query tools miss this entirely; don’t use D1 alone. |
| **Phase** | 1 |

---

### `CONSOLIDATE_WAREHOUSES`

| Field | Spec |
|-------|------|
| **Detection** | Pair/group score: high (1−C3), C7, C10; C6 true; C8/C9 pass; material C4. |
| **Counterfactual** | `MergeWarehouses(members → target)` — bill |⋃ uptime| × target rate. |
| **Savings** | C4 with bootstrap; **must claim** union-saved seconds in uptime budget. |
| **Gates** | **C6 hard**; **C9 hard**; C8; resource monitor / chargeback prerequisite (`TAG_COVERAGE_GAP` may block); not under active `ISOLATE_WORKLOAD` on same members. |
| **Evidence** | Overlap heatmap, affinity Jaccard, peak concurrency vs capacity, size spread. |
| **Confidence** | `exact` on uptime arithmetic; `observational` on cache benefit |
| **Risk** | Med–High (blast radius, isolation loss) |
| **Effort** | `architecture` |
| **Owner** | Platform + finance (chargeback) |
| **Verification** | Members retired / traffic on target; DiD on combined credits vs predicted union. |
| **Failure modes** | Merging overlapping busy windows; finance model break; runaway batch starving BI. |
| **Phase** | 1 (non-negotiable) |

---

### `ISOLATE_WORKLOAD`

| Field | Spec |
|-------|------|
| **Detection** | F2 low (e.g. dominant class < 70% with material minority ETL on BI WH). |
| **Counterfactual** | Split classes onto separate warehouses (opposite of merge). |
| **Savings** | Often **performance / risk**, not credit reduction; credit Δ may be ~0 or negative. Rank by risk reduction + optional queue/spill improvement valued qualitatively unless modelled. |
| **Gates** | Conflicts with open `CONSOLIDATE_WAREHOUSES` on same WH — **arbitration prefers isolation first** when F2 breach severe. |
| **Evidence** | Class mix, queue during mixed periods, SLA incidents. |
| **Confidence** | `observational` |
| **Risk** | Med (more warehouses → more idle risk) |
| **Effort** | `architecture` |
| **Owner** | Platform |
| **Verification** | Class purity↑; queue/blocked↓; optional cost watch. |
| **Failure modes** | Treating as savings card; double-counting with consolidation. |
| **Phase** | 1–2 |

---

### `RIGHTSIZE_DOWN`

| Field | Spec |
|-------|------|
| **Detection** | B14 direction down: remote spill≈0, queue≈0, p95 load headroom; optional natural experiment / cross-WH elasticity. |
| **Counterfactual** | `ChangeSize(size−1)` (or −N with stronger evidence). |
| **Savings** | Interval credits × (1 − rate_new/rate_old); latency impact as bound, not fake precision. |
| **Gates** | **B11 remote spill → fail**; B4 queue; F1 interactive → higher bar; B6 blocked ≠ resize. |
| **Evidence** | Spill, queue, load, historical RESIZE DiD if any, same-hash cross-size runs. |
| **Confidence** | `natural_experiment` > `observational` > spill heuristic |
| **Risk** | Med–High |
| **Effort** | `config` |
| **Owner** | Platform |
| **Verification** | Size snapshot/events; DiD credits + latency proxies. |
| **Failure modes** | Naked “saves $X”; AVG_RUNNING as utilization. |
| **Phase** | 2 (after J2 mature) |

---

### `RIGHTSIZE_UP`

| Field | Spec |
|-------|------|
| **Detection** | Sustained remote spill (B11) and/or overload queue (B4) with SLA pain. |
| **Counterfactual** | `ChangeSize(size+1)` — may **increase** credits but reduce runtime enough to cut uptime or unblock; price both. |
| **Savings** | Net credits after simulator + optional shorter busy windows; can be negative (cost for latency). Label as **cost/performance tradeoff**, not pure savings. |
| **Gates** | F1, budget |
| **Evidence** | Spill, queue, runtime distribution |
| **Confidence** | `observational` / `modelled` |
| **Risk** | Med |
| **Effort** | `config` |
| **Owner** | Platform / app |
| **Verification** | Spill↓, runtime↓, credit DiD. |
| **Failure modes** | Selling as “savings” when bill rises. |
| **Phase** | 2 |

---

### `MULTICLUSTER_MAX_REDUCTION`

| Field | Spec |
|-------|------|
| **Detection** | B10: max_cluster_count − p99(clusters_active) large over lookback. |
| **Counterfactual** | `ChangeMaxClusters(new_max ≥ p99 + buffer)`. |
| **Savings** | Rarely direct if max unused; savings = avoided worst-case spin + clarity; credit Δ from periods where max forced extra clusters only if observed. Often **risk/config hygiene** with small direct $. Still emit if headroom absurd. |
| **Gates** | Burstiness F5; SLA; don’t cut below observed p99+margin. |
| **Evidence** | Cluster timeline, B9 |
| **Confidence** | `observational` |
| **Risk** | Med (burst days) |
| **Effort** | `config` |
| **Owner** | Platform |
| **Verification** | Config max↓; no new sustained queue. |
| **Failure modes** | Cutting max that exists for rare Black Friday. |
| **Phase** | 2 |

---

### `RESUME_THRASH`

| Field | Spec |
|-------|------|
| **Detection** | High B8; elevated B5; often auto_suspend too low **or** polling cadence. |
| **Counterfactual** | Raise T toward B13 optimum **or** fix cadence (`UPTIME_ANCHOR` / `RELENTLESS`). |
| **Savings** | Via simulator (resume minimums dominate). |
| **Gates** | Same as auto-suspend / shape |
| **Evidence** | Resume timeline, gap histogram |
| **Confidence** | `exact` |
| **Risk** | Low–Med |
| **Effort** | `config` / `query_change` |
| **Owner** | Platform / app |
| **Verification** | Resume count↓; credits DiD. |
| **Failure modes** | Blindly lowering auto_suspend further. |
| **Phase** | 1 |

---

## B. Query / workload

### `RELENTLESS_SHAPE`

| Field | Spec |
|-------|------|
| **Detection** | E4 Relentless: low E3, very high E1; E2 ≥ floor; E1 ≥ 10. |
| **Counterfactual** | Reduce cadence, cache results, MV/dynamic table, or consolidate polls. Prefer `ThinCadence` for pricing when uptime-linked. |
| **Savings** | Attributed credits × reducible fraction **and/or** simulator if E7-linked. Use addressability. |
| **Gates** | Don’t flag human ad-hoc (E1 floor). |
| **Evidence** | E1–E5, sample SQL features (sanitized) |
| **Confidence** | `observational` |
| **Risk** | Low–Med |
| **Effort** | `query_change` |
| **Owner** | Dashboard / app team |
| **Verification** | Executions↓; credits DiD on shape. |
| **Failure modes** | Pricing bytes scanned as cost. |
| **Phase** | 1 |

---

### `HEAVY_SHAPE`

| Field | Spec |
|-------|------|
| **Detection** | E4 Heavy or Both; D11 insights optional; high D2/D4 etc. |
| **Counterfactual** | Tune SQL / clustering / warehouse; price via attributed Δ **only if** uptime or size pressure reduces — else label efficiency with weak $. |
| **Savings** | Prefer marginal uptime contribution × frequency; consume D11 for “what to fix.” |
| **Gates** | Addressability J5; exclude QUERY_TYPEs where huge scan is intentional (see catalog D2). |
| **Evidence** | D1–D11, plan insights |
| **Confidence** | `observational` |
| **Risk** | Med |
| **Effort** | `query_change` |
| **Owner** | Query author team |
| **Verification** | Shape credits/runtime↓. |
| **Failure modes** | Reimplementing QUERY_INSIGHTS. |
| **Phase** | 1–2 |

---

### `POLLING_ON_STATIC_DATA`

| Field | Spec |
|-------|------|
| **Detection** | E5 regular + E6 high (base tables unchanged between runs via `TABLE_DML_HISTORY`). |
| **Counterfactual** | Align poll to data cadence; cache. |
| **Savings** | Removable executions’ attributed + uptime if anchor. |
| **Gates** | Near-real-time product requirements. |
| **Evidence** | DML vs query timeline |
| **Confidence** | `observational` / near-`exact` |
| **Risk** | Low |
| **Effort** | `query_change` |
| **Owner** | App |
| **Verification** | Cadence change; spend↓. |
| **Failure modes** | Missing writes outside ACCESS/DML views latency. |
| **Phase** | 2 (needs solid DML join) |

---

### `FAILED_RETRY_WASTE`

| Field | Spec |
|-------|------|
| **Detection** | D9/D10 material credits on FAIL / retry loops. |
| **Counterfactual** | Fix root cause; stop retries; circuit break. |
| **Savings** | Credits on failed/retry executions (and uptime if exclusive). |
| **Gates** | None special |
| **Evidence** | Error codes, frequency, users |
| **Confidence** | `exact` on attributed fail credits |
| **Risk** | Low |
| **Effort** | `query_change` |
| **Owner** | Pipeline owner |
| **Verification** | Fail rate↓; credits↓. |
| **Failure modes** | Ignoring warehouse-seconds of failures. |
| **Phase** | 1 |

---

### `CLUSTERING_KEY_CANDIDATE`

| Field | Spec |
|-------|------|
| **Detection** | D13 top columns from `COLUMN_QUERY_PRUNING_HISTORY`; pair with AC cost. |
| **Counterfactual** | Set/change clustering key; ROI vs `AUTOMATIC_CLUSTERING_HISTORY`. |
| **Savings** | Scan/credits avoided − maintenance credits (modelled/observational). |
| **Gates** | Write churn; existing AC ROI |
| **Evidence** | Pruning histories, I1 |
| **Confidence** | `observational` |
| **Risk** | Med |
| **Effort** | `architecture` |
| **Owner** | Data engineering |
| **Verification** | Pruning↑; AC credits vs scan. |
| **Failure modes** | Clustering forever with no selective queries (see negative ROI). |
| **Phase** | 2–3 |

---

## C. Feature ROI

### `FEATURE_NEGATIVE_ROI`

| Field | Spec |
|-------|------|
| **Detection** | I1 < 0 for clustering / search optimization / MV / QAS as applicable. |
| **Counterfactual** | Disable feature. |
| **Savings** | Maintenance credits (and storage where relevant). |
| **Gates** | Confirm benefit views truly low |
| **Evidence** | Cost vs benefit views (SO benefits, etc.) |
| **Confidence** | Near-`exact` when benefit view exists |
| **Risk** | Low–Med |
| **Effort** | `config` |
| **Owner** | Platform / table owner |
| **Verification** | Feature off; maintenance→0; watch query regressions. |
| **Failure modes** | Turning off still-useful SO. |
| **Phase** | 2 |

---

### `DEAD_BUT_MAINTAINED`

| Field | Spec |
|-------|------|
| **Detection** | I2: cold (H5) ∩ nonzero maintenance credits. |
| **Counterfactual** | Disable maintenance and/or archive object (storage path). |
| **Savings** | Maintenance credits (+ storage via H later). |
| **Gates** | Governance; write-hot guard |
| **Evidence** | Access + AC/MV/SO histories |
| **Confidence** | High observational |
| **Risk** | Low |
| **Effort** | `config` |
| **Owner** | Table owner |
| **Verification** | Maintenance→0. |
| **Failure modes** | False cold from ACCESS_HISTORY retention. |
| **Phase** | 1 (demo-friendly); needs GOVERNANCE for strong H5 |

*Degrades without ACCESS_HISTORY: use proxy last DDL / query touch from QUERY_HISTORY if available.*

---

### `FEATURE_ADD_POSITIVE_ROI`

| Field | Spec |
|-------|------|
| **Detection** | Eligible benefit (e.g. QAS eligible time, SO benefit potential) >> projected cost. |
| **Counterfactual** | Enable feature. |
| **Savings** | Net (often latency; credit net may be small). |
| **Gates** | Budget; churn |
| **Evidence** | Eligibility / benefits views |
| **Confidence** | `observational` |
| **Risk** | Med |
| **Effort** | `config` |
| **Owner** | Platform |
| **Verification** | Benefit metrics↑; net ROI. |
| **Failure modes** | Only recommending “add,” never “remove.” |
| **Phase** | 3 |

---

## D. Storage (delegated)

Import from Tech Debt Radar; SnowLens wraps as opportunities for one backlog.

### `REDUNDANT_TABLE_FAMILY`

| Field | Spec |
|-------|------|
| **Detection** | Family clustering + redundancy_score ≥ threshold; verdict `retire_candidate` / `review`. |
| **Counterfactual** | Retire redundant members after owner review. |
| **Savings** | H1 true reclaimable × storage rate (A2-like storage price). |
| **Gates** | Write-recency guard; link_status; never auto-delete. |
| **Evidence** | Fingerprint, coldness, orphans, writers |
| **Confidence** | Per existing engine |
| **Risk** | Med–High |
| **Effort** | `governance` |
| **Owner** | Product owner |
| **Verification** | Objects gone; storage metrics↓ after TT/failsafe delay. |
| **Failure modes** | Claiming immediate fail-safe free. |
| **Phase** | 1 (import) |

### `RETENTION_OVERSIZED`

| Field | Spec |
|-------|------|
| **Detection** | H4/H10: retention_time × churn high. |
| **Counterfactual** | Reduce time-travel retention. |
| **Savings** | Time-travel byte reduction × rate. |
| **Gates** | Compliance minimums |
| **Evidence** | DML churn, TABLES.retention |
| **Risk** | Med |
| **Effort** | `config` |
| **Phase** | 2 |

### `COLD_TABLE_ARCHIVE`

| Field | Spec |
|-------|------|
| **Detection** | H5/H13 cold with reclaimable bytes; Kaplan-Meier style probability preferred over “0 reads = dead.” |
| **Counterfactual** | Archive / drop after approval. |
| **Savings** | H1 × rate after failsafe delay messaging. |
| **Gates** | Dependencies H7; regulatory |
| **Risk** | High |
| **Effort** | `governance` |
| **Phase** | 2 |

---

## E. Incidents (not savings portfolio)

### `COST_ANOMALY`

| Field | Spec |
|-------|------|
| **Detection** | G1 robust z on DoW strata \|z\| > 3.5. |
| **Counterfactual** | N/A — investigate decomposition. |
| **Savings** | None until root opportunity typed. |
| **Lifecycle** | Incident: open → root-caused → linked opportunity / dismissed. |
| **Phase** | 2 |

### `COST_REGRESSION`

| Field | Spec |
|-------|------|
| **Detection** | G2–G5 changepoints; normalize for size and data growth (G7). |
| **Lifecycle** | Incident + optional `HEAVY_SHAPE` / config change link. |
| **Phase** | 2 |

---

## F. Prerequisites

### `TAG_COVERAGE_GAP`

| Field | Spec |
|-------|------|
| **Detection** | Tag coverage % below threshold (e.g. 30%) for queries/objects needed for chargeback or merge approvals. |
| **Counterfactual** | Improve tagging convention. |
| **Savings** | $0 — **blocker** on A6/A9 quality and sometimes consolidation approval. |
| **Risk** | N/A |
| **Effort** | `governance` |
| **Phase** | 1 (as blocker signal) |

---

## Arbitration priority (summary)

When two types claim the same uptime or conflict logically, apply dependency order from [05](05-savings-accounting.md). Headline rule:

1. Fix isolation breaches before consolidating the same warehouse.  
2. Optimize auto-suspend / anchors before consolidating (claims nest).  
3. Portfolio total = union of uptime claims + non-overlapping non-uptime claims (storage, maintenance).

---

## Count checklist

| Category | Types |
|----------|-------|
| Compute | IDLE_AUTOSUSPEND, ZOMBIE_WAREHOUSE, UPTIME_ANCHOR_SHAPE, CONSOLIDATE_WAREHOUSES, ISOLATE_WORKLOAD, RIGHTSIZE_DOWN, RIGHTSIZE_UP, MULTICLUSTER_MAX_REDUCTION, RESUME_THRASH |
| Query | RELENTLESS_SHAPE, HEAVY_SHAPE, POLLING_ON_STATIC_DATA, FAILED_RETRY_WASTE, CLUSTERING_KEY_CANDIDATE |
| Feature | FEATURE_NEGATIVE_ROI, DEAD_BUT_MAINTAINED, FEATURE_ADD_POSITIVE_ROI |
| Storage | REDUNDANT_TABLE_FAMILY, RETENTION_OVERSIZED, COLD_TABLE_ARCHIVE |
| Incidents | COST_ANOMALY, COST_REGRESSION |
| Prerequisite | TAG_COVERAGE_GAP |

**Total: 23 types** (closed set; extend only with a full spec).
