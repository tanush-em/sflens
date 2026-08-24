# 02 — Decision Model

**Purpose:** Answer the core product question — how do raw KPIs become meaningful insights, data, and business decisions?

---

## The gap

The KPI catalog measures. Measurement alone does not decide.

| KPI alone | What a decision needs |
|-----------|------------------------|
| Idle share = 43% | Counterfactual: if auto_suspend = 60s, billed seconds change by Δ |
| Temporal overlap = 0.12 | Feasibility: merged p99 concurrency ≤ capacity; SLA classes compatible |
| Query cost score = 91 | Addressability: does this shape cause warehouse-seconds to exist? |

SnowLens promotes values through a fixed pipeline. Nothing user-facing appears before **Opportunity**.

---

## Pipeline: Signal → Verified

```text
KPI value
   │
   ▼
Signal          threshold / rule fired
   │
   ▼
Candidate       named counterfactual exists
   │
   ▼
Gates           risk / feasibility / prerequisites
   │ fail → Suppressed (reason codes, still queryable for debugging)
   ▼ pass
Priced          p10–p90 credits (+ dollars if rate known)
   │
   ▼
Arbitration     conflicts + uptime-budget claims (no double-count)
   │
   ▼
Opportunity     ranked backlog item
   │
   ▼
Observed        human change detected (passive)
   │
   ▼
Verified        difference-in-differences vs controls → savings ledger
```

### Stage definitions

| Stage | Definition | User-visible? |
|-------|------------|---------------|
| **KPI** | Computed metric at a grain (warehouse×day, shape×week, …) | Internal / drill-down |
| **Signal** | KPI crossed a rule (e.g. B2 idle share ≥ 30% and B1 ≥ 100 credits) | Debug only |
| **Candidate** | Signal + opportunity type + counterfactual parameters | Debug only |
| **Suppressed** | Candidate failed a Gate; reason recorded | Ops / analyst |
| **Priced** | Simulator (or closed-form) produced a savings range | Internal |
| **Opportunity** | Survived arbitration; has claim IDs, priority score | **Yes — backlog** |
| **Observed** | Config snapshot or events show the recommended change happened | Opportunity status |
| **Verified** | Diff-in-diff CI does not span zero (or reported as zero) | **Savings ledger** |

---

## Every KPI has exactly one primary job

| Job | Role | Example |
|-----|------|---------|
| **Driver** | Feeds the savings numerator | B1 idle credits, C4 consolidation saving, E7 uptime attributable to shape |
| **Gate** | Allows / blocks a recommendation | F1 workload class, B11 spill, C6 concurrency feasibility, C9 SLA conflict |
| **Evidence** | Explains and builds trust; never drives dollars alone | D5 cache hit, F5 burstiness, A8 coverage |

Secondary jobs are allowed (e.g. B3 load is Evidence for sizing and a Driver input for consolidation busy sets), but **one primary job** is declared so engineers know whether to put the metric in the savings formula.

### Classification by family (catalog Parts 4–13)

#### Family A — Foundation & spend attribution

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| A1 | Total billed credits | Evidence | Ground truth; reconciliation |
| A2 | Effective credit price | Driver (pricing) | Required for $; fallback rate card |
| A3 | Cloud services billable | Evidence / Gate | Gate: do not flag free allowance as waste |
| A4 | Spend by warehouse | Evidence | Concentration / focus |
| A5 | Spend by user / role | Evidence | Chargeback; allocation only |
| A6 | Spend by team / cost centre | Evidence | **Gated by tag coverage** |
| A7 | Spend concentration (Gini) | Evidence | Few shapes own bill → focus |
| A8 | Attribution coverage | Evidence | Honesty metric — always publish |
| A9 | Fully-loaded query cost | Driver (chargeback) | Idle redistributed; political |

#### Family B — Warehouse efficiency

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| B1 | Idle credits | **Driver** | Core idle pool |
| B2 | Idle share | Signal / Gate | Threshold with absolute floor |
| B3 | Query load | Evidence / Gate | Consolidation busy; **not** utilization % |
| B4 | Queue load | Gate | Blocks downsize / bad merges |
| B5 | Provisioning queue | Gate / Signal | Resume thrash / aggressive suspend |
| B6 | Blocked load | Gate | Not a sizing problem |
| B7 | Uptime hours | Driver input | Denominator; ledger |
| B8 | Resume count | Driver / Signal | 60s minimum waste |
| B9 | Cluster-hours | Driver input | Multi-cluster cost |
| B10 | Max cluster headroom | Driver | MULTICLUSTER_MAX_REDUCTION |
| B11 | Spill rate | **Gate** | Right-size direction |
| B12 | Local spill rate | Evidence | Mild pressure |
| B13 | Optimal auto_suspend | **Driver** | Deterministic simulator |
| B14 | Right-size verdict | **Driver** | Direction + confidence |

#### Family C — Consolidation

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| C1 | Busy interval set | Driver input | From load history |
| C2 | Uptime interval set | Driver input | From events |
| C3 | Pairwise temporal overlap | Driver / Signal | Low overlap → attractive |
| C4 | Consolidation saving | **Driver** | Union vs sum |
| C5 | Combined peak concurrency | Gate input | |
| C6 | Concurrency feasibility | **Gate** | Hard fail if false |
| C7 | Workload affinity | Evidence / Score | Cache warmth |
| C8 | Role/security affinity | Gate | |
| C9 | SLA class conflict | **Gate** | Never merge interactive+batch |
| C10 | Size compatibility | Score | |

#### Family D — Query efficiency

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| D1 | Credits per execution | Driver input | |
| D2 | Scan-to-output ratio | Evidence / Score | Addressability required |
| D3 | Partition pruning ratio | Evidence / Gate | |
| D4 | Remote spill ratio | Gate / Evidence | |
| D5 | Cache hit | Evidence | Symptom, not lever |
| D6 | Queue fraction | Evidence | |
| D7 | Compilation fraction | Evidence | |
| D8 | Row explosion | Evidence | |
| D9 | Failed-query spend | **Driver** | FAILED_RETRY_WASTE |
| D10 | Retry waste | Driver | |
| D11 | Native insight flags | Evidence → priced Driver | Consume, don’t reimplement |
| D12 | Table pruning efficiency | Evidence | |
| D13 | Clustering key candidate | **Driver** | CLUSTERING_KEY_CANDIDATE |

#### Family E — Recurring workload

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| E1 | Executions per shape | Driver input | |
| E2 | Credits per shape | Driver input | |
| E3 | Per-execution cost | Signal | Heavy vs relentless |
| E4 | Waste archetype | Signal | Routes to opportunity type |
| E5 | Cadence regularity | Signal | Polling |
| E6 | Result-identical repeats | **Driver** | POLLING_ON_STATIC_DATA |
| E7 | Uptime attributable to shape | **Driver** | UPTIME_ANCHOR_SHAPE |

#### Family F — Workload classification

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| F1 | Workload class | **Gate** | Risk for every resize/merge |
| F2 | Class purity | Signal / Gate | Isolation vs consolidation |
| F3 | Interactivity index | Gate | |
| F4 | Diurnal profile | Evidence / Matching | Controls, consolidation |
| F5 | Burstiness | Evidence / Gate | |

#### Family G — Anomaly & regression

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| G1–G9 | Anomaly / regression family | **Incident drivers** | Separate lifecycle; not savings portfolio |

#### Family H — Storage

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| H1–H13 | Storage family | Driver / Gate / Evidence per H | Mostly via Tech Debt Radar import |

#### Family I — Feature ROI

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| I1 | Feature ROI ratio | **Driver** | FEATURE_NEGATIVE_ROI / ADD |
| I2 | Dead-but-maintained | **Driver** | Cleanest zero-risk win |

#### Family J — Opportunity meta

| ID | KPI | Primary job | Notes |
|----|-----|-------------|-------|
| J1 | Estimated saving | Driver (meta) | Always range |
| J2 | Confidence | Ranking | |
| J3 | Risk | Ranking / Gate | |
| J4 | Effort | Ranking | |
| J5 | Addressability | **Gate** | Ranking |
| J6 | Priority | Ranking output | Refined in doc 06 |
| J7 | Realization rate | Calibration | |
| J8 | Verified saving | Ledger | |
| J9 | Coverage | Evidence | |

---

## From KPI to opportunity type (routing)

| Signal pattern | Opportunity type |
|----------------|------------------|
| High B1/B2; B13 recommends T ≠ current | `IDLE_AUTOSUSPEND` |
| Near-zero load, sustained credits | `ZOMBIE_WAREHOUSE` |
| High E7 for one shape | `UPTIME_ANCHOR_SHAPE` |
| Low C3, high score, C6 pass | `CONSOLIDATE_WAREHOUSES` |
| Low F2 (mixed classes) | `ISOLATE_WORKLOAD` |
| B14 down + spill≈0 + queue≈0 | `RIGHTSIZE_DOWN` |
| High B11 remote spill | `RIGHTSIZE_UP` |
| B10 large unused headroom | `MULTICLUSTER_MAX_REDUCTION` |
| High B8 + B5 | `RESUME_THRASH` |
| E4 Relentless | `RELENTLESS_SHAPE` |
| E4 Heavy / Both | `HEAVY_SHAPE` |
| E6 + regular E5 | `POLLING_ON_STATIC_DATA` |
| D9/D10 material | `FAILED_RETRY_WASTE` |
| D13 ranked columns | `CLUSTERING_KEY_CANDIDATE` |
| I1 < 0 | `FEATURE_NEGATIVE_ROI` |
| I2 | `DEAD_BUT_MAINTAINED` |
| I1 opportunity to add | `FEATURE_ADD_POSITIVE_ROI` |
| H via storage engine | `REDUNDANT_TABLE_FAMILY`, `RETENTION_OVERSIZED`, `COLD_TABLE_ARCHIVE` |
| G1/G2 | `COST_ANOMALY`, `COST_REGRESSION` (incident) |
| Tag coverage low | `TAG_COVERAGE_GAP` (prerequisite blocker) |

Full specs: [04-opportunity-catalog.md](04-opportunity-catalog.md).

---

## Worked example: idle → decision → (later) verified

### Inputs (raw)

From `WAREHOUSE_METERING_HISTORY` for `ANALYTICS_WH`, last 30d:

- `credits_used_compute` = 8,420
- `credits_attributed_compute_queries` = 4,900  
- → **B1** idle = 3,520 credits; **B2** idle share ≈ 42%

From `QUERY_HISTORY`: inter-query gaps `G`.  
From config snapshot: `auto_suspend = 600`.

### Signal

`B2 ≥ 0.30` and `B1 ≥ 100` → Signal `IDLE_HIGH` on `ANALYTICS_WH`.

### Candidate

Type `IDLE_AUTOSUSPEND`. Counterfactual: set `T* = argmin billed_seconds(T)` over {30,60,…,900} (see catalog §5.3 / semantic layer). Suppose `T* = 120`.

### Gates

| Gate | Check | Result |
|------|-------|--------|
| F1 | Not pure interactive SLA with resume latency sensitivity | Pass (mixed BI, acceptable) |
| Cache | Weight against aggressive T if high `PERCENTAGE_SCANNED_FROM_CACHE` | Pass with mild risk bump |
| Config known | Snapshot has auto_suspend | Pass |

### Priced

Simulator: billed seconds at T=600 vs T=120 → Δ credits p10=280, p50=410, p90=520 over 30d → annualize carefully (bootstrap over days).  
Dollars if A2 known.

### Arbitration

Claim idle seconds that would suspend under T* but not under current T. If a consolidation candidate also claims those seconds, arbitration keeps one claim (see [05](05-savings-accounting.md)).

### Opportunity card (user sees)

- **Do:** Change `ANALYTICS_WH` auto_suspend 600 → 120  
- **Save:** 280–520 credits / 30d (p10–p90)  
- **Risk:** Low–Med  
- **Evidence:** B1, B2, gap CDF chart, resume count, cache %  
- **Owner:** Platform  
- **Status:** Recommended (advisory)

### Observed → Verified (later)

Next ingest: snapshot shows `auto_suspend = 120`. Mark Observed. After 14–28 days, diff-in-diff vs control warehouses → **Verified** line on ledger (or zero if CI spans 0).

---

## What “business decision” means in SnowLens

A business decision is a **human approval** of a specific change, informed by:

1. **Named action** (ALTER parameter, merge warehouse A into B, rewrite shape X, drop maintenance feature, archive table family).
2. **Priced counterfactual** with uncertainty.
3. **Risk / SLA gate** outcome.
4. **Portfolio honesty** (this claim does not double-count another open opportunity).
5. Optionally later: **verified** outcome for calibration.

SnowLens does not make the change. It makes the decision *cheap and defensible*.

---

## Suppression reason codes (examples)

| Code | Meaning |
|------|---------|
| `GATE_SLA_CONFLICT` | C9 / interactive+batch merge blocked |
| `GATE_CONCURRENCY` | C6 failed |
| `GATE_SPILL` | Downsize blocked by remote spill |
| `GATE_TAG_COVERAGE` | Chargeback / merge blocked pending tags |
| `GATE_ADAPTIVE_NULL` | Attribution NULL; use alternate path or suppress |
| `ARB_CLAIM_LOST` | Same uptime claimed by higher-priority type |
| `ADDR_NO_OWNER` | J5 addressability failed |
| `BELOW_FLOOR` | Absolute credits below noise floor |

---

## Principles for engineers implementing this

1. Never surface a Driver without a counterfactual type.
2. Never price without stating the counterfactual in one sentence.
3. Never sum opportunity cards for the headline — use union of claims ([05](05-savings-accounting.md)).
4. Always store Signal → Candidate → Opportunity lineage for audit.
5. Prefer deterministic Drivers in phase 1; put elasticity models behind confidence class “modelled”.
