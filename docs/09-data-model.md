# 09 — Data Model (Postgres)

**Purpose:** Physical schema sketch for staging, dimensions, interval ledger, marts, opportunities, ledger, provenance.

Conventions: `timestamptz`, credits as `numeric`, IDs as `uuid` or text natural keys where stable. All analytical tables carry `run_id`.

---

## Schemas

| Schema | Role |
|--------|------|
| `stg` | Raw landed extracts |
| `dim` | Conformed dimensions |
| `fact` | Interval ledger, occupancy |
| `mart` | KPI aggregates |
| `app` | Opportunities, ledger, user actions |
| `meta` | Runs, watermarks, access probe |

---

## meta

### `meta.runs`

```sql
create table meta.runs (
  run_id          uuid primary key,
  stack_id        text not null,
  started_at      timestamptz not null,
  finished_at     timestamptz,
  git_sha         text,
  extract_as_of   timestamptz,  -- snowflake time cursor
  status          text,         -- running|success|failed
  notes           jsonb
);
```

### `meta.extract_watermarks`

```sql
create table meta.extract_watermarks (
  stack_id    text,
  view_name   text,
  last_ts     timestamptz,
  last_run_id uuid,
  primary key (stack_id, view_name)
);
```

### `meta.access_probe_results`

```sql
create table meta.access_probe_results (
  probed_at   timestamptz,
  stack_id    text,
  object_name text,
  ok          boolean,
  error       text,
  primary key (probed_at, stack_id, object_name)
);
```

---

## stg (examples)

Land as-is from Snowflake; column names snake_case mirrors source.

- `stg.warehouse_metering_history`  
- `stg.warehouse_load_history`  
- `stg.warehouse_events_history`  
- `stg.metering_history` / `stg.metering_daily_history`  
- `stg.query_history_slim` (only needed columns / sampled)  
- `stg.aggregate_query_history`  
- `stg.query_attribution_history`  
- `stg.query_insights`  
- `stg.table_storage_metrics`  
- `stg.access_history_agg` (if governance)  
- `stg.warehouse_config_snapshot`  
- `stg.org_usage_in_currency_daily` (if org)  
- `stg.storage_opportunities_import`  

Each staging table: `run_id`, `_loaded_at`.

---

## dim

### `dim.warehouse`

```sql
create table dim.warehouse (
  warehouse_sk   bigserial primary key,
  stack_id       text not null,
  warehouse_name text not null,
  warehouse_id   text,  -- if available
  unique (stack_id, warehouse_name)
);
```

### `dim.warehouse_config_history`

```sql
create table dim.warehouse_config_history (
  stack_id           text,
  warehouse_name     text,
  valid_from         timestamptz,
  valid_to           timestamptz,  -- null = current
  size               text,
  auto_suspend       integer,
  auto_resume        boolean,
  min_cluster_count  integer,
  max_cluster_count  integer,
  scaling_policy     text,
  warehouse_type     text,
  raw                jsonb,
  run_id             uuid
);
```

### `dim.date`

Standard date dimension for joins / DoW strata.

### `dim.query_shape`

```sql
create table dim.query_shape (
  shape_sk                 bigserial primary key,
  stack_id                 text,
  query_parameterized_hash text,
  sample_query_id          text,
  fingerprint_features     jsonb,  -- sanitized, no raw SQL required
  unique (stack_id, query_parameterized_hash)
);
```

---

## fact

### `fact.warehouse_uptime_interval`

Canonical ledger ([03](03-semantic-layer.md)):

```sql
create table fact.warehouse_uptime_interval (
  interval_id            uuid primary key,
  run_id                 uuid not null,
  stack_id               text not null,
  warehouse_name         text not null,
  cluster_number         integer not null default 0,
  start_ts               timestamptz not null,
  end_ts                 timestamptz not null,
  size                   text,
  credits_per_hour       numeric,
  theoretical_credits    numeric,
  reconciled_credits     numeric,
  open_ended             boolean default false,
  source                 text
);
create index on fact.warehouse_uptime_interval (stack_id, warehouse_name, start_ts);
```

### `fact.interval_occupancy`

```sql
create table fact.interval_occupancy (
  run_id           uuid,
  interval_id      uuid,
  shape_sk         bigint,
  query_id         text,
  overlap_seconds  numeric,
  role_name        text,
  user_name        text
);
```

### `fact.busy_bucket`

```sql
create table fact.busy_bucket (
  run_id          uuid,
  warehouse_name  text,
  bucket_start    timestamptz,  -- 5 min
  avg_running     numeric,
  avg_queued_load numeric,
  avg_queued_provisioning numeric,
  avg_blocked     numeric,
  is_busy         boolean
);
```

---

## mart

### `mart.warehouse_daily`

B1–B2, credits, idle, A8 components, F4 hourly rollups, etc.

### `mart.shape_period`

E1–E7 inputs: executions, credits, cadence stats, uptime attribution.

### `mart.consolidation_pair`

C3–C10 scores for candidate pairs/groups.

### `mart.kpi_latest`

Generic key-value or typed columns for API convenience:

```sql
create table mart.kpi_value (
  run_id      uuid,
  kpi_id      text,      -- A8, B1, ...
  grain_type  text,      -- account|warehouse|shape|table
  grain_key   text,
  period_start date,
  period_end   date,
  value       numeric,
  extras      jsonb
);
```

---

## app

### `app.opportunities`

```sql
create table app.opportunities (
  opportunity_id     uuid primary key,
  stack_id           text not null,
  run_id_created     uuid not null,
  run_id_updated     uuid,
  type               text not null,
  status             text not null,
  title              text,
  warehouse_names    text[],
  shape_hash         text,
  object_refs        jsonb,
  owner_key          text,
  estimated_credits_p10 numeric,
  estimated_credits_p50 numeric,
  estimated_credits_p90 numeric,
  estimated_usd_p50  numeric,
  confidence_class   text,
  risk               text,
  effort             text,
  priority_score     numeric,
  bucket             text,
  counterfactual     jsonb,
  expected_change    jsonb,
  evidence           jsonb,
  gate_results       jsonb,
  claim              jsonb,
  suppression_reason text,
  observed_at        timestamptz,
  dismiss_reason     text,
  created_at         timestamptz,
  updated_at         timestamptz
);
create index on app.opportunities (stack_id, status, priority_score desc);
```

### `app.opportunity_claims`

Normalized claims for arbitration audit:

```sql
create table app.opportunity_claims (
  claim_id         uuid primary key,
  opportunity_id   uuid references app.opportunities,
  claim_class      text,
  warehouse_name   text,
  interval_set     jsonb,  -- or use side table of ranges
  credits_p50      numeric,
  won              boolean
);
```

### `app.savings_ledger`

```sql
create table app.savings_ledger (
  ledger_id          uuid primary key,
  opportunity_id     uuid,
  opportunity_type   text,
  estimated_p50      numeric,
  verified_point     numeric,
  verified_p10       numeric,
  verified_p90       numeric,
  status             text,
  pre_start          date,
  pre_end            date,
  post_start         date,
  post_end           date,
  control_keys       jsonb,
  computed_at        timestamptz,
  run_id             uuid
);
```

### `app.user_actions`

Dismiss / assign owner (application-level only).

---

## Retention

| Data | Keep |
|------|------|
| Staging raw | 7–30 days (or drop after transform) |
| Facts / marts | 400+ days aligned to Snowflake retention |
| Opportunities | Full history of status transitions (append-only events table recommended) |
| Ledger | Indefinite |

Optional `app.opportunity_events` for status transition audit.

---

## Migration approach

Use numbered SQL migrations in `sql/postgres/` (e.g. Alembic or plain files). Schema evolves with opportunity types via `type` text + jsonb flexibility rather than one table per type.
