# 08 — System Design

**Purpose:** Architecture for FastAPI + React + Postgres SnowLens, extending Tech Debt Radar patterns (extract/analyze, stack YAML), with Snowflake-side aggregation by default.

---

## High-level architecture

```text
┌─────────────────────────────────────────────────────────────┐
│                     Snowflake Account(s)                      │
│  ACCOUNT_USAGE · ORGANIZATION_USAGE · SHOW WAREHOUSES         │
└────────────────────────────┬────────────────────────────────┘
                             │ extract (read-only role)
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  Extract workers (Python)                                     │
│  • Push-down SQL aggregations                                 │
│  • Incremental watermarks per view                            │
│  • Config snapshot                                            │
│  • Optional: pull Tech Debt Radar artifacts                   │
└────────────────────────────┬────────────────────────────────┘
                             │ land files / COPY
                             ▼
┌─────────────────────────────────────────────────────────────┐
│  Postgres                                                      │
│  staging → dims → interval ledger → marts → opportunities      │
│  savings ledger · run provenance                               │
└────────────────────────────┬────────────────────────────────┘
                             │
              ┌──────────────┼──────────────┐
              ▼              ▼              ▼
        Analyze jobs    Simulator      Verification
        (KPIs, detect)  (counterfact)  (passive DiD)
              │              │              │
              └──────────────┼──────────────┘
                             ▼
                    FastAPI (JSON APIs)
                             ▼
                    React dashboard
```

Optional narrow LLM: **after** opportunity emit, input = sanitized feature JSON only → explanation text stored as `ai_assisted`.

---

## Design principles inherited from Tech Debt Radar

| Principle | How SnowLens uses it |
|-----------|----------------------|
| EXTRACT ≠ ANALYZE | Expensive Snowflake pull once; re-run scoring without re-query |
| Config-driven stack YAML | Account connection, floors, weights, rate card, enabled opportunity types |
| Advisory verdicts | No delete/execute |
| Provenance | `run_id`, extract timestamps, git SHA when available |
| Partial visibility honesty | Don’t claim “unused” without governance caveats |

---

## Repo layout (target)

```text
snowLens/
  docs/                 # this package
  configs/
    stacks/
      cdo_prod.yaml
  src/
    snowlens/
      extract/          # Snowflake pullers
      transform/        # ledger build, KPI marts
      simulate/         # counterfactual engine
      detect/           # opportunity detectors
      arbitrate/        # claims / conflicts
      verify/           # passive detection + DiD
      integrate/        # Tech Debt Radar import
      api/              # FastAPI
      llm/              # optional explainer
  web/                  # React app
  sql/
    snowflake/          # push-down aggregations
    postgres/           # DDL migrations
  scripts/
    access_probe.py     # wraps docs/access-probe.sql
```

If Tech Debt Radar lives in a sibling repo, treat it as a **dependency contract**, not a mandatory monorepo merge on day one.

---

## Stack YAML (sketch)

```yaml
stack_id: cdo_prod
snowflake:
  account: ...
  role: SNOWLENS_READER
  warehouse: SNOWLENS_XS   # for extract only
  database: SNOWFLAKE      # usage views
lookbacks:
  compute_days: 90
  query_agg_days: 90
  access_days: 365
rate_card:
  source: manual           # or organization
  usd_per_credit: 3.00     # example placeholder
floors:
  min_idle_credits: 100
  min_shape_credits: 1.0
  min_shape_executions: 10
ranking: { ... }           # see doc 06
opportunities:
  enabled: [IDLE_AUTOSUSPEND, CONSOLIDATE_WAREHOUSES, ...]
integration:
  tech_debt_radar:
    enabled: true
    artifact_path: /path/or/s3
llm:
  enabled: false
  mode: explain_only
```

---

## Extract strategy

### Push-down first

Never pull full `QUERY_HISTORY` for 90 days into the app by default. Prefer:

| Need | Snowflake output grain |
|------|------------------------|
| Spend | Metering hourly / daily aggregates |
| Idle | Metering columns already idle-capable |
| Load / consolidation | `WAREHOUSE_LOAD_HISTORY` as-is (5 min) |
| Events | Event stream filtered by time |
| Shapes | `AGGREGATE_QUERY_HISTORY` + attribution by hash |
| Gaps (B13) | Query start/end **only** for warehouses above spend percentile |
| Insights | `QUERY_INSIGHTS` filtered `is_opportunity` |

### Incremental watermarks

Table `extract_watermark(view_name, last_ts, last_run_id)`.

Respect view latency (e.g. don’t expect QUERY_HISTORY fresher than ~45m; attribution 6–8h).

### Config snapshot

Each run:

```sql
SHOW WAREHOUSES;
-- RESULT_SCAN → land as warehouse_config_snapshot
```

Requires a session privilege to run SHOW (MONITOR or ownership patterns — confirm with DBA).

### Dual path (optional later)

Near-real-time Information Schema (7d) for incidents; Account Usage for history. v1 can be Account Usage only.

---

## Analyze pipeline (ordered jobs)

1. Load staging → validate row counts  
2. Build dims (warehouse, date)  
3. Build interval ledger + reconcile metering ([03](03-semantic-layer.md))  
4. Build occupancy / shape marts  
5. Compute KPI marts ([10](10-kpi-implementation-spec.md))  
6. Detect candidates → gate → price via simulator  
7. Arbitrate claims ([05](05-savings-accounting.md))  
8. Rank ([06](06-ranking-and-prioritization.md))  
9. Import storage opportunities  
10. Passive observe + verify due items ([07](07-verification.md))  
11. Optional LLM explanations  
12. Publish API views / refresh flags  

Idempotent on `run_id`: re-analyze same extract allowed.

---

## API surface (v1)

| Method | Path | Purpose |
|--------|------|---------|
| GET | `/v1/summary` | Spend, portfolio addressable, A8, verified MTD |
| GET | `/v1/opportunities` | Filtered backlog |
| GET | `/v1/opportunities/{id}` | Card detail + evidence |
| GET | `/v1/warehouses/{id}` | Warehouse profile |
| GET | `/v1/ledger` | Savings ledger |
| GET | `/v1/runs/{id}` | Provenance |
| POST | `/v1/opportunities/{id}/dismiss` | User dismiss (app DB only) |

No endpoints that execute Snowflake DDL.

---

## Frontend

React SPA:

- Home (FinOps summary)  
- Backlog  
- Opportunity detail  
- Warehouse explorer  
- Savings ledger  
- Access / coverage status (which tier active)

See [13](13-ui-and-ux.md).

---

## Integration contract: Tech Debt Radar

SnowLens does **not** reimplement table fingerprinting.

**Import artifact** (JSON/Parquet) per run:

```text
storage_opportunity:
  external_id
  type: REDUNDANT_TABLE_FAMILY | ...
  object_refs[]
  scores (redundancy_score, coldness, ...)
  true_reclaimable_bytes
  verdict: review | retire_candidate | keep
  link_status
  evidence_uri
  produced_at
  producer_version
```

SnowLens maps → opportunity row, prices storage with rate card, includes in backlog with `claim_class=storage`.

If artifact missing: storage types disabled; compute still runs.

---

## Scheduling

| Job | Cadence |
|-----|---------|
| Extract + analyze | Daily (or 2×/day) |
| Config snapshot | Every extract |
| Verification sweep | Daily |
| Access probe | Weekly / on deploy |

Orchestration: cron, Airflow, or systemd — keep simple for solo.

---

## Scale & performance

- Extract warehouse: X-Small dedicated; monitor its own cost (meta).  
- Postgres: indexes on `(warehouse_id, ts)`, opportunity status, hash.  
- DuckDB optional for local analyze of Parquet extracts before Postgres load.  
- Cap consolidation pair enumeration: only top-N warehouses by spend; blocking on affinity.

---

## Security touchpoints

- Secrets in env / vault, not YAML committed.  
- Network: Snowflake egress only from extract workers.  
- LLM: off by default; no raw SQL.  
- Full detail: [12](12-security-and-privacy.md).

---

## Solo-dev pragmatism

Phase 1 can ship:

- CLI extract/analyze producing Postgres + thin FastAPI  
- Minimal React or even authenticated JSON first  

Do not block on perfect UI before ledger + consolidation + idle work.
