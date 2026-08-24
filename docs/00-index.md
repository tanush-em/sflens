# SnowLens Documentation Index

**Product:** SnowLens — Intelligent Snowflake FinOps  
**Audience:** Platform / FinOps engineers building or reviewing the product  
**Status:** Product design package (pre-implementation)  
**Related sources:** [`kpi-catalog.md`](../kpi-catalog.md), [`discussion.md`](../discussion.md), [`current-kpis-&-existing-project.md`](../current-kpis-&-existing-project.md), [`pitch.html`](../pitch.html)

---

## How to read this set

Read in order the first time. After that, use the matrix and opportunity catalog as day-to-day references.

| # | Doc | One-paragraph summary |
|---|-----|------------------------|
| 00 | [This index](00-index.md) | Reading order, glossary, conventions. |
| 01 | [Product vision](01-product-vision.md) | What SnowLens is and is not; positioning vs Snowsight / `QUERY_INSIGHTS`; central FinOps persona; explicit non-goals. |
| 02 | [Decision model](02-decision-model.md) | How raw KPIs become decisions: Signal → Candidate → Opportunity → Verified. Driver / Gate / Evidence classification for every KPI family. Worked example. |
| 03 | [Semantic layer](03-semantic-layer.md) | The warehouse-second interval ledger and Counterfactual Uptime Simulator — the single engine under almost every compute opportunity. |
| 04 | [Opportunity catalog](04-opportunity-catalog.md) | Closed set of ~20 opportunity types with detection, counterfactual, savings formula, gates, evidence, risk, verification, failure modes. |
| 05 | [Savings accounting](05-savings-accounting.md) | Uptime budget, claim arbitration, dependency order, conflict table, bootstrap ranges — why portfolio total is a **union**, not a sum. |
| 06 | [Ranking & prioritization](06-ranking-and-prioritization.md) | Backlog sort key, addressability gate, why naive `J1×J2/(J3×J4)` fails, tie-breaks, per-owner slicing. |
| 07 | [Verification](07-verification.md) | Passive change detection with zero write privileges; control groups; difference-in-differences; savings ledger; ignored-recommendation cost. |
| 08 | [System design](08-system-design.md) | FastAPI + React + Postgres; extract/analyze split; Snowflake-side aggregation; scheduling; stack YAML; integration with Tech Debt Radar. |
| 09 | [Data model](09-data-model.md) | Physical Postgres schemas: staging, dims, interval fact, marts, opportunity store, ledger, provenance. |
| 10 | [KPI implementation spec](10-kpi-implementation-spec.md) | Per-KPI computation: columns, grain, window, NULLs, edge cases. |
| 11 | [Access tiers & degradation](11-access-tiers-and-degradation.md) | Four access tiers and what the product can/cannot do at each — build before the DBA answers. |
| 12 | [Security & privacy](12-security-and-privacy.md) | Least-privilege role request; SQL sanitization; LLM boundary; audit. |
| 13 | [UI & UX](13-ui-and-ux.md) | Screens, backlog, opportunity card anatomy, drill-downs, “why believe this” panel. |
| 14 | [Roadmap](14-roadmap.md) | Solo-dev phasing over 6+ months; consolidation + idle waste in phase 1. |
| 15 | [Risks & traps](15-risks-and-traps.md) | Catalog traps plus product traps: double-counting, Goodhart, control contamination. |
| 16 | [KPI source & access matrix](16-kpi-source-access-matrix.md) | **Take this to the DBA.** Every final KPI → schema → view → columns → role → tier → dependent opportunities. Reverse index included. |
| — | [access-probe.sql](access-probe.sql) | Run against your role; prints pass/fail for every required view/column. |

---

## Design decisions locked in (context for all docs)

| Decision | Choice |
|----------|--------|
| Product relation to Tech Debt Radar | **Extend** — reuse extract/analyze, stack YAML; storage engine is a module with a defined contract |
| v1 form factor | **FastAPI + React + Postgres** |
| Primary persona | **Central platform / FinOps** — one ranked backlog across the account |
| Ambition | **Strictly advisory** — no ALTER, no execution role; **passive verification** still ships |
| Access today | **Unknown** — everything is access-tiered with graceful degradation |
| Scale | **Unknown** — push aggregation into Snowflake by default |
| LLM | **Narrow & optional** — sanitized features only; explains already-computed recommendations |
| Headline differentiators | **Warehouse consolidation** and **idle / uptime waste** are non-negotiable; other useful decisions are kept, phased |

---

## Glossary

| Term | Meaning |
|------|---------|
| **Warehouse-second** | One second of billed warehouse (or cluster) uptime. The atomic unit of compute cost. |
| **Interval ledger** | Canonical fact of billed uptime intervals, reconstructed from events and reconciled to metering. |
| **Counterfactual simulator** | Replays the interval ledger under a hypothetical (different auto-suspend, merge warehouses, drop a shape) and diffs credits. |
| **Driver** | KPI that feeds a savings number. |
| **Gate** | KPI that decides whether a recommendation is legal (risk / feasibility). |
| **Evidence** | KPI shown to justify a recommendation; never used as a savings driver alone. |
| **Signal** | Threshold crossed on a Driver (or Gate that implies a Driver). |
| **Candidate** | Signal with a named counterfactual. |
| **Opportunity** | Candidate that cleared gates, was priced (p10–p90), and survived arbitration. |
| **Uptime budget** | Shared pool of claimable billed seconds; opportunities claim non-overlapping slices. |
| **Passive verification** | Detecting that a human acted on a recommendation via config snapshots / events, then measuring with diff-in-diff — no write privileges. |
| **Access tier** | Capability level of the service role (T0–T3); see [11](11-access-tiers-and-degradation.md). |
| **Realization rate (J7)** | Verified savings / estimated savings, tracked per opportunity type. |
| **Addressability** | Whether anyone can and will own the change; gates ranking. |

---

## Conventions

- KPI IDs (`A1`, `B13`, `C4`, …) match [`kpi-catalog.md`](../kpi-catalog.md).
- Opportunity type IDs are `SCREAMING_SNAKE` (e.g. `CONSOLIDATE_WAREHOUSES`).
- Credits are the internal unit; dollars only appear when Organization billing or a manual rate card exists.
- Ranges are always **p10–p90** unless labelled “point estimate (internal)”.
- Verdict language is advisory: `review`, `retire_candidate`, `recommended` — never `delete` or auto-execute.
