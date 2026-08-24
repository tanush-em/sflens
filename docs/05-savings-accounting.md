# 05 — Savings Accounting

**Purpose:** Make the headline number honest. Prevent double-counting. Define claim arbitration, dependency order, ranges, and realization.

---

## The failure mode

Two true statements:

1. Fixing auto-suspend on `WH_A` saves ~400 idle credits / 30d.  
2. Consolidating `WH_A` into `WH_B` saves ~600 credits / 30d of overlapping uptime waste.

Sum = 1,000. Reality ≤ max of compatible scenarios ≈ 600–700. **Summing cards destroys trust.**

---

## Uptime budget

Treat billed warehouse-seconds (reconciled credits convertible to seconds via size rate) as a **shared budget**.

### Claim

Every uptime-affecting opportunity attaches one or more **claims**:

| Field | Meaning |
|-------|---------|
| `claim_id` | UUID |
| `opportunity_id` | Parent |
| `warehouse_id` (+ cluster) | |
| `time_range` or interval set | Seconds being “taken off the bill” under the counterfactual |
| `credits_p10/p50/p90` | Monetized claim |
| `claim_class` | `uptime` \| `maintenance` \| `storage` \| `query_attributed` \| `tradeoff` |

**Rule:** Within `claim_class = uptime`, claims that overlap in (warehouse × time) **must be arbitrated** so at most one winning claim owns each second (or ownership is explicitly partitioned).

Non-uptime classes:

- **maintenance** — AC/SO/MV credits (can stack with uptime if truly independent).
- **storage** — bytes × rate (independent of compute uptime).
- **query_attributed** — only allowed when **not** also claiming the same uptime seconds (prefer uptime claim when E7-style).
- **tradeoff** — RIGHTSIZE_UP style; excluded from “savings” headline; shown separately as investment.

---

## Portfolio totals

```text
headline_addressable_credits =
    credits(union of winning uptime claims)
  + credits(maintenance claims)
  + credits(storage claims)
  + credits(query_attributed claims that do not overlap uptime claims)

headline ≠ Σ(opportunity card point estimates)
```

UI must show:

- **Portfolio (de-duplicated)** — headline  
- **Sum of cards (raw)** — struck or secondary, labelled “not additive”

---

## Dependency order (apply before mutual exclusion)

Process candidates in this order; later types see remaining unclaimed budget.

| Priority | Types | Rationale |
|----------|-------|-----------|
| 0 | `TAG_COVERAGE_GAP` | Blocker metadata, $0 |
| 1 | `ISOLATE_WORKLOAD` | Safety before merge |
| 2 | `FAILED_RETRY_WASTE`, incident links | Pure waste |
| 3 | `UPTIME_ANCHOR_SHAPE`, `RESUME_THRASH`, `RELENTLESS_SHAPE` (uptime-linked) | Change gap structure |
| 4 | `IDLE_AUTOSUSPEND`, `ZOMBIE_WAREHOUSE` | Recompute idle on post-gap world **or** claim residual idle |
| 5 | `CONSOLIDATE_WAREHOUSES` | Union on post-autosuspend intervals preferred |
| 6 | `RIGHTSIZE_*`, `MULTICLUSTER_MAX_REDUCTION` | Rate/capacity on resulting topology |
| 7 | Feature ROI / storage / clustering | Mostly independent classes |

**Practical implementation:**  

1. Score all candidates independently (gross).  
2. Run arbitration pass that assigns claims in dependency order.  
3. Re-price consolidation using intervals **after** applying winning suspend/anchor counterfactuals when configured (`compose_counterfactuals: true` recommended for phase 2; phase 1 may use exclusive claims without full compose).

---

## Conflict table

| Conflict | Resolution |
|----------|------------|
| `CONSOLIDATE` vs `ISOLATE` on same WH | If F2 breach severe → keep ISOLATE, suppress CONSOLIDATE (`ARB_ISOLATE_FIRST`). Else suppress ISOLATE if purity OK. |
| `IDLE_AUTOSUSPEND` vs `CONSOLIDATE` overlapping idle seconds | Prefer AUTOSUSPEND claim on those seconds; CONSOLIDATE claims only residual non-idle overlap waste **or** compose (merge after T*). |
| `UPTIME_ANCHOR` vs `IDLE_AUTOSUSPEND` | Anchor first; recompute T* on residual; if both claim same seconds, anchor wins those seconds. |
| `RELENTLESS` attributed $ vs same shape as `UPTIME_ANCHOR` | Single opportunity: prefer UPTIME_ANCHOR if E7 material; else RELENTLESS. |
| Two consolidations sharing a warehouse | Max-weight matching / greedy by C4 after gates; one warehouse in at most one winning merge set. |
| `RIGHTSIZE_DOWN` vs high spill | Already gated; no claim. |
| Storage vs compute | Additive (different claim_class). |
| `FEATURE_NEGATIVE_ROI` vs `DEAD_BUT_MAINTAINED` | Merge into one card if same object. |
| `RIGHTSIZE_UP` | `tradeoff` class — not in savings headline. |

---

## Claim-and-arbitrate algorithm

```text
remaining = full uptime budget (interval set per WH)
winners = []

for type in dependency_order:
  cands = candidates(type) sorted by gross_p50 desc
  for c in cands:
    if fails_gates(c): suppress; continue
    claim = c.proposed_claim ∩ remaining
    if measure(claim) < floor: suppress BELOW_FLOOR; continue
    if logical_conflict(c, winners): suppress; continue
    c.final_claim = claim
    c.priced = price(claim)  # bootstrap
    winners.append(c)
    remaining = remaining − claim

portfolio = aggregate(winners)
```

---

## Bootstrap ranges (J1)

For each winning opportunity:

1. Define atomic units (usually calendar days in lookback).  
2. Resample units with replacement (e.g. 500 draws).  
3. Recompute claim credits each draw.  
4. Store p10, p50, p90.  
5. UI shows **p10–p90**; p50 for sorting only if needed.

Do **not** present a single dollar figure as fact.

---

## Dollars vs credits

```text
dollars = credits × effective_rate
```

- Prefer `ORGANIZATION_USAGE` / `RATE_SHEET_DAILY` / `USAGE_IN_CURRENCY_DAILY` (A2).  
- Else manual rate card in stack YAML.  
- If neither: show **credits only**; never invent $.

---

## Realization factors (J7)

After verified outcomes exist:

```text
realization_rate(type) = sum(verified) / sum(estimated_p50)
```

Apply as **calibration prior** on future estimates (optional phase 2+):

```text
calibrated_p50 = raw_p50 × clip(realization_rate, 0.5, 1.1)
```

Publish rates per type; never hide underperforming types.

---

## Accounting identities to publish every run

| Identity | Formula |
|----------|---------|
| Bill anchor | A1 / metering totals |
| Idle identity | used_compute − attributed_queries ≈ idle |
| Attribution coverage | A8 |
| Portfolio vs raw sum | union claims vs Σ cards |
| Verified vs estimated | ledger |

---

## Anti-patterns

| Don’t | Do |
|-------|-----|
| Sum all opportunity cards for exec slide | Show de-duplicated portfolio |
| Count attributed query savings + idle on same seconds | One claim class wins |
| Annualize 7 days × 52 blindly | Bootstrap; state window |
| Mix tradeoff upsizing into “savings” | Separate “invest for latency” |
| Ignore fail-safe delay on storage | Phrase “after 7 days” for permanent tables |
