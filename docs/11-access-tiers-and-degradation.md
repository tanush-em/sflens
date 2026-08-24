# 11 — Access Tiers & Degradation

**Purpose:** Build before the DBA answers. Define four tiers, what works at each, and graceful degradation.

**Today’s access:** unknown — assume Tier 0 until [`access-probe.sql`](access-probe.sql) passes.

---

## Tier overview

| Tier | Capabilities | Typical grants |
|------|--------------|----------------|
| **T0** | Nothing verified | — |
| **T1** | Core compute FinOps: idle, auto-suspend, consolidation (load+events), shapes, metering | `USAGE_VIEWER` + ability to `SHOW WAREHOUSES` |
| **T2** | Objects, dependencies, stronger storage/feature metadata | + `OBJECT_VIEWER` |
| **T3** | Access lineage, cold tables, affinity via tables touched | + `GOVERNANCE_VIEWER` (Enterprise) |
| **T4** | Dollars + multi-account rollup | + `ORGANIZATION_*` viewers |

(T4 is org; label as Tier 4 for clarity even if docs sometimes say T0–T3 — **use T0–T4** here.)

---

## What each tier unlocks

### T0 — Probe failed / no access

| Can | Cannot |
|-----|--------|
| Develop against synthetic fixtures | Any production opportunity |
| Ship UI mock | |

### T1 — USAGE_VIEWER (+ SHOW WAREHOUSES)

| Opportunity types | Status |
|-------------------|--------|
| IDLE_AUTOSUSPEND | Full |
| ZOMBIE_WAREHOUSE | Full |
| RESUME_THRASH | Full |
| CONSOLIDATE_WAREHOUSES | **Partial**: C1–C6, C8–C10 from query/load; C7 affinity degraded |
| UPTIME_ANCHOR_SHAPE | Full (query timeline) |
| RELENTLESS / HEAVY / FAILED_RETRY | Full / strong |
| RIGHTSIZE_* | Partial (spill/queue; weaker natural experiments) |
| MULTICLUSTER_MAX | Needs config snapshot |
| COST_ANOMALY / REGRESSION | Partial |
| DEAD_BUT_MAINTAINED | Weak without access+object maintenance joins |
| Dollars | Only with manual rate card |
| Storage Radar import | If files provided externally |

**Phase 1 non-negotiables (idle + consolidation) are designed to be T1-buildable.**

### T2 — + OBJECT_VIEWER

| Unlocks |
|---------|
| TABLES / COLUMNS / OBJECT_DEPENDENCIES / TAG_REFERENCES |
| Better ownership, orphan signals, tag coverage (`TAG_COVERAGE_GAP`) |
| Storage metrics joins for reclaim messaging |
| Feature object metadata |

### T3 — + GOVERNANCE_VIEWER

| Unlocks |
|---------|
| ACCESS_HISTORY / aggregates |
| Strong H5 coldness, H12 unused columns, C7 table Jaccard |
| Build-credits style graphs (Radar parity) |
| DEAD_BUT_MAINTAINED high confidence |

Without T3: never claim “0 reads = safe to delete”; use weaker proxies and Kaplan-Meier only if history allows.

### T4 — Organization usage / billing / accounts

| Unlocks |
|---------|
| Real currency (A2 from org) |
| Cross-account discovery |
| Org anomaly rollups |

Without T4: single-account credits + YAML rate card.

---

## Feature matrix (summary)

| Feature | T1 | T2 | T3 | T4 |
|---------|----|----|----|----|
| Cost dashboard (credits) | ✓ | ✓ | ✓ | ✓ |
| Cost dashboard ($) | rate card | rate card | rate card | ✓ org |
| Idle / auto-suspend | ✓ | ✓ | ✓ | ✓ |
| Consolidation | ✓* | ✓* | ✓ full C7 | ✓ |
| Query shapes / anchors | ✓ | ✓ | ✓ | ✓ |
| QUERY_INSIGHTS pricing | if view granted under usage | ✓ | ✓ | ✓ |
| Tag coverage | weak | ✓ | ✓ | ✓ |
| Cold table / access affinity | ✗ | weak | ✓ | ✓ |
| Org rollup | ✗ | ✗ | ✗ | ✓ |
| Write / ALTER | ✗ forever in v1 | ✗ | ✗ | ✗ |

\*C7 degraded without T3.

---

## Runtime degradation behavior

On each run:

1. Read latest `meta.access_probe_results` (or probe inline).  
2. Set `stack.access_tier` and `capabilities` flags.  
3. Disable opportunity detectors whose `required_tier` not met.  
4. UI banner: “Running at Tier 1 — table-affinity consolidation scoring limited.”  
5. Never fail the whole pipeline because GOVERNANCE was refused.

### Capability flags (examples)

```yaml
capabilities:
  attribution_columns: true|false
  show_warehouses: true|false
  access_history: true|false
  org_currency: true|false
  query_insights: true|false
```

---

## SHOW WAREHOUSES special case

Config is **not** in ACCOUNT_USAGE. If SHOW fails:

- Still compute idle from metering.  
- Suppress IDLE_AUTOSUSPEND **recommendation of new T** if current auto_suspend unknown (or infer poorly from events only).  
- Suppress MULTICLUSTER_MAX.  
- Consolidation size target less reliable.

Document SHOW as **Tier-1 hard requirement** for full warehouse optimizer.

---

## Request sequencing (to DBA)

1. Ask: same Snowflake org? Org usage available?  
2. Request dedicated role: `USAGE_VIEWER` + `OBJECT_VIEWER` (no ACCOUNTADMIN).  
3. Confirm SHOW WAREHOUSES for service user.  
4. Later: `GOVERNANCE_VIEWER`.  
5. Later: org billing/usage/accounts viewers.  

Template language: [12](12-security-and-privacy.md).

---

## Mapping to roadmap

| Roadmap phase | Min tier |
|---------------|----------|
| Phase 1 idle + consolidation + shapes | T1 |
| Phase 1 storage import | Artifacts; T2 preferred |
| Phase 2 storage-native + ROI + anomalies | T2–T3 |
| Phase 3 org + richer CI stories | T4 optional |
