# 16 — KPI Source & Access Matrix

**Purpose:** Take this to the DBA / platform admin. For every KPI used in the product: formula summary, Snowflake schema/view, key columns, role, access tier, latency/retention, and which opportunity types break without it.

**How to use**

1. Run [`access-probe.sql`](access-probe.sql) as the proposed service role.  
2. Mark each view Pass/Fail.  
3. Use the **reverse index** to see exactly which opportunities you lose.  
4. Request the smallest grant that unlocks your next roadmap phase.

Verified against the catalog’s August 2026 view inventory; re-check after Snowflake changes.

---

## Role legend

| Code | Snowflake role / privilege | Tier |
|------|----------------------------|------|
| UV | `SNOWFLAKE.USAGE_VIEWER` | T1 |
| OV | `SNOWFLAKE.OBJECT_VIEWER` | T2 |
| GV | `SNOWFLAKE.GOVERNANCE_VIEWER` | T3 |
| OUV | `ORGANIZATION_USAGE_VIEWER` | T4 |
| OBV | `ORGANIZATION_BILLING_VIEWER` | T4 |
| OAV | `ORGANIZATION_ACCOUNTS_VIEWER` | T4 |
| SHOW | Ability to run `SHOW WAREHOUSES` + `RESULT_SCAN` | T1* |
| MON | Warehouse `MONITOR` (Information Schema near-real-time; optional) | opt |

\*SHOW is not a SNOWFLAKE DB role; confirm separately.

---

## Part A — Final KPI → source → role

### Foundation (A)

| KPI | Formula (short) | Schema.View | Key columns | Role | Tier | Latency / retention | Opportunities needing it |
|-----|-----------------|-------------|-------------|------|------|---------------------|---------------------------|
| A1 | SUM credits by service | `ACCOUNT_USAGE.METERING_HISTORY` | `SERVICE_TYPE`, `CREDITS_USED`, `START_TIME` | UV | T1 | ~2h / 365d | All (bill anchor) |
| A2 | Currency / usage rate | `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY`, `RATE_SHEET_DAILY` | `USAGE`, `USAGE_IN_CURRENCY`, `RATING_TYPE` | OBV/OUV | T4 | ~24h | $ display; else YAML |
| A3 | CS used − adjustment | `ACCOUNT_USAGE.METERING_DAILY_HISTORY` | `CREDITS_USED_CLOUD_SERVICES`, `CREDITS_ADJUSTMENT_CLOUD_SERVICES` | UV | T1 | ~2h / 365d | Honesty; suppress false CS waste |
| A4 | SUM credits by WH | `ACCOUNT_USAGE.WAREHOUSE_METERING_HISTORY` | `WAREHOUSE_NAME`, `CREDITS_USED`, `CREDITS_USED_COMPUTE`, `START_TIME` | UV | T1 | 3h / 365d | Focus, ranking |
| A5 | Attributed by user/role | `ACCOUNT_USAGE.QUERY_ATTRIBUTION_HISTORY` | `USER_NAME`, `ROLE_NAME`, `CREDITS_ATTRIBUTED_COMPUTE` | UV | T1 | 6–8h / 365d | Chargeback views |
| A6 | By tag / cost centre | Attribution ⋈ `TAG_REFERENCES` | tags | UV+OV | T2 | — | Team rollup; gated by coverage |
| A7 | Gini over shapes | Attribution by hash | `QUERY_PARAMETERIZED_HASH`, credits | UV | T1 | 6–8h | Evidence |
| A8 | attributed_queries / used_compute | `WAREHOUSE_METERING_HISTORY` | `CREDITS_ATTRIBUTED_COMPUTE_QUERIES`, `CREDITS_USED_COMPUTE` | UV | T1 | 3h | **Always publish** |
| A9 | attributed + idle share | derived A8 + attribution | — | UV | T1 | — | Fair chargeback |

### Warehouse efficiency (B)

| KPI | Formula (short) | Schema.View | Key columns | Role | Tier | Latency / retention | Opportunities |
|-----|-----------------|-------------|-------------|------|------|---------------------|---------------|
| B1 | used_compute − attributed_queries | `WAREHOUSE_METERING_HISTORY` | same as A8 | UV | T1 | 3h / 365d | IDLE_*, ZOMBIE |
| B2 | B1 / used_compute | derived | — | UV | T1 | — | IDLE_* signals |
| B3 | AVG_RUNNING | `WAREHOUSE_LOAD_HISTORY` | `AVG_RUNNING`, `START_TIME`, `WAREHOUSE_NAME` | UV | T1 | 3h / 365d | CONSOLIDATE, evidence |
| B4 | AVG_QUEUED_LOAD | `WAREHOUSE_LOAD_HISTORY` | `AVG_QUEUED_LOAD` | UV | T1 | 3h | RIGHTSIZE gates, C6 |
| B5 | AVG_QUEUED_PROVISIONING | `WAREHOUSE_LOAD_HISTORY` | `AVG_QUEUED_PROVISIONING` | UV | T1 | 3h | RESUME_THRASH |
| B6 | AVG_BLOCKED | `WAREHOUSE_LOAD_HISTORY` | `AVG_BLOCKED` | UV | T1 | 3h | Gate (not sizing) |
| B7 | Uptime hours | `WAREHOUSE_EVENTS_HISTORY` → ledger | `EVENT_NAME`, `TIMESTAMP`, warehouse | UV | T1 | 3h / 365d | Simulator, CONSOLIDATE |
| B8 | COUNT RESUME | `WAREHOUSE_EVENTS_HISTORY` | `RESUME_WAREHOUSE` | UV | T1 | 3h | RESUME_THRASH, IDLE |
| B9 | Cluster-hours | Events SPINUP/SPINDOWN | cluster events | UV | T1 | 3h | MULTICLUSTER, cost |
| B10 | max − p99 clusters | SHOW + events | `max_cluster_count` | SHOW+UV | T1* | snapshot | MULTICLUSTER_MAX |
| B11 | remote spill / scanned | `QUERY_HISTORY` | `BYTES_SPILLED_TO_REMOTE_STORAGE`, `BYTES_SCANNED` | UV | T1 | 45m / 365d | RIGHTSIZE_* |
| B12 | local spill share | `QUERY_HISTORY` | `BYTES_SPILLED_TO_LOCAL_STORAGE` | UV | T1 | 45m | Evidence |
| B13 | argmin gap cost(T) | `QUERY_HISTORY` gaps + SHOW | start/end times, `auto_suspend` | UV+SHOW | T1* | 45m | IDLE_AUTOSUSPEND |
| B14 | right-size verdict | events + query + load | RESIZE events, spill, queue | UV+SHOW | T1* | — | RIGHTSIZE_* |

### Consolidation (C)

| KPI | Formula (short) | Schema.View | Key columns | Role | Tier | Opportunities |
|-----|-----------------|-------------|-------------|------|------|---------------|
| C1 | Busy buckets | `WAREHOUSE_LOAD_HISTORY` | `AVG_RUNNING>0` | UV | T1 | CONSOLIDATE |
| C2 | Uptime sets | Events / ledger | — | UV | T1 | CONSOLIDATE |
| C3 | Busy Jaccard | derived C1 | — | UV | T1 | CONSOLIDATE |
| C4 | Σuptime − \|⋃\| × rate | C2 + size | — | UV+SHOW | T1* | CONSOLIDATE |
| C5 | p99 combined load | Load aligned | — | UV | T1 | CONSOLIDATE |
| C6 | C5 ≤ capacity | C5 + config | min/max clusters, size | SHOW | T1* | **Hard gate** |
| C7 | Table Jaccard | `ACCESS_HISTORY` | objects accessed | GV | T3 | Score; **degrades** without |
| C8 | Role overlap | `QUERY_HISTORY` | `ROLE_NAME` | UV | T1 | Gate |
| C9 | SLA class conflict | F1 | — | UV | T1 | **Hard gate** |
| C10 | Size spread | `QUERY_HISTORY` / SHOW | `WAREHOUSE_SIZE` | UV/SHOW | T1 | Score |

### Query efficiency (D)

| KPI | View | Key columns | Role | Tier | Opportunities |
|-----|------|-------------|------|------|---------------|
| D1 | `QUERY_ATTRIBUTION_HISTORY` | `CREDITS_ATTRIBUTED_COMPUTE`, `QUERY_ID` | UV | T1 | HEAVY, RELENTLESS |
| D2 | `QUERY_HISTORY` | `BYTES_SCANNED`, `BYTES_WRITTEN_TO_RESULT`, `QUERY_TYPE` | UV | T1 | HEAVY score |
| D3 | `QUERY_HISTORY` | `PARTITIONS_SCANNED`, `PARTITIONS_TOTAL` | UV | T1 | Evidence |
| D4 | `QUERY_HISTORY` | remote spill, scanned | UV | T1 | Gates |
| D5 | `QUERY_HISTORY` | `PERCENTAGE_SCANNED_FROM_CACHE` | UV | T1 | Autosuspend risk |
| D6 | `QUERY_HISTORY` | queued_* / elapsed | UV | T1 | Evidence |
| D7 | `QUERY_HISTORY` | `COMPILATION_TIME` | UV | T1 | Evidence |
| D8 | `QUERY_HISTORY` | rows_* | UV | T1 | Evidence |
| D9 | QH ⋈ attribution | `EXECUTION_STATUS` | UV | T1 | FAILED_RETRY_WASTE |
| D10 | `QUERY_HISTORY` | `QUERY_RETRY_TIME` | UV | T1 | FAILED_RETRY_WASTE |
| D11 | `QUERY_INSIGHTS` | `INSIGHT_TYPE_ID`, `SUGGESTIONS`, `IS_OPPORTUNITY`, `QUERY_ID` | UV† | T1† | HEAVY pricing |
| D12 | `TABLE_PRUNING_HISTORY` | partitions pruned/scanned | UV | T1 | Clustering ROI |
| D13 | `COLUMN_QUERY_PRUNING_HISTORY` | column pruning contrib | UV | T1 | CLUSTERING_KEY_CANDIDATE |

†Confirm `QUERY_INSIGHTS` included in USAGE_VIEWER on your edition; probe it.

**Adaptive substitute:** `QUERY_METERING_HISTORY` when attribution NULL — UV, T1.

### Recurring (E)

| KPI | View | Key columns | Role | Tier | Opportunities |
|-----|------|-------------|------|------|---------------|
| E1 | `AGGREGATE_QUERY_HISTORY` | `QUERY_PARAMETERIZED_HASH`, counts | UV | T1 | RELENTLESS/HEAVY |
| E2 | Attribution by hash | credits | UV | T1 | same |
| E3 | E2/E1 | — | UV | T1 | archetype |
| E4 | derived | — | UV | T1 | routing |
| E5 | `QUERY_HISTORY` | timestamps | UV | T1 | POLLING |
| E6 | QH ⋈ `TABLE_DML_HISTORY` | DML times | UV | T1 | POLLING_ON_STATIC_DATA |
| E7 | Simulator + QH | hash | UV | T1 | UPTIME_ANCHOR_SHAPE |

### Workload class (F)

| KPI | View | Key columns | Role | Tier | Opportunities |
|-----|------|-------------|------|------|---------------|
| F1 | `QUERY_HISTORY` (+ `SESSIONS` if avail) | `QUERY_TYPE`, roles, client | UV | T1 | All risk gates, ISOLATE, C9 |
| F2 | derived | — | UV | T1 | ISOLATE_WORKLOAD |
| F3 | QH / SESSIONS | arrival pattern | UV | T1 | Autosuspend risk |
| F4 | `WAREHOUSE_METERING_HISTORY` | hour×DoW | UV | T1 | Controls, consolidate |
| F5 | `WAREHOUSE_LOAD_HISTORY` | p99/median load | UV | T1 | MULTICLUSTER |

### Anomaly (G)

| KPI | View | Role | Tier | Opportunities |
|-----|------|------|------|---------------|
| G1 | `WAREHOUSE_METERING_HISTORY` | UV | T1 | COST_ANOMALY |
| G2 | Attribution by hash | UV | T1 | COST_REGRESSION |
| G3 | `TABLE_PRUNING_HISTORY` | UV | T1 | Regression cause |
| G4/G5 | `AGGREGATE_QUERY_HISTORY` | UV | T1 | Regression cause |
| G6 | `WAREHOUSE_EVENTS_HISTORY` | UV | T1 | Config correlation |
| G7 | `TABLE_STORAGE_METRICS` over time | UV/OV | T1–T2 | Normalize scan growth |
| G8 | ACCESS + attribution | GV+UV | T3 | New-object cost |
| G9 | `METERING_DAILY_HISTORY` | UV | T1 | Forecast variance |

### Storage (H) — product + Radar

| KPI | View | Role | Tier | Opportunities |
|-----|------|------|------|---------------|
| H1 | `TABLE_STORAGE_METRICS` | UV/OV | T1–T2 | Storage types |
| H2/H3 | same | UV/OV | T1–T2 | Evidence |
| H4 | `TABLES` + `TABLE_DML_HISTORY` | OV+UV | T2 | RETENTION_OVERSIZED |
| H5 | `ACCESS_HISTORY` | GV | T3 | COLD_*, DEAD_BUT_MAINTAINED |
| H6 | ACCESS | GV | T3 | Write-hot guard |
| H7 | `OBJECT_DEPENDENCIES` | OV | T2 | Orphan / retire gate |
| H8 | `COLUMNS` | OV | T2 | Radar families |
| H9 | ACCESS | GV | T3 | Access similarity |
| H10 | DML + TABLES | UV+OV | T2 | RETENTION_OVERSIZED |
| H11 | `STAGE_STORAGE_USAGE_HISTORY` | UV | T1 | Storage hygiene |
| H12 | ACCESS columns | GV | T3 | Unused columns |
| H13 | ACCESS history | GV | T3 | COLD_TABLE_ARCHIVE |

### Feature ROI (I)

| Feature | Cost view | Benefit view | Role | Tier | Opportunities |
|---------|-----------|--------------|------|------|---------------|
| Auto clustering | `AUTOMATIC_CLUSTERING_HISTORY` | `TABLE_PRUNING_HISTORY` | UV | T1 | FEATURE_*, DEAD_BUT_MAINTAINED, CLUSTERING_* |
| Search opt | `SEARCH_OPTIMIZATION_HISTORY` | `SEARCH_OPTIMIZATION_BENEFITS` | UV | T1 | FEATURE_* |
| QAS | `QUERY_ACCELERATION_HISTORY` | `QUERY_ACCELERATION_ELIGIBLE` | UV | T1 | FEATURE_* |
| MVs | `MATERIALIZED_VIEW_REFRESH_HISTORY` | modelled | UV | T1 | FEATURE_* |
| Dynamic tables | `DYNAMIC_TABLE_REFRESH_HISTORY` | modelled | UV | T1 | optional |
| Snowpipe | `PIPE_USAGE_HISTORY` | — | UV | T1 | optional |

### Config (not ACCOUNT_USAGE)

| Need | Source | Role | Tier | Opportunities |
|------|--------|------|------|---------------|
| auto_suspend, clusters, size | `SHOW WAREHOUSES` | SHOW | T1* | IDLE_AUTOSUSPEND, MULTICLUSTER, RIGHTSIZE, C6 |
| Tags | `TAG_REFERENCES` | OV | T2 | A6, TAG_COVERAGE_GAP, consolidate politics |

### Organization (optional)

| Need | View | Role | Tier |
|------|------|------|------|
| $ and rates | `USAGE_IN_CURRENCY_DAILY`, `RATE_SHEET_DAILY` | OBV/OUV | T4 |
| Account inventory | org accounts views | OAV | T4 |
| Cross-account usage | `ORGANIZATION_USAGE.*` | OUV | T4 |

---

## Part B — Opportunity → minimum access

| Opportunity | Min tier | Must-have views | Breaks / degrades if missing |
|-------------|----------|-----------------|------------------------------|
| IDLE_AUTOSUSPEND | T1* | Metering, QH gaps, SHOW | Without SHOW: cannot know current T |
| ZOMBIE_WAREHOUSE | T1 | Metering, load/events | — |
| UPTIME_ANCHOR_SHAPE | T1 | QH + events/ledger | — |
| CONSOLIDATE_WAREHOUSES | T1* | Load, events, metering, SHOW for C6 | Without GV: C7 degraded |
| ISOLATE_WORKLOAD | T1 | QH for F1/F2 | — |
| RIGHTSIZE_DOWN/UP | T1* | QH spill, load, SHOW/events | Weak without resize history |
| MULTICLUSTER_MAX | T1* | Events + SHOW | Hard fail without SHOW |
| RESUME_THRASH | T1 | Events, load | — |
| RELENTLESS / HEAVY | T1 | Agg QH + attribution | Insights optional |
| POLLING_ON_STATIC_DATA | T1 | QH + TABLE_DML_HISTORY | — |
| FAILED_RETRY_WASTE | T1 | QH + attribution | — |
| CLUSTERING_KEY_CANDIDATE | T1 | COLUMN_QUERY_PRUNING + AC history | — |
| FEATURE_NEGATIVE_ROI | T1 | Feature cost + benefit views | — |
| DEAD_BUT_MAINTAINED | T1–T3 | Maintenance + **H5 ACCESS** for strong | Degrades without GV |
| FEATURE_ADD_POSITIVE_ROI | T1 | Eligibility views | — |
| REDUNDANT_TABLE_FAMILY | import or T2 | Radar / TABLES+COLUMNS | — |
| RETENTION_OVERSIZED | T2 | TABLES + DML + storage | — |
| COLD_TABLE_ARCHIVE | T3 | ACCESS + storage + deps | Hard degrade without GV |
| COST_ANOMALY | T1 | Metering | — |
| COST_REGRESSION | T1 | Attribution / agg QH | — |
| TAG_COVERAGE_GAP | T2 | TAG_REFERENCES | — |

---

## Part C — Reverse index: view → role → dependents

| View | Role | If FAIL, you lose / weaken |
|------|------|----------------------------|
| `METERING_HISTORY` | UV | A1 bill by service |
| `METERING_DAILY_HISTORY` | UV | A3, G9 |
| `WAREHOUSE_METERING_HISTORY` | UV | **Idle, A4, A8, F4, G1 — Phase 1 blocked** |
| `WAREHOUSE_LOAD_HISTORY` | UV | **Consolidation busy sets, queues, B3–B6** |
| `WAREHOUSE_EVENTS_HISTORY` | UV | **Ledger, B7–B9, verification of resizes** |
| `QUERY_HISTORY` | UV | Gaps B13, spill, F1, fails, roles |
| `AGGREGATE_QUERY_HISTORY` | UV | Scale path for E1 |
| `QUERY_ATTRIBUTION_HISTORY` | UV | D1, E2, chargeback; adaptive may NULL |
| `QUERY_METERING_HISTORY` | UV | Adaptive substitute |
| `QUERY_INSIGHTS` | UV† | Priced anti-patterns |
| `TABLE_STORAGE_METRICS` | UV/OV | H1 storage $ |
| `TABLE_DML_HISTORY` | UV | E6, H4/H10 |
| `TABLE_PRUNING_HISTORY` | UV | D12, clustering ROI, G3 |
| `COLUMN_QUERY_PRUNING_HISTORY` | UV | D13 |
| `AUTOMATIC_CLUSTERING_HISTORY` | UV | I / DEAD_BUT_MAINTAINED |
| `SEARCH_OPTIMIZATION_*` | UV | Feature ROI |
| `QUERY_ACCELERATION_*` | UV | Feature ROI |
| `MATERIALIZED_VIEW_REFRESH_HISTORY` | UV | Feature ROI |
| `TABLES` / `COLUMNS` / `OBJECT_DEPENDENCIES` / `TAG_REFERENCES` | OV | Storage, tags, orphans |
| `ACCESS_HISTORY` | GV | **Cold, C7 strong, H12, writers** |
| `ORGANIZATION_USAGE.*` | OUV/OBV/OAV | Dollars, multi-account |
| `SHOW WAREHOUSES` | SHOW | **Config-dependent optimizers** |

---

## Part D — Checklist for DBA conversation

Copy/paste:

- [ ] Service user created (no human ACCOUNTADMIN reuse)  
- [ ] `USAGE_VIEWER` granted  
- [ ] `OBJECT_VIEWER` granted  
- [ ] `SHOW WAREHOUSES` works  
- [ ] Extract warehouse XS granted  
- [ ] Probe script all T1 objects PASS  
- [ ] (Later) `GOVERNANCE_VIEWER`  
- [ ] (Later) Org viewers if same organization  
- [ ] Confirm edition: Enterprise for ACCESS_HISTORY  
- [ ] Confirm whether adaptive warehouses are in use  

---

## Part E — Final KPI set used by SnowLens (concise)

**Always on (T1):** A1, A3, A4, A8, B1–B9, B11–B14 (B13/B14 need SHOW), C1–C6, C8–C10, D1–D10, E1–E5, E7, F1–F5, G1–G2, G6, selected I cost views, J1–J9 (app).

**T2 adds:** A6 quality, H1/H4/H7/H8/H10, TAG_COVERAGE_GAP, stronger storage.

**T3 adds:** C7 strong, H5/H6/H9/H12/H13, G8, strong DEAD_BUT_MAINTAINED / COLD_*.

**T4 adds:** A2 from org, multi-account.

**Consumed not reinvented:** D11 `QUERY_INSIGHTS`; Radar storage scores.

**Explicitly deferred as Drivers until phased:** full modelled latency elasticity; CI/CD cost gates; autonomous execution.
