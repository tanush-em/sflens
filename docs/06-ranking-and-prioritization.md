# 06 — Ranking & Prioritization

**Purpose:** Define how the central FinOps backlog is sorted. Primary persona: one global ranked list.

---

## Why naive J6 fails

Catalog meta-KPI:

```text
J6 ≈ J1 × J2 / (J3 × J4)
```

Problems in practice:

1. **Huge unaddressable items float up** — big warehouse, no owner, political merge.  
2. **Low-effort tiny wins drown** if J1 is only point estimate without floors.  
3. **Risk and effort are ordinal**, not interval scales — multiplying fake precision.  
4. **Incidents** are urgent but not “savings.”  
5. **Tradeoffs** (RIGHTSIZE_UP) shouldn’t compete on savings sort.

---

## Ranking pipeline

```text
1. Filter: status = open opportunity (post-arbitration winners)
2. Gate: addressability J5 must pass (or soft-score)
3. Bucket: Quick wins | Structural | Governance | Tradeoffs | Incidents
4. Score within bucket
5. Optional: round-robin by owner to avoid one team dominating
```

---

## Addressability gate (J5)

An opportunity is addressable if **all** hold:

| Check | Example |
|-------|---------|
| Named owner role or team | Warehouse owner tag, last ALTER role, QUERY_TAG team |
| Change type matches authority | Config → platform; SQL → app team |
| Not blocked by prerequisite | `TAG_COVERAGE_GAP` for merges needing chargeback redesign |
| Reversible or low blast radius **or** risk accepted | |
| Not suppressed by arbitration | |

If owner unknown: status `needs_owner` — visible but **sorted below** addressable items (or filtered out of default backlog).

---

## Buckets (UI sections)

| Bucket | Types (examples) | Sort philosophy |
|--------|------------------|-----------------|
| **Quick wins** | IDLE_AUTOSUSPEND, DEAD_BUT_MAINTAINED, FAILED_RETRY_WASTE, RESUME_THRASH, FEATURE_NEGATIVE_ROI | Cheapest credible win first |
| **Structural** | CONSOLIDATE, ISOLATE, UPTIME_ANCHOR, RELENTLESS/HEAVY | Savings × confidence / risk |
| **Governance** | TAG_COVERAGE_GAP, storage retire candidates | Unblock value; not $ sort |
| **Tradeoffs** | RIGHTSIZE_UP | Separate; sort by SLA urgency |
| **Incidents** | COST_ANOMALY, COST_REGRESSION | Recency × severity, not J1 |

Default home view for FinOps: **Quick wins + Structural** merged with a composite score, Governance as banner if blockers exist, Incidents as alert rail.

---

## Composite score (within savings buckets)

Use **rank-normalized** components (percentile within account), not raw multiplies of ordinals.

```text
priority_score =
    w1 * pctile(savings_p50_credits)
  + w2 * pctile(confidence_weight)
  + w3 * pctile(1 / effort_rank)
  + w4 * pctile(1 / risk_rank)
  + w5 * addressability_bonus
  − w6 * days_open_penalty? (optional)
```

Suggested defaults (tune in YAML):

| Weight | Value | Notes |
|--------|-------|-------|
| w1 | 0.40 | Money |
| w2 | 0.20 | exact > natural_experiment > observational > modelled |
| w3 | 0.15 | config > query > architecture > governance |
| w4 | 0.15 | |
| w5 | 0.10 | Binary or owner-known |

**Confidence weights:** exact=1.0, natural_experiment=0.85, observational=0.65, modelled=0.45.

**Effort ranks:** config=1, query_change=2, architecture=3, governance=4.

**Risk ranks:** low=1, med=2, high=3.

### “Cheapest credible win first” overlay

For Quick wins bucket, secondary key:

```text
sort_key = savings_p50 / effort_rank
```

so a 50-credit config change can outrank a 200-credit architecture fantasy when in Quick wins.

---

## Consolidation-specific ranking

Among `CONSOLIDATE_WAREHOUSES` candidates:

```text
consolidation_rank =
    consolidation_score (from catalog §6.3)
  × savings_p50
  × confidence
```

with **hard filter** C6, C9. Never show infeasible merges “for completeness” in default backlog (keep in suppressed with reason).

---

## Tie-breaks

1. Higher confidence class  
2. Lower risk  
3. Lower effort  
4. Broader evidence (more observation days)  
5. Stable id (warehouse name) for determinism  

---

## Per-owner slicing

Central backlog remains canonical. Optional views:

- Filter `owner = X`  
- **Fairness mode:** round-robin top-N across owners so one team’s 40 warehouses don’t bury others  

Not required for v1 UI; data model should store `owner_key`.

---

## What not to rank on

| Bad signal | Why |
|------------|-----|
| Bytes scanned | Not cost |
| Raw AVG_RUNNING | Not utilization |
| Card count | Vanity |
| LLM “importance” | Non-deterministic |
| Sum of overlapping claims | Accounting lie |

---

## Exec summary metrics (not sort keys)

For leadership home:

1. Portfolio addressable credits (de-duplicated p50 + band)  
2. Verified savings MTD / QTD  
3. Realization rate by type  
4. Ignored recommendation cost (see [07](07-verification.md))  
5. Attribution coverage A8  

---

## Config knobs (stack YAML)

```yaml
ranking:
  weights: { w1: 0.40, w2: 0.20, w3: 0.15, w4: 0.15, w5: 0.10 }
  floors:
    min_credits_p50: 1.0
  addressability_required_for_default_view: true
  compose_counterfactuals: false  # phase 1
```
