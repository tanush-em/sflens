# 01 — Product Vision

## One-line definition

**SnowLens** is a least-privilege, advisory Snowflake FinOps layer that turns account telemetry into a **ranked, non-double-counted backlog of cost/performance opportunities**, with **passive verification** of savings after humans make changes.

---

## What problem it solves

At enterprise scale (AT&T CDO Prod as the POC account), Snowflake already shows the bill. The hard problem is:

> Which ~50 changes, out of 100,000+ signals, are worth making — and can we prove they worked?

SnowLens is for a **central platform / FinOps** team that needs one globally ranked backlog, not a prettier chart. Product owners remain the decision-makers; SnowLens is the mirror and the ledger.

---

## What SnowLens is

| Capability | Description |
|------------|-------------|
| Cost intelligence | Reconcile spend to the invoice; attribute idle vs query; publish honesty metrics (attribution coverage). |
| Warehouse optimizer | Idle / auto-suspend, consolidation, right-sizing, multi-cluster headroom, resume thrash. |
| Query / workload optimizer | Repeated shapes, uptime anchors, polling on static data, failed-retry waste, priced native insights. |
| Workload intelligence | Classify ETL / BI / ad-hoc / batch; gate risk; reconcile consolidation vs isolation. |
| Feature ROI | Turn off negative-ROI clustering / search optimization / MVs; flag dead-but-maintained objects. |
| Storage (delegated) | Consume Tech Debt Radar outputs as opportunity types; do not reimplement storage clustering. |
| Anomaly & regression | Spike → warehouse → shape → cause decomposition (incidents, not savings). |
| Opportunity center | Detected → priced → arbitrated → ranked → (human) acted → passively verified. |
| Savings ledger | Estimated vs verified; realization rates by type; cost of ignored recommendations. |

---

## What SnowLens is not

| Non-goal | Why |
|----------|-----|
| Another Snowsight cost dashboard | Visibility without prioritization and proof is already available. |
| Automatic warehouse resize / ALTER | v1 is **strictly advisory**; no execution role. |
| ACCOUNTADMIN / unrestricted SQL | Least privilege; metadata + usage only. |
| Raw SQL egress to external LLMs | Parse to features inside the boundary; LLM sees sanitized features only (optional). |
| Rebuild of Snowflake billing | Anchor to metering; do not invent a parallel invoice. |
| Reimplementation of `QUERY_INSIGHTS` | Consume Snowflake’s plan-based anti-patterns; differentiate on **pricing, ranking, verification**. |
| Forced action on teams | Advisory only — `review` / `retire_candidate` / `recommended`, never `delete`. |
| Storage-first product | Storage is real but smaller money; lead with compute (idle + consolidation). |

---

## Positioning

### vs native Snowsight / cost views

Snowsight answers “what did we spend?” SnowLens answers “what should we change, in what order, with what evidence, and did it actually save?”

### vs `QUERY_INSIGHTS`

Snowflake now ships structured anti-pattern detection (`insight_type_id`, suggestions, `is_opportunity`). SnowLens does **not** compete on detection quality. It:

1. Prices the insight in credits / dollars (marginal uptime contribution, not bytes alone).
2. Ranks it against warehouse-level opportunities in one backlog.
3. Verifies outcomes after humans change SQL or config.

### vs the existing Tech Debt Radar

| Layer | Owner | Relation |
|-------|--------|----------|
| Storage redundancy / table families | Existing engine (`object_fingerprint`, `redundancy_score`, …) | SnowLens **imports** candidates via an integration contract |
| Compute cost attribution | Partially built (`warehouse_ledger`, `query_shape_spend`) | SnowLens **extends** into efficiency, consolidation, simulation, arbitration, verification |
| Config / stack YAML | Existing pattern | Reused; SnowLens is another stack consumer |

---

## Primary persona (v1)

**Central platform / FinOps engineer**

- Sees one account-wide (later org-wide) ranked backlog.
- Filters by opportunity type, warehouse, owner, risk, effort.
- Opens an opportunity card: counterfactual, evidence, gates passed, claimed uptime slice, confidence.
- Tracks verified savings and ignored recommendations over time.
- Does **not** need per-team sandbox views in v1 (those can be later slices of the same store).

Secondary consumers (read-only slices later): product owners (“the mirror”), leadership (spend + addressable + verified).

---

## Product thesis (locked)

1. **Unit economics first.** Bill for warehouse uptime, not queries. Marginal cost of a query on an awake warehouse ≈ 0.
2. **One semantic core.** Almost every compute decision is a question about warehouse-seconds → one interval ledger + one counterfactual simulator.
3. **KPIs have jobs.** Driver / Gate / Evidence — most KPIs never become user-facing “findings.”
4. **Opportunities, not metrics.** Closed catalog of ~20 types; 90 KPIs are machinery underneath.
5. **No double-counting.** Portfolio savings = union of claimed uptime (and non-uptime claims), not sum of cards.
6. **Advisory + passive verification.** Prove value without write privileges.
7. **Deterministic core first.** No ML required for phase-1 money (idle, auto-suspend, consolidation, uptime anchors, dead-but-maintained).

---

## Success metrics for the product itself

| Metric | Target sense |
|--------|----------------|
| Verified savings (J8) | Non-zero within first quarters of use |
| Realization rate (J7) by type | Published; used to calibrate future estimates |
| Attribution coverage (A8) | Published every run; honesty with the bill |
| Recommendation ignore cost | Visible; creates urgency without automation |
| Time-to-first-credible-opportunity | Days after Tier-1 access, not months |

---

## Access posture (summary)

- Prefer `USAGE_VIEWER` + `OBJECT_VIEWER`; request `GOVERNANCE_VIEWER` and org roles as separate phases.
- Never require table `SELECT` on business data.
- `SHOW WAREHOUSES` snapshot every ingest (config has no ACCOUNT_USAGE history).
- Full detail: [11](11-access-tiers-and-degradation.md), [12](12-security-and-privacy.md), [16](16-kpi-source-access-matrix.md).

---

## Demo narrative (what Puneet should remember)

Open with:

1. **Idle / uptime waste** — exact idle credits from metering; optimal auto-suspend from gap CDF.
2. **Warehouse consolidation** — interval union vs sum; low temporal overlap + affinity; concurrency gate.
3. Then: uptime-anchor shapes, dead-but-maintained, priced insights, savings ledger (estimated → verified).

Close with: “We didn’t change Snowflake. Someone resized this warehouse after our card; we measured **verified** savings against control warehouses.”
