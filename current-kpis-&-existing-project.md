# Tech Debt Radar — Project Chat Session Summary

**Date:** 2026-08-22

---

## Part 1: Project Goal & Vision

### Aim & Goal

The core goal is **cost optimization / spend consolidation on Snowflake** at AT&T. Puneet's bigger vision is to consolidate AT&T's cloud spending across platforms, but this project starts with **one Snowflake account (CDO Prod)** as a proof of concept. 

The idea is to build a **config-driven dashboard that pulls the entire account's metadata (telemetry)** and surfaces where money is being wasted — so product owners can see their "reality" and decide whether to act on it. 

**Importantly, the tool is advisory only** ("we just show them the mirror") — it flags waste but the decision to optimize sits with each team's product owner.

### Two Dimensions: Storage & Compute

The problem is broken into **two dimensions: storage and compute.**

#### Storage
Puneet has already built a dashboard that groups objects into "table families" — clusters of tables with identical DDL (columns, constraints). It identifies:
- **Redundant tables** (UAT/backup/test copies sitting in prod, e.g., a family occupying ~38 TB)
- **Tables with no reads/writes** that are ready to retire or consolidate, freeing up storage

#### Compute
The next phase (your focus) is analyzing **credit consumption at the object/warehouse level** — flagging inefficiencies like:
- Redundant reads
- **Byte spillage** (queries spilling to remote disk)
- Poor **cache hits**
- The ratio of bytes scanned vs. result produced

The plan is to define **custom statistical rules/KPIs** to automatically flag wasteful objects and queries, largely from Snowflake's own telemetry views (Usage Viewer, Object Viewer). Puneet estimates **~75% can be done purely from telemetry** without domain knowledge; the remaining ~25% (the actual decision-making) is handed to product owners.

### One-Line Summary
The project builds a **telemetry-driven dashboard that flags redundant storage and inefficient compute across a Snowflake account**, giving teams visibility into cost-saving/consolidation opportunities without forcing action.

---

## Part 2: Project Assessment

### Is Your Understanding Correct?

**YES — you've got the project right.** Your summary in the goal section matches what's actually built, with one important nuance about the compute side.

#### Where You're Spot-On

1. **The two-dimension framing (storage + compute)** is exactly what the code does.
   - **Storage / redundancy** is fully realized. `object_fingerprint.py` clusters look-alike tables by column-signature hashing + Jaccard similarity, and `redundancy_score.py` ranks each member (cold + unread + unreferenced → retire candidate). That's your "table families / redundant UAT/backup copies / ready-to-retire" idea, implemented deterministically.
   
   - **Advisory-only ("show them the mirror")** is baked in at the code level, not just aspiration. Verdicts are always `review` / `retire_candidate`, **never** `delete` — see the README's explicit note and the `link_status` 4-state model. That's a genuinely good design decision, especially given ACCOUNT_USAGE's ~3h latency and finite retention (0 reads ≠ never used).
   
   - **Config-driven / re-pointable** matches: `cdo_prod.yaml` is the CDO proof-of-concept, and the stack YAML abstraction is what lets it generalize to Puneet's bigger cross-account vision.
   
   - **~75% from telemetry, 25% human** — the code encodes precisely this split. Deterministic scores are "FROZEN"; the LLM enrichment layer is opt-in and every field is tagged `ai_assisted` + `requires_human_review`.

#### The One Place to Sharpen Your Mental Model

Your goal text lists the compute signals as **byte spillage, cache-hit ratio, bytes-scanned-vs-result**. Those specific query-*execution-efficiency* KPIs are **not implemented yet**.

**What the compute layer actually does today** is **cost attribution**, which is a different (and complementary) angle:

- `warehouse_ledger.py` — anchors to the *billed* number (`WAREHOUSE_METERING_HISTORY`) and surfaces the idle-time / reconciliation gap.
- `query_shape_spend.py` — groups repeated queries by `query_parameterized_hash` to catch the "same expensive SELECT run forever" waste.
- Build-credit attribution per object (`QUERY_ATTRIBUTION_HISTORY` ⋈ `ACCESS_HISTORY.OBJECTS_MODIFIED`) — "what compute was burned *producing* this object."

### Accurate Project Status Statement

**Storage redundancy + compute cost *attribution* are built; the compute *efficiency* signals (spillage, cache hit, scan ratio) are still on the roadmap, not in the codebase.** That's consistent with you calling compute "the next phase (your focus)" — just be clear those KPIs are design intent, not yet shipped.

### Architecture Commentary

The architecture is sound and honestly better-disciplined than most internal cost tools:

1. **The EXTRACT/ANALYZE split is the right call** — the expensive Snowflake scan runs once and dumps to disk; analysis is cheap and re-runnable. This is what makes threshold-tuning and adding new KPIs practical without re-burning credits.

2. **The partial-repo-visibility model (the "20-of-300 problem")** is the strongest idea in here. Mining writers org-wide from `ACCESS_HISTORY.OBJECTS_MODIFIED` so "no reference in my repos" never means "unused" is exactly the kind of trap most people fall into. Keep protecting that invariant.

3. **Reproducibility** — recording Snowflake query time + Git commit SHA per run means reports are diffable over time. That's a feature Puneet's "track savings over time" vision will need.

---

## Part 3: Complete KPI Catalog

The engine mines everything from `SNOWFLAKE.ACCOUNT_USAGE` views. KPIs fall into **7 tiers**. All lookback windows are fixed at extract time (reads = 365d, compute = 90d by default).

### Tier 1 — Redundancy / Retirement Score (the Headline Verdict)

The core deterministic score. Each sub-signal is normalized to 0–1 (1 = more retirable), then weighted.

| KPI | Formula | Source Field(s) | What It Indicates |
|---|---|---|---|
| **Coldness** | `norm(days_since_last_access, 30..365)` | `ACCESS_HISTORY` → `MAX(query_start_time)` on `base_objects_accessed` | How long since anyone read it. High = stale. |
| **Unused** | `1 - norm(access_count, 0..1000)` | `ACCESS_HISTORY` → `COUNT(*)` of reads | Low read volume. 0 reads = max signal. |
| **Orphaned** | `1 - norm(dependents, 0..20)` | `OBJECT_DEPENDENCIES` → `COUNT(DISTINCT referencing_object_id)` | Fan-in. Nothing depends on it = safe to drop. |
| **redundancy_score** | `100 × (0.4·cold + 0.35·unused + 0.25·orphaned)` | (weights from stack YAML) | 0–100. ≥70 → `retire_candidate`, ≥40 → `review`, else `keep`. |

**Write-Recency Guard:** If `write_count > 0` and last write ≤ 90d, a `retire_candidate` is capped down to `review`. Indicates a table that's *read-cold but write-hot* (backup/DR/nightly-staging) — retiring it would break live ETL.

---

### Tier 2 — Storage / True Reclaim

| KPI | Formula | Source Field(s) | What It Indicates |
|---|---|---|---|
| **bytes / row_count** | direct | `TABLES.bytes`, `TABLES.row_count` | Active footprint + size context. |
| **true_reclaimable_bytes** | `max(0, active + time_travel + failsafe − clone_retained)` | `TABLE_STORAGE_METRICS.active_bytes / time_travel_bytes / failsafe_bytes / retained_for_clone_bytes` | The *real* space a DROP frees. Active bytes alone under-count (time-travel + 7d fail-safe go too); clone-pinned bytes don't free, so subtracted. |
| **storage_class** | `object_type` + `TABLES.is_iceberg` | derived | `native` (DROP frees FDN storage), `iceberg` (DROP frees ADLS files), `external` (DROP frees nothing — reclaim forced to 0). |

---

### Tier 3 — Column-Level Usage (Informational, Never Changes Verdict)

Computed only for **wide tables**.

| KPI | Formula | Source Field(s) | What It Indicates |
|---|---|---|---|
| **unused_column_ratio** | `confidently_unused / total_columns` | `ACCESS_HISTORY.base_objects_accessed[].columns[]` per-column flatten | How bloated a *live* table is — a refactor candidate. |
| **column_attributed_reads** | count of reads that resolved any column | same | The **masking guard**: if 0, every read was `SELECT *`/view, so columns are `possibly_masked` (unprovable), never `confidently_unused`. |

---

### Tier 4 — Compute-Waste Profile (Per Object, Informational)

Mined by joining the write graph → per-query credits → query shape.

| KPI | Formula | Source Field(s) | What It Indicates |
|---|---|---|---|
| **build_credits** | `SUM(credits_attributed_compute)` per target | `ACCESS_HISTORY.objects_modified` ⋈ `QUERY_ATTRIBUTION_HISTORY.credits_attributed_compute` (join on `query_id`) | Compute $ burned *producing* this table. |
| **write_shapes** | `COUNT(DISTINCT query_parameterized_hash)` | `QUERY_HISTORY.query_parameterized_hash` | Many distinct shapes writing one target = cloned/redundant ETL pipelines. |
| **distinct_writers** | `COUNT(DISTINCT user_name)` | `ACCESS_HISTORY.objects_modified` | Independent producers of one target. |
| **redundant_load** | `write_shapes > 1 OR distinct_writers > 1` | derived | Duplicated load logic. |
| **expensive_but_unread** | `build_credits ≥ 1.0 AND read_count ≤ 5` | derived | **The money finding**: paid real compute to build something almost nobody reads. |

---

### Tier 5 — Maintenance Credits (Recurring Drain)

| KPI | Formula | Source Field(s) | What It Indicates |
|---|---|---|---|
| **auto_clustering_credits** | credits per table | `AUTOMATIC_CLUSTERING_HISTORY` | A dead-but-clustered table reclusters forever — bleeds credits daily. |
| **mv_refresh_credits** | credits per table | materialized-view refresh history | Redundant matview refresh cost. |
| **maintenance_credits** | `clustering + mv_refresh` | derived | A dead object that *still costs credits every day* — louder retire signal than storage alone. |

---

### Tier 6 — Warehouse Spend Ledger (Live, On-Demand)

Anchors to the **billed** number.

| KPI | Formula | Source Field(s) | What It Indicates |
|---|---|---|---|
| **total_billed_credits** | `SUM(credits_used)` by service | `METERING_HISTORY.credits_used / service_type` | The actual bill, by service type. |
| **per-warehouse credits** | `SUM(credits_used)` etc. | `WAREHOUSE_METERING_HISTORY.credits_used / credits_used_compute / credits_used_cloud_services` | Where warehouse spend concentrates. |
| **attributed_credits** | `SUM(credits_attributed_compute)` | `QUERY_ATTRIBUTION_HISTORY` | How much of the bill maps to specific queries. |
| **unattributed_credits / idle_share_pct** | `billed − attributed`; `/billed` | derived | Idle warehouse time + sub-threshold queries. **idle_flagged** when ≥100 credits and idle_share ≥30% → auto-suspend/oversizing smell. |
| **coverage_pct** | `measured_build_credits / billed_scaled` | reconcile | Honesty metric: how much of real spend this tool can explain (read compute, idle time, serverless are named gaps). |

---

### Tier 7 — Repeated-Query Spend (Live, On-Demand)

Pivots from object to **query**. Groups `QUERY_ATTRIBUTION_HISTORY` by `query_parameterized_hash`.

| KPI | Formula | Source Field(s) | What It Indicates |
|---|---|---|---|
| **executions / credits per shape** | `COUNT(*)`, `SUM(credits)` grouped by hash | `QUERY_ATTRIBUTION_HISTORY.query_parameterized_hash` | The single biggest bill line most tools miss: the same SELECT run forever by a dashboard/poller. |
| **per_exec credits** | `credits / executions` | derived | Splits the lever: **heavy** (≥0.5/exec → tune the query) vs **relentless** (≥10k execs → cut cadence/cache). |
| **consolidation_hint** | threshold logic | derived | Names the actual fix, since "costs a lot" isn't an action. |

**Filters:** `MIN_EXECUTIONS=10` (below = ad-hoc human noise), `MIN_CREDITS=1.0` (below = rounding error).

---

### Cross-Cutting: link_status (Ownership Under Partial Repo Access)

Pairs repo grep with org-wide writer attribution (`ACCESS_HISTORY.objects_modified` → `writers` array):

| State | Meaning |
|---|---|
| `linked` | Loader found in a visible repo |
| `owned_elsewhere` | No repo hit, but has org-wide writers → loader lives in a repo you can't clone |
| `candidate_orphan` | No repo hit, no writers, cold → REVIEW (never auto-delete) |
| `unknown` | Some reads, no identifiable owner |

---

## Part 4: Design Principles

**The consistent design rule across every tier:** telemetry signals (Tiers 4–7 especially) are **advisory** — verdicts say `review`/`retire_candidate`, never `delete`, because `ACCOUNT_USAGE` has ~3h latency and finite retention, so "0 reads" means "none seen in the window," not "provably unused."

---

## Summary

This tech debt radar project is a **mature, well-architected telemetry-driven cost optimization tool** that:

1. ✅ Cleanly separates **storage** (redundancy/dedup) from **compute** (cost attribution)
2. ✅ Implements a deterministic, **reproducible** scoring engine grounded in Snowflake ACCOUNT_USAGE telemetry
3. ✅ Handles **partial repo visibility** correctly (the 20-of-300 problem)
4. ✅ Keeps the tool **advisory-only** by design
5. ✅ Separates cheap re-runnable **ANALYZE** from expensive one-time **EXTRACT**
6. ⏳ Leaves **compute efficiency** signals (spillage, cache-hit, scan ratio) as future work

The 7-tier KPI catalog provides a comprehensive foundation for expanding into the efficiency dimension when ready.
