# 14 — Roadmap

**Purpose:** Solo-developer phasing over 6+ months. Each phase is demoable. Idle waste + consolidation are phase-1 non-negotiable.

**Constraints:** Advisory only; FastAPI+React+Postgres; extend Tech Debt Radar; access unknown → T1-first.

---

## Phase 0 — Foundations (weeks 1–3)

**Ship**

- Repo layout, stack YAML, Postgres migrations  
- [`access-probe.sql`](access-probe.sql) + results in `meta`  
- Synthetic fixtures for ledger + simulator unit tests  
- Extract stubs for metering, load, events  

**Demo:** Probe report; empty UI shell with tier banner.

**Exit:** Know actual access tier.

---

## Phase 1a — Billing anchor + idle (weeks 3–7)

**Ship**

- Extract metering + events + SHOW snapshot  
- Interval ledger + reconciliation  
- KPIs A1, A8, B1, B2, B7, B8  
- Simulator: `ChangeAutoSuspend`  
- Opportunities: `IDLE_AUTOSUSPEND`, `ZOMBIE_WAREHOUSE`, `RESUME_THRASH`  
- Arbitration stub (single-type claims)  
- Summary + backlog API/UI  

**Demo:** “Idle credits and exact auto-suspend recommendation on WH X.”

---

## Phase 1b — Consolidation + shapes (weeks 7–12)

**Ship**

- Load history busy sets; C1–C6, C8–C10  
- C7 degraded affinity  
- `CONSOLIDATE_WAREHOUSES` + conflict with `ISOLATE_WORKLOAD`  
- Shape marts; `UPTIME_ANCHOR_SHAPE`, `RELENTLESS_SHAPE`, `HEAVY_SHAPE` (basic), `FAILED_RETRY_WASTE`  
- Uptime budget arbitration across idle + consolidate + anchors  
- Portfolio de-duplicated headline  
- Tech Debt Radar import → `REDUNDANT_TABLE_FAMILY`  
- `TAG_COVERAGE_GAP` if OBJECT_VIEWER  

**Demo:** Consolidation union-vs-sum + idle in one ranked backlog without double-count.

**Phase 1 exit criteria**

- T1 data live  
- ≥1 consolidation candidate and ≥1 autosuspend candidate on real account  
- Honest A8 published  

---

## Phase 2 — Depth (months 3–5)

**Ship**

- Passive verification + savings ledger + ignored cost  
- Anomalies `COST_ANOMALY` / `COST_REGRESSION` with decomposition  
- `POLLING_ON_STATIC_DATA`  
- Feature ROI: `DEAD_BUT_MAINTAINED`, `FEATURE_NEGATIVE_ROI`  
- Right-sizing `RIGHTSIZE_DOWN` (evidence hierarchy), `MULTICLUSTER_MAX_REDUCTION`  
- D11 QUERY_INSIGHTS pricing attach  
- Stronger C7 if GOVERNANCE granted  
- Storage `RETENTION_OVERSIZED`, `COLD_TABLE_ARCHIVE`  
- Ranking calibration with early J7  
- Optional LLM explain  

**Demo:** “Recommended → observed → verified” on a real autosuspend change someone made.

---

## Phase 3 — Breadth (months 5–7+)

**Ship**

- `RIGHTSIZE_UP` tradeoff lane  
- `CLUSTERING_KEY_CANDIDATE`, `FEATURE_ADD_POSITIVE_ROI`  
- Compose counterfactuals (autosuspend before consolidate re-price)  
- Org tier if T4  
- Multi-stack YAML accounts  
- UX polish: consolidation explorer, fairness owner mode  
- Realization dashboards for leadership  

**Still out of scope unless strategy changes:** execution role, cost-aware CI/CD PR bots (can be a separate proposal).

---

## Explicit non-goals (entire horizon under current ambition)

- Autonomous ALTER  
- ACCOUNTADMIN  
- Rebuilding Snowsight billing  
- Reimplementing QUERY_INSIGHTS detection  
- Full lineage platform  

---

## Dependency on access

| If stuck at… | Do this |
|--------------|---------|
| No access | Synthetic + docs; unblock with probe ticket |
| T1 only | Complete Phase 1; Phase 2 minus cold-table strength |
| T1+T2 | Full Phase 1–2 except access-affinity |
| T3 | Full storage + C7 |
| T4 | $ and multi-account |

---

## Suggested demo script for Puneet (anytime after 1b)

1. Home: credits, A8, portfolio addressable ≠ sum of cards  
2. Idle card: gap CDF, T*, range  
3. Consolidation card: overlap heatmap, C6 gate, union savings  
4. Uptime anchor: cheap query, expensive awake time  
5. Coverage: what we still can’t see  
6. (Phase 2+) Ledger verified line  

Close: advisory mirror + path to verified savings without admin rights.
