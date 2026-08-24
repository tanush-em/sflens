# 13 — UI & UX

**Purpose:** Screen inventory and opportunity card anatomy for the central FinOps persona.

**Stack:** React SPA + FastAPI. Advisory only — no “Apply” that hits Snowflake.

---

## Navigation

| Nav item | Route | Purpose |
|----------|-------|---------|
| Home | `/` | Spend, portfolio savings, A8, verified, ignored cost, tier banner |
| Backlog | `/opportunities` | Ranked opportunities |
| Incidents | `/incidents` | Anomalies / regressions |
| Warehouses | `/warehouses` | Estate browser |
| Ledger | `/ledger` | Estimated vs verified |
| Coverage | `/coverage` | Access tier, tag coverage, attribution gaps |
| Runs | `/runs` | Pipeline provenance |

---

## Home (FinOps)

**Hero metrics**

1. Compute credits (lookback) — anchor to metering  
2. **Portfolio addressable** (de-duplicated p50 + p10–p90 band) — labelled “not sum of cards”  
3. Verified savings (MTD / lookback)  
4. Open opportunities count  
5. Attribution coverage A8  
6. Ignored recommendation cost  

**Rails**

- Incidents needing attention  
- Quick wins (top 5)  
- Blockers (`TAG_COVERAGE_GAP`)  

**Banner** if access tier < T3 / missing SHOW WAREHOUSES.

---

## Backlog

**Filters:** type, bucket, warehouse, owner, risk, effort, status, min credits.

**Default sort:** priority_score within Quick wins + Structural ([06](06-ranking-and-prioritization.md)).

**Row fields:** title, type badge, p50 credits (band on hover), confidence, risk, effort, owner, age, status.

**Bulk actions:** dismiss (with reason), assign owner — app DB only.

**Toggle:** “Show raw sum of cards” (educational; warn not additive).

---

## Opportunity card anatomy

Every card must answer: **What? So what? Why believe? What could go wrong? Who?**

### Header

- Title (human action sentence)  
- Type · Status · Bucket  
- Savings band (credits; $ if rate known)  
- Confidence class · Risk · Effort  

### Counterfactual (required)

One sentence: “If we set auto_suspend 600→120 on ANALYTICS_WH…”

Parameters JSON rendered as readable list.

### Why believe this (“Evidence”)

- KPI chips with values (B1, B2, C3, …)  
- Sparkline / gap CDF / overlap heatmap as relevant  
- Observation days, n executions  
- A8 / data quality footnotes  

### Gates

Pass/fail list with codes (`GATE_CONCURRENCY`, …). Failed gates only on suppressed detail view.

### Portfolio honesty

- Claim class  
- “These seconds are reserved; consolidating WH_A would double-count — consolidation card suppressed / reduced.”  

### Expected change (for verification)

Fingerprint the detector will look for.

### Actions

- Copy recommendation for ticket  
- Dismiss  
- Mark “owner contacted” (optional)  
- **No Execute**  

### Optional explanation

If LLM on: paragraph with badges `ai_assisted`, `requires_human_review`.

---

## Warehouse detail

- Size, config (from snapshot), spend, idle share, load profile, class mix (F1/F2)  
- Open opportunities on this WH  
- Event timeline (resize/suspend)  
- Top shapes by E2 / E7  

---

## Consolidation explorer (phase 1 highlight)

- Pair/group candidate  
- Busy overlap heatmap  
- Union vs sum bar  
- Concurrency vs capacity gauge (C6)  
- Affinity / SLA / roles  

---

## Ledger

Table: type, estimated p50, verified, status, window, controls used.

Summary: realization rates by type.

---

## Coverage page

- Access probe results (pass/fail per view)  
- Current tier  
- Tag coverage %  
- Attribution availability (adaptive NULLs)  
- Which opportunity types disabled  

This page is for the builder + DBA conversation.

---

## Empty & degraded states

| State | UX |
|-------|-----|
| No SHOW WAREHOUSES | Banner; autosuspend recs limited |
| No GOVERNANCE | Cold-table & C7 limited messaging |
| No org billing | Credits only |
| First run | Progress / expected latency callout (Account Usage lag) |

---

## Copy guidelines

- Prefer “recommended” / “candidate” / “review” — never “delete” / “will save exactly $X”.  
- Always show ranges.  
- Always separate **estimated** vs **verified**.  
- Idle is “money for empty seconds,” not “query is expensive.”  

---

## Accessibility & ops

- Keyboard-reachable filters  
- Don’t rely on color alone for risk  
- Export CSV of backlog for offline FinOps rituals  
