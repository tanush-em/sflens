# 12 — Security & Privacy

**Purpose:** Least-privilege access model, SQL sanitization, LLM boundary, audit — suitable for a security-strict enterprise (AT&T-style).

---

## Non-negotiables

| Rule | Detail |
|------|--------|
| No ACCOUNTADMIN / SYSADMIN / GLOBALORGADMIN for the app | |
| No SELECT on business tables | Metadata + usage only |
| No Snowflake writes in v1 | Advisory only |
| No raw SQL to external LLMs | Features only, optional |
| Dedicated service role | `SNOWLENS_READER` (name illustrative) |
| Secrets not in git | Env / vault |

---

## Recommended role grants (phased)

### Phase A — POC (request first)

```text
SNOWFLAKE database roles:
  USAGE_VIEWER
  OBJECT_VIEWER

Plus:
  Privilege to run SHOW WAREHOUSES / RESULT_SCAN
  USAGE on a small extract warehouse
```

### Phase B

```text
GOVERNANCE_VIEWER   -- ACCESS_HISTORY
```

### Phase C (optional)

```text
ORGANIZATION_USAGE_VIEWER
ORGANIZATION_ACCOUNTS_VIEWER
ORGANIZATION_BILLING_VIEWER
```

### Never for v1

```text
MODIFY / OPERATE on customer warehouses
OWNERSHIP
IMPORTED PRIVILEGES (prefer granular DB roles)
```

Snowflake recommends granular SNOWFLAKE database roles over broad imported privileges when avoiding accidental org exposure.

---

## Email / ticket template (paste to DBA)

> **Request:** Read-only Snowflake access for SnowLens FinOps POC  
>  
> We are building an internal **advisory** optimization tool. It analyzes usage telemetry and warehouse configuration metadata only.  
>  
> **We do not need:** ACCOUNTADMIN, SYSADMIN, SELECT on business data, INSERT/UPDATE/DELETE/CREATE/ALTER/DROP on customer objects.  
>  
> **Please create service user + role with:**  
> - `SNOWFLAKE.USAGE_VIEWER`  
> - `SNOWFLAKE.OBJECT_VIEWER`  
> - Ability to run `SHOW WAREHOUSES` and `RESULT_SCAN`  
> - Usage on a dedicated X-Small warehouse for extraction  
>  
> **Later (separate approval):** `GOVERNANCE_VIEWER`; organization usage/billing viewers if accounts share one Snowflake organization.  
>  
> **POC scope:** single account (CDO Prod) acceptable initially.

---

## Sensitive data classes

| Data | Risk | Handling |
|------|------|----------|
| Query text in QUERY_HISTORY | May contain PII literals | Store hash + sanitized AST features; retain raw text only if policy allows, encrypted, no egress |
| Role / user names | HR-sensitive | Need-to-know in UI; access control on API |
| Object names | Usually OK | Still internal |
| Credits / $ | Financial | Standard FinOps ACL |
| Access history | Higher sensitivity | T3 only |

---

## SQL sanitization pipeline

```text
query_text (optional retain)
    → sqlglot parse
    → AST features: tables, predicates present?, select-star?, join count, ...
    → drop literals / replace with typed placeholders
    → fingerprint_features JSON
```

Rules:

- Default **do not** persist full query text in Postgres.  
- If needed for human debug: separate locked store, retention ≤ 30d, no LLM.  
- `QUERY_PARAMETERIZED_HASH` is the join key for shapes.

---

## LLM boundary (narrow, optional)

| Allowed input | Forbidden |
|---------------|-----------|
| Opportunity type, KPI evidence numbers, sanitized features, warehouse size, idle %, spill rate | Raw SQL, table samples, PII, credentials |
| Output: explanation paragraph tagged `ai_assisted` + `requires_human_review` | Autonomous actions |

LLM **off** by default in stack YAML. Product works fully without it.

---

## Application security

- AuthN: SSO / corporate IdP for React + API.  
- AuthZ: FinOps group default; optional per-stack ACL.  
- Audit: who dismissed opportunities; who viewed query text (if ever).  
- Extract workers: network allowlist to Snowflake only.  
- Postgres: encryption at rest per platform standard.

---

## Provenance & reproducibility

Every opportunity and ledger row traces to `run_id` with:

- Extract as-of timestamp  
- Stack config hash  
- Git SHA (when built from CI)  
- Access tier / capability flags  

Supports “why did this number appear on date D?”

---

## Future autonomous mode (out of scope for v1)

If ever added:

- Separate `SNOWLENS_EXECUTOR` role with least MODIFY on **opt-in** warehouses  
- Human approval mandatory  
- Idempotent ALTER + rollback snapshot  
- LLM never holds executor credentials  

Do not design v1 APIs that imply execution.
