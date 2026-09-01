# Production Readiness Engine — Product, Scoring & Implementation Spec

**Version:** 1.0 (design baseline)
**Purpose:** Bridge the gap between vibecoded projects and production-grade enterprise systems by making organisational standards *checkable at promotion time* and *fixable via generated remediation prompts*.

---

## 1. Product Thesis

The organisation does not have a standards problem. It has an **enforcement-point problem**. Standards exist as documents, skills and instructions; they are absent at the two moments that matter:

1. **Generation time** — the coding agent does not know them.
2. **Promotion time** — nothing verifies them before code reaches production.

This product owns moment #2, and produces the artifact that lets a developer retroactively fix moment #1: a **copy-pasteable remediation prompt** scoped to a specific finding.

### Design constraints (non-negotiable)

| Constraint | Rationale |
|---|---|
| **Read-only access to repos** | No write access = no lengthy security review, no blast radius, faster adoption. |
| **Deterministic checks dominate the score** | A score that moves between runs on identical code is dead on arrival. |
| **Every finding carries evidence** | File, line range, snippet. A finding without evidence is an opinion. |
| **Every finding carries a fix prompt** | Scoring without remediation is homework. Nobody does homework. |
| **The rule catalogue is data, not code** | Adding a newly-observed failure mode must be a config edit + review, not a release. |

---

## 2. Core Domain Model

Five first-class entities. Everything else in the product is a view over these.

```
Rule ──────< Finding >────── Scan ──────> Scorecard
  │                            │
  │                            └──> RemediationPrompt(s)
  │
  └──< Waiver
```

### 2.1 Rule (the editable catalogue entry)

The **Rule** is the central primitive. The entire product's extensibility rests on this being a versioned, reviewable data object rather than hardcoded logic.

```yaml
# rules/security/SEC-001.yaml
id: SEC-001
version: 4
title: Hardcoded credential or secret in source
category: SECURITY
subcategory: secrets_management
status: active            # active | draft | deprecated
detection:
  method: HYBRID          # STATIC | LOOKUP | METRIC | LLM | HYBRID
  static:
    engine: regex+entropy
    patterns:
      - id: aws_key
        pattern: 'AKIA[0-9A-Z]{16}'
      - id: generic_assignment
        pattern: '(?i)(password|passwd|secret|api[_-]?key|token)\s*[:=]\s*["''][^"'']{8,}["'']'
    entropy_threshold: 4.2
    file_globs: ["**/*"]
    exclude_globs: ["**/test/fixtures/**", "**/*.example", "**/*.md"]
  llm_adjudication:
    enabled: true
    purpose: "Classify candidate as REAL_SECRET | PLACEHOLDER | TEST_FIXTURE | FALSE_POSITIVE"
    max_candidates: 50
severity:
  default: BLOCKER
  escalation:
    - when: "service_tier in [0,1] and environment == 'prod'"
      to: BLOCKER
    - when: "file is test-only"
      to: MINOR
scoring:
  base_penalty: 10
  occurrence_model: LOG_DAMPED
  max_penalty_per_rule: 30
gate:
  blocking: true          # part of the hard-gate set
  waivable: true
  max_waiver_days: 30
remediation:
  standard_ref: "ENG-SEC-114 §3.2 — Secret Management via Vault"
  target_pattern_ref: "internal.security.vault_client"
  prompt_template: templates/sec-001.j2
  verification: "grep -rE '<pattern>' returns no matches; secret resolves via vault client at runtime"
metadata:
  owner: platform-security
  created: 2026-09-01
  last_reviewed: 2026-09-01
  false_positive_rate: 0.04     # tracked automatically, see §7.3
  rationale: |
    Observed 3 incidents in FY26 where generated code embedded a live
    connection string committed to a shared repo.
```

**Why this shape matters:** severity, penalty, gate behaviour, and the remediation prompt are all *properties of the rule*, editable by a rule owner. When you notice a new recurring failure mode, you author one YAML file and open a PR against the catalogue repo. No product release.

### 2.2 Finding

An instance of a Rule violation in a specific scan.

```json
{
  "finding_id": "f_9a2c...",
  "rule_id": "SEC-001",
  "rule_version": 4,
  "scan_id": "s_4471",
  "severity": "BLOCKER",
  "confidence": "CONFIRMED",
  "evidence": {
    "file": "src/db/connection.py",
    "line_start": 14,
    "line_end": 14,
    "snippet": "conn = psycopg2.connect(password=\"Pr0d!ncr3ds\")",
    "detector": "static:generic_assignment",
    "adjudicator": "llm:REAL_SECRET"
  },
  "status": "OPEN",
  "first_seen_scan": "s_4102",
  "age_days": 12,
  "fingerprint": "sha256(rule_id|normalized_path|normalized_snippet)"
}
```

The **fingerprint** is critical. It is what lets you compute deltas, track age, detect regressions, and prevent a line-number shift from being reported as a new finding.

### 2.3 Scan, Scorecard, Waiver

- **Scan** — one execution against one repo at one commit, in one profile.
- **Scorecard** — the computed output: category scores, overall score, gate verdict, delta vs. baseline.
- **Waiver** — a time-bounded, owned, justified acceptance of a finding or rule for a repo.

---

## 3. Category Model

Eleven categories. Ten are conventional production-readiness domains; the eleventh (**AI-Generated Code Integrity**) is the differentiator and does not exist in off-the-shelf tooling.

### 3.1 Default weight profile — `backend-service`

| # | Category | Code | Weight | Primary detection posture |
|---|---|---|---|---|
| 1 | Security & Secrets | `SEC` | 15 | Static-dominant + LLM adjudication |
| 2 | Reliability & Resilience | `REL` | 13 | Static AST + LLM judgment |
| 3 | Observability & Operability | `OBS` | 12 | Static + config lookup |
| 4 | Testing & Verification | `TST` | 12 | Metric + LLM (test quality) |
| 5 | Dependency & Supply Chain | `DEP` | 10 | Pure lookup |
| 6 | Deployment & Release | `DPL` | 10 | File presence + IaC parse |
| 7 | Architecture & Structure | `ARC` | 6 | Static graph + LLM |
| 8 | Code Quality & Maintainability | `COD` | 6 | Static metrics |
| 9 | Documentation & Ownership | `DOC` | 6 | Lookup + LLM (drift) |
| 10 | Data & Compliance | `DAT` | 5 | Static + lookup |
| 11 | AI-Generated Code Integrity | `AIC` | 5 | Lookup-dominant |
| | **Total** | | **100** | |

### 3.2 Profiles (weights are per-profile, editable)

Different workload shapes need different weightings. Profiles are catalogue objects too.

| Profile | Notable weight shifts | Rules disabled |
|---|---|---|
| `backend-service` | baseline | — |
| `frontend-app` | OBS↓8, REL↓8, SEC↑18 (XSS/CSP), COD↑8 | server-side resilience rules |
| `batch-etl-job` | REL↑16 (idempotency, restartability), OBS↑14, DAT↑10, TST↓9 | health-endpoint, autoscaling rules |
| `internal-tool` | all thresholds relaxed; gate advisory-only | most DPL rules |
| `ml-service` | adds model-artifact provenance, drift monitoring under `DAT` | — |
| `library-package` | DPL↓2, DOC↑12, ARC↑10 | runtime/deployment rules |

---

## 4. The Check Catalogue

Notation for **Method**:

| Code | Meaning | Cost | Determinism |
|---|---|---|---|
| `S` | Static — AST, regex, file presence, config parse | very low | fully deterministic |
| `L` | Lookup — external system of record (registry, catalog, CMDB, schema) | low | deterministic given source |
| `M` | Metric — ingested from CI, coverage, build tooling | low | deterministic |
| `H` | Hybrid — static prefilter, LLM adjudicates candidates | medium | bounded non-determinism |
| `A` | LLM judgment — no reliable static equivalent | high | non-deterministic, advisory |

**Rule of thumb enforced in the catalogue:** `A`-method rules may not be `blocking: true`, and their combined penalty contribution to any category is capped at 25%. This is what keeps the score defensible when a developer disputes a finding.

---

### 4.1 SECURITY & SECRETS (`SEC`) — weight 15

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| SEC-001 | Hardcoded credential / secret / token in source | H | BLOCKER | ✅ |
| SEC-002 | Secret committed in git history (not just HEAD) | S | BLOCKER | ✅ |
| SEC-003 | Secrets in env files, IaC, k8s manifests, CI config | S | BLOCKER | ✅ |
| SEC-004 | No approved secret-manager client in use | L | CRITICAL | ✅ |
| SEC-005 | SQL/NoSQL injection — string-concatenated queries | H | CRITICAL | ✅ |
| SEC-006 | Command injection — shell exec with interpolated input | H | CRITICAL | ✅ |
| SEC-007 | Unsafe deserialisation (`pickle`, `yaml.load`, `eval`) | S | CRITICAL | ✅ |
| SEC-008 | Path traversal on user-controlled file paths | H | MAJOR | — |
| SEC-009 | Missing authentication on exposed route/endpoint | H | BLOCKER | ✅ |
| SEC-010 | Missing authorisation check (authn present, authz absent) | A | CRITICAL | — |
| SEC-011 | TLS verification disabled (`verify=False`, `rejectUnauthorized:false`) | S | CRITICAL | ✅ |
| SEC-012 | Wildcard CORS with credentials enabled | S | MAJOR | — |
| SEC-013 | Missing security headers / CSP (web-facing) | S | MAJOR | — |
| SEC-014 | Weak or deprecated crypto (MD5, SHA1, DES, ECB, static IV) | S | MAJOR | — |
| SEC-015 | No input validation at trust boundary | A | MAJOR | — |
| SEC-016 | PII / secrets written to logs | H | CRITICAL | ✅ |
| SEC-017 | Container runs as root / privileged / writable rootfs | S | MAJOR | — |
| SEC-018 | Over-broad IAM in IaC (`Action:*`, `Resource:*`) | S | CRITICAL | ✅ |
| SEC-019 | Debug mode / verbose stack traces enabled for prod | S | MAJOR | — |
| SEC-020 | Rate limiting absent on public-facing endpoint | S+A | MAJOR | — |

**Implementation notes.** SEC-001/003/016 are the false-positive battleground. The pattern is: aggressive static regex + entropy scan to generate candidates → cap candidates at N → single batched LLM call classifies each as `REAL_SECRET | PLACEHOLDER | TEST_FIXTURE | FALSE_POSITIVE`. Only `REAL_SECRET` scores; the rest are reported as informational. SEC-002 requires a git-history walk, which is expensive — run it on first scan and on a weekly cadence, not every commit.

SEC-009 is high-value and underrated: enumerate route decorators/handlers via AST, cross-reference against auth middleware registration, flag routes with no auth in the chain. Mostly static; LLM only resolves ambiguous middleware composition.

---

### 4.2 RELIABILITY & RESILIENCE (`REL`) — weight 13

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| REL-001 | Outbound HTTP/RPC call with no timeout | S | CRITICAL | ✅ |
| REL-002 | No retry policy on transient-failure-prone call | S | MAJOR | — |
| REL-003 | Retry without exponential backoff / jitter | S | MAJOR | — |
| REL-004 | No circuit breaker on critical downstream dependency | S+A | MAJOR | — |
| REL-005 | Swallowed exception (`except: pass`, empty catch) | S | MAJOR | — |
| REL-006 | Overly broad exception handler masking real failures | S | MINOR | — |
| REL-007 | Unbounded resource use (no pagination, no result limit) | H | MAJOR | — |
| REL-008 | Connection pool not configured / unbounded | S | MAJOR | — |
| REL-009 | Resource leak — unclosed file, connection, client | S | MAJOR | — |
| REL-010 | No graceful shutdown / SIGTERM handling | S | MAJOR | — |
| REL-011 | Non-idempotent operation on retryable path | A | MAJOR | — |
| REL-012 | Blocking I/O inside async context | S | MAJOR | — |
| REL-013 | No dead-letter queue / poison-message handling | S+A | MAJOR | — |
| REL-014 | Missing transaction boundary on multi-step write | A | CRITICAL | — |
| REL-015 | Race condition — unguarded shared mutable state | A | MAJOR | — |
| REL-016 | Hardcoded `sleep()` used as synchronisation | S | MINOR | — |
| REL-017 | No batch restartability / checkpointing (ETL profile) | S+A | CRITICAL | ✅* |

`*` blocking only under `batch-etl-job` profile.

**Implementation notes.** REL-001 is the single highest-ROI static check in the whole product and is trivially deterministic: AST-match known HTTP/DB client call sites, inspect kwargs/config for a timeout parameter, resolve to a default-configured session object if one exists. Do this one first.

REL-011, 014, 015 have no reliable static form and are LLM-only — keep them advisory, present them under a distinct "Model Judgment" section in the UI, and never let them block.

---

### 4.3 OBSERVABILITY & OPERABILITY (`OBS`) — weight 12

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| OBS-001 | No structured logging framework (print/console.log in prod paths) | S | CRITICAL | ✅ |
| OBS-002 | Logs lack correlation / trace / request ID | S | MAJOR | — |
| OBS-003 | No liveness endpoint | S | CRITICAL | ✅ |
| OBS-004 | No readiness endpoint (or readiness == liveness) | S | MAJOR | — |
| OBS-005 | No metrics emission (RED/USE not covered) | S | MAJOR | — |
| OBS-006 | No distributed tracing instrumentation | S | MAJOR | — |
| OBS-007 | Error paths unlogged — silent failure | S | MAJOR | — |
| OBS-008 | Log levels misused (errors at INFO, noise at ERROR) | H | MINOR | — |
| OBS-009 | No alert definitions / SLO defined for the service | L | CRITICAL | ✅ |
| OBS-010 | No dashboard registered for the service | L | MAJOR | — |
| OBS-011 | Log volume risk — logging inside hot loop | S | MINOR | — |
| OBS-012 | No runbook linked for defined alerts | L | MAJOR | — |
| OBS-013 | Non-standard log schema (violates org logging contract) | S | MAJOR | — |

**Implementation notes.** OBS-009/010/012 are pure lookups against your monitoring and alerting platforms — no code analysis at all. They're cheap, they're unambiguous, and they catch the classic vibecoded service that has beautiful code and zero operational presence. Prioritise the integrations.

---

### 4.4 TESTING & VERIFICATION (`TST`) — weight 12

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| TST-001 | No test suite present | S | BLOCKER | ✅ |
| TST-002 | Line coverage below profile threshold | M | MAJOR | — |
| TST-003 | Critical-path / changed-lines coverage below threshold | M | CRITICAL | ✅ |
| TST-004 | **Assertion-free tests** (test executes, asserts nothing) | S | CRITICAL | ✅ |
| TST-005 | **Tautological assertions** (`assert True`, `assert x == x`) | S | MAJOR | — |
| TST-006 | **Over-mocking** — the unit under test is itself mocked | H | MAJOR | — |
| TST-007 | Coverage concentrated on trivial code (getters, DTOs) | M+A | MAJOR | — |
| TST-008 | No error-path or negative-case tests | A | MAJOR | — |
| TST-009 | Skipped / disabled / commented-out tests | S | MAJOR | — |
| TST-010 | Flaky tests (historical pass-rate variance) | M | MAJOR | — |
| TST-011 | Tests depend on live external systems | H | MAJOR | — |
| TST-012 | No integration or contract tests for external interfaces | S+L | MAJOR | — |
| TST-013 | Tests not wired into CI | L | BLOCKER | ✅ |
| TST-014 | No load / performance test for Tier-0/1 service | L | MAJOR | — |

**Implementation notes.** TST-004/005/006 form the **"test theater" detector** and are your credibility feature for this category. Coverage percentage alone is actively misleading on generated code — LLMs produce high-coverage, low-assertion test suites as a matter of course. All three are statically detectable: parse each test function's AST, count assertion nodes, check whether the mocked symbol matches the imported symbol under test. Deterministic, fast, and it lands hard in demos.

TST-003 (coverage of *changed lines*) should carry more weight than TST-002 (absolute coverage). It's the metric that lets legacy repos improve incrementally instead of being permanently red.

---

### 4.5 DEPENDENCY & SUPPLY CHAIN (`DEP`) — weight 10

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| DEP-001 | **Phantom package** — does not exist in internal registry or public index | L | BLOCKER | ✅ |
| DEP-002 | Unapproved package (exists, not on org allowlist) | L | CRITICAL | ✅ |
| DEP-003 | Known CVE — critical/high severity | L | BLOCKER | ✅ |
| DEP-004 | Known CVE — medium severity | L | MAJOR | — |
| DEP-005 | Unpinned / floating version specifier | S | MAJOR | — |
| DEP-006 | No lockfile committed | S | CRITICAL | ✅ |
| DEP-007 | Abandoned dependency (no release in N months) | L | MAJOR | — |
| DEP-008 | Typosquat / slopsquat candidate — near-match to popular package | L | BLOCKER | ✅ |
| DEP-009 | License incompatible with org policy | L | CRITICAL | ✅ |
| DEP-010 | Dependency not resolvable from internal artifact registry | L | CRITICAL | ✅ |
| DEP-011 | Deprecated internal library still in use | L | MAJOR | — |
| DEP-012 | No SBOM generated | S | MAJOR | — |
| DEP-013 | Direct dependency count anomalously high for project size | M | MINOR | — |

**Implementation notes.** This entire category is **pure lookup — zero LLM**. Parse the manifest, hit your registry, hit the vulnerability database, compare against allowlists. It is the cheapest, fastest, most deterministic category in the product and it catches the most dangerous class of AI failure.

DEP-001 and DEP-008 together are the **hallucinated-dependency detector**. Generated code routinely imports plausible-sounding packages that don't exist, or that an attacker has registered *because* models hallucinate that name. Implementation for DEP-008: for each dependency not on the allowlist, compute Levenshtein distance against the top-N package names in that ecosystem; distance of 1–2 combined with low download count is a strong signal. This check alone justifies the product to a security audience.

---

### 4.6 DEPLOYMENT & RELEASE (`DPL`) — weight 10

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| DPL-001 | No CI pipeline defined | L | BLOCKER | ✅ |
| DPL-002 | No infrastructure-as-code — environment provisioned manually | S+L | CRITICAL | ✅ |
| DPL-003 | **No documented rollback path** | S+L | BLOCKER | ✅ |
| DPL-004 | No containerisation / build artifact definition | S | CRITICAL | — |
| DPL-005 | Container image not built from approved base image | S+L | MAJOR | — |
| DPL-006 | No resource requests / limits declared | S | MAJOR | — |
| DPL-007 | Config not externalised — env-specific values in code | S | CRITICAL | ✅ |
| DPL-008 | **Machine-local artifacts** — absolute paths, `localhost`, hardcoded ports | S | CRITICAL | ✅ |
| DPL-009 | Single replica / no HA for Tier-0/1 service | S | CRITICAL | — |
| DPL-010 | No database migration strategy | S+A | MAJOR | — |
| DPL-011 | Breaking schema change without backward-compatible path | A | CRITICAL | — |
| DPL-012 | No environment parity (dev/stage/prod configs diverge structurally) | S | MAJOR | — |
| DPL-013 | Build not reproducible (`latest` tags, unpinned build deps) | S | MAJOR | — |
| DPL-014 | No artifact signing / provenance attestation | L | MAJOR | — |
| DPL-015 | Deployment lacks health-gated rollout | S | MAJOR | — |

**Implementation notes.** DPL-008 is the "works on my machine" detector and is the most viscerally recognisable finding in the entire product — a regex for absolute filesystem paths, `localhost`/`127.0.0.1`, and hardcoded ports outside config files. Trivial to build, universally understood, great demo material.

---

### 4.7 ARCHITECTURE & STRUCTURE (`ARC`) — weight 6

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| ARC-001 | Layering violation (e.g. DB access from controller) | S | MAJOR | — |
| ARC-002 | Circular dependency between modules | S | MAJOR | — |
| ARC-003 | God module / god class — excessive responsibility concentration | M | MAJOR | — |
| ARC-004 | Business logic embedded in framework/entry-point code | A | MAJOR | — |
| ARC-005 | Duplicated logic across modules (clone detection) | S | MINOR | — |
| ARC-006 | Non-conformance to org reference architecture for profile | A | MAJOR | — |
| ARC-007 | Synchronous call chain depth exceeds threshold | S | MAJOR | — |
| ARC-008 | Shared mutable global state | S | MAJOR | — |
| ARC-009 | Missing abstraction over external vendor SDK | A | MINOR | — |
| ARC-010 | Inconsistent internal API contract style | S+A | MINOR | — |

---

### 4.8 CODE QUALITY & MAINTAINABILITY (`COD`) — weight 6

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| COD-001 | Cyclomatic complexity above threshold | M | MAJOR | — |
| COD-002 | Function / file length above threshold | M | MINOR | — |
| COD-003 | Dead / unreachable code | S | MINOR | — |
| COD-004 | Commented-out code blocks | S | MINOR | — |
| COD-005 | `TODO` / `FIXME` / `HACK` markers on production paths | S | MINOR | — |
| COD-006 | Magic numbers / unexplained literals | S | MINOR | — |
| COD-007 | Linter / formatter not configured or failing | L | MAJOR | — |
| COD-008 | Type annotations absent (typed-language ecosystems) | S | MAJOR | — |
| COD-009 | Inconsistent naming conventions | S | MINOR | — |
| COD-010 | Duplicate-block ratio above threshold | M | MINOR | — |
| COD-011 | Excessive nesting depth | M | MINOR | — |

**Implementation note.** Do not rebuild what exists. This category should be a thin adapter over your existing SAST/lint/complexity tooling, normalising its output into the Finding schema. Building your own complexity analyser is wasted effort.

---

### 4.9 DOCUMENTATION & OWNERSHIP (`DOC`) — weight 6

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| DOC-001 | **No registered owning team** | L | BLOCKER | ✅ |
| DOC-002 | No on-call rotation mapped to the service | L | CRITICAL | ✅ |
| DOC-003 | No README, or README is scaffold-default | S | MAJOR | — |
| DOC-004 | **Doc/code drift** — README describes behaviour code lacks | A | CRITICAL | — |
| DOC-005 | No architecture / design doc | S+L | MAJOR | — |
| DOC-006 | No runbook for common failure modes | L | CRITICAL | ✅ |
| DOC-007 | API contract missing or not published to catalog | S+L | MAJOR | — |
| DOC-008 | Setup instructions incomplete or non-reproducible | A | MAJOR | — |
| DOC-009 | No documented dependencies / downstream consumers | A | MINOR | — |
| DOC-010 | Undocumented configuration parameters | S+A | MINOR | — |
| DOC-011 | No changelog / release notes | S | MINOR | — |

**Implementation note.** DOC-004 is the flagship LLM check and the most persuasive one in the category — confidently wrong documentation is a hallmark of generated repos. Implementation: extract claims from the README (endpoints, env vars, commands, features), then verify each claim against the code index. Each unverifiable claim is a sub-finding with the specific sentence quoted. Structure it as claim-by-claim verification rather than "read the README and judge it" — that's what makes the output specific enough to act on and cheap enough to run.

---

### 4.10 DATA & COMPLIANCE (`DAT`) — weight 5

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| DAT-001 | PII handled without classification / tagging | H | CRITICAL | ✅ |
| DAT-002 | No encryption at rest configured for data store | S+L | CRITICAL | ✅ |
| DAT-003 | No encryption in transit | S | CRITICAL | ✅ |
| DAT-004 | No data retention / purge policy | L | MAJOR | — |
| DAT-005 | Cross-region data movement without approval | L | CRITICAL | — |
| DAT-006 | No audit trail for sensitive data access | A | MAJOR | — |
| DAT-007 | Test data contains production-derived records | H | CRITICAL | ✅ |
| DAT-008 | No backup / recovery configuration | L | CRITICAL | — |
| DAT-009 | Schema change without data-quality validation | S+A | MAJOR | — |
| DAT-010 | Regulated data in non-approved store | L | BLOCKER | ✅ |

---

### 4.11 AI-GENERATED CODE INTEGRITY (`AIC`) — weight 5

This category is the product's differentiator. Every check targets a failure mode specific to how LLMs produce code, not to bad code generally. Low weight, high severity — these are rare but almost always blockers.

| ID | Check | Method | Default severity | Blocking |
|---|---|---|---|---|
| AIC-001 | **Phantom internal API** — call to an endpoint absent from the service catalog | L | BLOCKER | ✅ |
| AIC-002 | **Phantom schema object** — reference to non-existent table/column/field | L | BLOCKER | ✅ |
| AIC-003 | **Phantom config key** — reads env var never defined anywhere | S | CRITICAL | ✅ |
| AIC-004 | Non-existent method on an internal library | L | CRITICAL | ✅ |
| AIC-005 | Deprecated API version used where a current one exists | L | MAJOR | — |
| AIC-006 | Plausible-but-wrong internal convention (naming, auth flow, error envelope) | A | MAJOR | — |
| AIC-007 | Scaffold residue — placeholder values, `TODO: implement`, stub returns | S | CRITICAL | ✅ |
| AIC-008 | Copy-paste variance — same logic implemented inconsistently across files | S+A | MINOR | — |
| AIC-009 | Unused generated abstraction (interfaces/classes with one or zero uses) | S | MINOR | — |
| AIC-010 | Comment describes behaviour the code does not implement | A | MAJOR | — |
| AIC-011 | Mixed idioms suggesting uncoordinated generation sessions | A | MINOR | — |

**Implementation notes.** AIC-001 through AIC-004 are the crown jewels and they are **pure lookup, not LLM**. You extract the symbol (URL path, table name, method call, env var) statically, then check existence against a system of record — service catalog, information schema, library type stubs, config manifests. If the org lacks these registries, this category degrades to advisory. Getting read access to the service catalog and database information schemas is therefore a **prerequisite dependency**, and worth chasing early because nothing else in the market does this.

AIC-007 (scaffold residue) is trivially static and catches genuinely dangerous cases — a `return True` stub where an authorisation check was supposed to be.

---

## 5. Scoring Logic

### 5.1 Principles

1. Penalty-based, not credit-based. Start at 100, subtract. Nobody earns points for a README.
2. Damped occurrence. Ten instances of one rule must not zero a category — but must cost more than one.
3. Confidence-weighted. LLM findings contribute less than deterministic ones.
4. Capped LLM contribution. Model judgment cannot dominate any category.
5. Two numbers, not one. A tier-normalised **Readiness Score** for the repo, an absolute **Risk Unit** figure for fleet rollup.

### 5.2 Severity weights

| Severity | Base penalty `w` | Meaning |
|---|---|---|
| BLOCKER | 10 | Would cause an incident or breach. Cannot go to prod. |
| CRITICAL | 7 | Serious operational or security gap. |
| MAJOR | 4 | Real risk, degrades operability or maintainability. |
| MINOR | 2 | Hygiene. Cumulative effect only. |
| INFO | 0 | Reported, never scored. |

### 5.3 Confidence multipliers

| Confidence | Multiplier `c` | Source |
|---|---|---|
| CONFIRMED | 1.00 | Deterministic static / lookup / metric |
| HIGH | 0.85 | Static prefilter + LLM adjudication agreed |
| MEDIUM | 0.60 | LLM judgment, corroborating signal present |
| LOW | 0.30 | LLM judgment alone — advisory, displayed separately |

### 5.4 Per-rule penalty with occurrence damping

For rule $r$ with $n_r$ occurrences:

$$P_r = \min\left( w_r \cdot c_r \cdot \left(1 + \ln(n_r)\right) \cdot m_r,\; P^{max}_r \right)$$

Where:
- $w_r$ — severity base penalty (post-escalation)
- $c_r$ — confidence multiplier
- $n_r$ — occurrence count of that rule in this scan
- $m_r$ — tier escalation multiplier (§5.6)
- $P^{max}_r$ — per-rule cap from the rule definition

The $\ln$ damping means: 1 occurrence = 1.00×, 3 = 2.10×, 10 = 3.30×, 50 = 4.91×. Volume matters, but sublinearly.

### 5.5 Category and overall score

$$S_{cat} = \max\left(0,\; 100 - \sum_{r \in cat} P_r\right)$$

With the LLM-contribution cap applied before summation:

$$\sum_{r \in cat,\, method = A} P_r \;\le\; 0.25 \cdot \sum_{r \in cat} P_r$$

Overall readiness score, over active categories only (a category with no applicable rules is dropped and weights renormalised):

$$S_{overall} = \frac{\sum_{cat} W_{cat} \cdot S_{cat}}{\sum_{cat} W_{cat}}$$

### 5.6 Tier escalation

Service criticality tier comes from the CMDB/service catalog, not from the repo.

| Tier | Description | Multiplier $m$ | Gate threshold | Blocker tolerance |
|---|---|---|---|---|
| 0 | Customer-facing, revenue-critical | 1.5 | 85 | 0 |
| 1 | Business-critical internal | 1.3 | 75 | 0 |
| 2 | Standard production service | 1.0 | 65 | 0 |
| 3 | Internal tool, low blast radius | 0.7 | 50 | advisory |
| 4 | Experimental / sandbox | 0.4 | — | advisory only |

Some rules additionally define **severity escalation by tier** (a missing rollback path is MAJOR on Tier-3, BLOCKER on Tier-0). This is a rule-level property, not a global one.

### 5.7 Risk Units — the fleet-level currency

The Readiness Score is tier-normalised, which makes it good for comparing a repo to its own past and bad for aggregating across a fleet. So compute a second, absolute figure:

$$RU_{repo} = \sum_{r} w_r \cdot c_r \cdot n_r^{0.5} \cdot BlastRadius_{repo}$$

Where $BlastRadius$ is derived from tier, active user count, and downstream dependent count. Risk Units are additive across the fleet, which makes them the right input for "what is our aggregate exposure" and for prioritising remediation effort org-wide.

### 5.8 Grade bands

| Score | Grade | Label | Meaning |
|---|---|---|---|
| 90–100 | A | Production Ready | Ship it. |
| 75–89 | B | Minor Gaps | Ship with tracked follow-ups. |
| 60–74 | C | Needs Work | Conditional — remediate blockers first. |
| 40–59 | D | Not Ready | Significant gaps across categories. |
| 0–39 | F | Prototype | Not a production candidate. |

### 5.9 Gate verdict logic

The verdict is **not** a function of the score alone. Two independent conditions:

```
verdict = PASS
if any(finding.severity == BLOCKER and rule.blocking and not waived):
    verdict = FAIL_BLOCKER
elif overall_score < tier_threshold:
    verdict = FAIL_THRESHOLD
elif new_findings_since_baseline(severity >= MAJOR) > 0:
    verdict = FAIL_REGRESSION
elif score_delta < 0:
    verdict = WARN_DECLINE
```

**Rollout sequencing for the gate — do not skip this.** Enforce `FAIL_REGRESSION` before you ever enforce `FAIL_THRESHOLD`. Legacy and existing vibecoded repos will score in the 30s on first scan; blocking them on absolute score means the tool is an obstacle from day one, and that reputation does not wash off. "Don't make it worse" is enforceable immediately and is politically free.

### 5.10 Baseline and delta

Every repo gets a **baseline** at first scan. Reported thereafter:

- Absolute score and grade
- Delta vs. previous scan and vs. baseline
- **New** findings (fingerprint not in previous scan)
- **Fixed** findings (fingerprint present previously, absent now)
- **Regressed** findings (previously fixed, now returned)
- **Aging** findings (open > 30 / 60 / 90 days, by severity)

---

## 6. Remediation Prompt Generation

The output artifact that makes the product used rather than merely run.

### 6.1 Anatomy of a good remediation prompt

Every generated prompt must contain, in this order:

1. **Task statement** — one finding or one tight cluster of the same rule. Never "fix all 47 issues."
2. **Exact locations** — file paths with line ranges. The agent must not hunt.
3. **Current code** — the offending snippet(s) inline as context.
4. **The violated standard** — quoted or referenced from the org standards corpus. *This is the piece no off-the-shelf tool can produce, and it is the product's moat.*
5. **The target pattern** — the approved internal client/wrapper/middleware, with its exact import path and a usage example.
6. **Explicit non-goals** — "do not refactor surrounding code, do not rename, do not upgrade unrelated dependencies, do not modify tests other than X."
7. **Acceptance criteria** — checkable by a human in under a minute.
8. **Verification command** — the test or command that proves the fix.

Item 6 is load-bearing. Without it, the coding agent produces a 400-line diff and you have reproduced the original problem inside your own remediation tool.

### 6.2 Bundling modes

| Mode | Contents | Use case |
|---|---|---|
| Single finding | One finding, full context | Precise, surgical fix |
| Rule cluster | All occurrences of one rule | "Add timeouts to all 14 HTTP calls" — same fix, repeated |
| Category pack | Top N findings in one category | Focused remediation session |
| **Blocker pack** | Every gate-blocking finding | The default. "What do I need to pass?" |
| Quick wins | Highest score-gain per effort | Momentum for a demoralised team |

### 6.3 Generation approach

Templates per rule (Jinja2), populated from the finding's evidence plus retrieved standards context. Deterministic assembly for structure; a single LLM pass only to naturalise phrasing and merge multi-occurrence context. Templates live in the rule catalogue alongside the rule — same PR, same review.

Cache aggressively. The same rule against the same pattern produces near-identical prompts; there is no reason to pay for generation every time.

### 6.4 Closing the loop

When a user copies a prompt, record it. On the next scan of that repo, resolve the outcome:

| Outcome | Definition |
|---|---|
| FIXED | Fingerprint gone, no new findings introduced |
| PARTIAL | Occurrence count reduced, not zero |
| UNCHANGED | Fingerprint still present |
| COLLATERAL | Fixed, but ≥1 new finding appeared in the touched files |
| REGRESSED | Fixed previously, has returned |

**COLLATERAL is the metric that keeps the product honest.** If your remediation prompts introduce new findings, your prompts are bad and you need to know that before your users tell you.

---

## 7. Feature & Service Inventory

### 7.1 Core services

| Service | Responsibility |
|---|---|
| **Ingestion** | Clone/fetch at commit, language detect, profile assign, build file index |
| **Rule Catalogue** | Versioned rule store, profile definitions, validation, publishing |
| **Static Analysis** | AST/regex/config engines per language; adapters over existing SAST/lint |
| **Lookup Connectors** | Registry, CVE DB, service catalog, schema, CMDB, monitoring, CI |
| **LLM Adjudication** | Batched, bounded, cached model calls; candidate classification & judgment |
| **Scoring** | Penalty math, tier escalation, delta computation, gate verdict |
| **Remediation** | Prompt assembly, bundling, caching, outcome tracking |
| **Waiver** | Request, approve, expire, resurface |
| **Reporting** | Scorecards, fleet rollup, trends, exports |
| **Integration** | CI plugin, CLI, PR annotation, webhooks, chat notification |

### 7.2 User-facing features

**Repo scorecard** — overall grade, per-category radar, blocker list, delta since baseline, evidence-linked findings, one-click prompt copy.

**Fleet dashboard** — the leadership view. Distribution of grades, aggregate Risk Units, count of services that would fail today, trend, worst-offender and most-improved lists, category heatmap across the org.

**Rule catalogue admin** — browse, author, version, and test rules; per-rule false-positive rate and trigger frequency; dry-run a new rule against the historical corpus *before* activating it (see §7.3).

**Waiver console** — request with justification and owner, approval workflow, expiry calendar, audit trail.

**CI integration** — scan on PR, annotate changed lines, post scorecard delta as a comment, expose the gate verdict as a status check.

**CLI** — local scan before pushing. Same engine, same rules, no surprises at the gate.

### 7.3 Rule catalogue governance — the part that determines whether this survives

The editable catalogue is the feature you asked for and the thing most likely to rot. Governance requirements:

| Requirement | Detail |
|---|---|
| **Rules are versioned** | Findings record `rule_version`. Score history stays explicable. |
| **Rules are PR-reviewed** | Catalogue is a git repo. Two approvals for blocking rules. |
| **Dry-run before activation** | New rules run in `shadow` status against the last N scans across the fleet. You see trigger volume and sample findings before it affects a single score. **Mandatory for any `blocking: true` rule.** |
| **Auto-tracked false-positive rate** | Every finding is dismissible with a reason. FP rate per rule is computed and displayed. Rules crossing an FP threshold auto-demote from blocking to advisory and notify the owner. |
| **Rules have owners and review dates** | Unreviewed for 12 months → flagged stale. |
| **Score-impact preview** | Before activating, show the fleet-wide score shift the rule would cause. |
| **Incident-to-rule pathway** | A documented flow: post-incident, an action item is "author rule X." This is how the catalogue captures the failure modes you actually experience, which is the whole point. |

---

## 8. KPIs

### 8.1 Repo-level

| KPI | Definition |
|---|---|
| Readiness Score | 0–100, tier-normalised |
| Grade | A–F band |
| Gate verdict | PASS / FAIL_BLOCKER / FAIL_THRESHOLD / FAIL_REGRESSION / WARN_DECLINE |
| Blocker count | Open, unwaived, gate-blocking findings |
| Risk Units | Absolute weighted exposure |
| Score delta | vs. previous scan and vs. baseline |
| Finding age P50 / P90 | Days open, by severity |
| Waiver load | Active waivers; count expiring in 14 days |
| Fix velocity | Findings closed per week |
| Category floor | Lowest-scoring category — usually the real story |

### 8.2 Fleet-level (the leadership view)

| KPI | Why it matters |
|---|---|
| **Services that would fail production readiness today** | The headline number. Everything else is a drill-down. |
| Aggregate Risk Units | Total org exposure, additive and trendable |
| Grade distribution | Shape of the fleet |
| Blockers by category | Where to invest platform effort |
| Mean time to remediate, by severity | Is the org actually closing things |
| Regression rate | Fixed findings that came back |
| Baseline-adjusted improvement | Fleet movement since programme start |
| Coverage | % of production services ever scanned — your adoption metric |
| Top 10 recurring rules | Direct input into the paved-road roadmap |
| Tier-0/1 compliance rate | The number that gets escalated |

### 8.3 Product health (internal — do not put these on the leadership dashboard)

| KPI | Target posture |
|---|---|
| False-positive rate, per rule and overall | The single most important number in the product |
| Dismissal rate | High dismissals = bad rules |
| **Prompt fix-acceptance rate** | % of copied prompts resulting in FIXED |
| **Collateral rate** | % of remediations introducing new findings — must stay near zero |
| Time-to-remediate after prompt copy | Proves the prompt is the value, not the score |
| Scan latency P95 | Gate viability; target under 5 minutes for incremental |
| LLM cost per scan | Determines whether you can scan on every PR |
| Rule catalogue churn | Healthy = steady additions from incidents |

**If you feature one metric externally, feature "N services would fail production readiness today, here is the aggregate risk, here is what it would take to fix."** Not average score. Average score invites arguing about the average.

---

## 9. Implementation Approach

### 9.1 Detection method allocation

| Method | Share of rules | Share of score weight | Notes |
|---|---|---|---|
| Static | ~50% | ~50% | The backbone. Build depth here first. |
| Lookup | ~25% | ~28% | Cheapest, most defensible, best differentiation |
| Metric | ~8% | ~7% | Adapters over existing tooling |
| Hybrid | ~9% | ~10% | Static recall + LLM precision |
| LLM-only | ~8% | ~5% | Advisory, capped, never blocking |

The deliberate design point: **roughly 85% of the score is deterministic.** Someone can dispute a finding and you can show them the rule, the pattern, and the line. That is what makes a gate survive contact with an annoyed senior engineer.

### 9.2 Where LLM is genuinely required

Use the model only where no reliable static equivalent exists:

- **Adjudication** — is this candidate a real secret or a test fixture? (precision on a static-generated candidate set)
- **Semantic drift** — does the README's claim match the code? does the comment match the function?
- **Intent judgment** — is this error handling meaningful or performative? is this operation idempotent?
- **Convention conformance** — does this follow our internal pattern, where "our pattern" is prose in a standards doc?
- **Prompt naturalisation** — assembling the remediation text.

Never use it for: counting, presence checks, version comparison, existence lookups, or anything a registry can answer. Those must be deterministic.

### 9.3 Hybrid pattern (the workhorse)

```
1. Static pass          → high-recall candidate set, deliberately over-inclusive
2. Cheap filters        → path exclusions, entropy, known-safe allowlists
3. Cap                  → max N candidates per rule per scan
4. Batched LLM call     → structured output, one classification per candidate
5. Confidence assignment→ agreement raises confidence; disagreement → advisory
6. Cache                → keyed on content hash; unchanged code is never re-adjudicated
```

Structured output only — schema-constrained JSON, one enum classification plus a one-line rationale per candidate. Do not ask the model for prose findings; you cannot fingerprint prose, and without fingerprints you have no deltas.

### 9.4 Cost and latency control

- **Incremental scans by default.** Full scan on baseline and weekly; PR scans analyse changed files plus their dependency closure.
- **Content-hash caching** at file level for all LLM adjudication.
- **Two-tier models** — small/fast for classification and adjudication, larger only for drift analysis and prompt generation.
- **Per-scan LLM budget** with graceful degradation: exceed it and remaining LLM rules are skipped and marked `NOT_EVALUATED` rather than silently passing. Never let budget exhaustion look like a clean bill of health.

### 9.5 Language support sequencing

Start with the two ecosystems your vibecoded projects actually use. Static depth in two languages beats shallow coverage of eight. Language-agnostic rules (secrets, dependencies, IaC, deployment, docs, ownership, catalog lookups) work everywhere from day one — which means a new language gets partial coverage immediately and deepens over time.

### 9.6 Prerequisite integrations, in priority order

1. Source control (read) — non-negotiable
2. Artifact registry + vulnerability database — unlocks all of `DEP`
3. CI system — gate hook and `TST`/`DPL` signals
4. Service catalog / CMDB — unlocks tier, ownership, and most of `AIC`
5. Monitoring + alerting — unlocks `OBS` lookups
6. Standards corpus (indexed) — required for the remediation moat
7. Database information schema — unlocks AIC-002

Items 4 and 6 are what make this product yours rather than a repackaged SAST tool. Chase them early even if they're politically slower.

---

## 10. Build Sequence

| Phase | Scope | Gate posture |
|---|---|---|
| **P0 — Deterministic core** | SEC (static), DEP (full), DPL (file/config), OBS (presence), TST (theater detection), DOC (ownership lookup). Score + scorecard. CLI. | Advisory everywhere |
| **P1 — Remediation** | Prompt templates for all P0 rules, blocker packs, copy tracking, outcome resolution on re-scan | Advisory |
| **P2 — Catalogue & governance** | Rule authoring UI, versioning, shadow mode, FP tracking, waivers with expiry | Advisory |
| **P3 — Gate** | CI integration, PR annotation, status check. Enforce `FAIL_REGRESSION` only. | Regression-blocking |
| **P4 — Intelligence** | Hybrid adjudication, doc drift, AIC catalog lookups, judgment rules | Add 3–4 hard blockers once FP rate is proven low |
| **P5 — Fleet** | Org dashboard, Risk Units rollup, trends, leadership reporting | Threshold-blocking for Tier-0/1 |
| **P6 — Shift left** | Feed the catalogue back into generation: repo-level agent instruction files, standards served to the coding agent | Prevention |

Phase 6 is where the defect rate actually drops. Everything before it is detection — necessary to earn the right to build P6, and necessary to prove the prevention worked.

---

## 11. Failure Modes to Design Against

| Risk | Mitigation |
|---|---|
| False positives destroy trust | Deterministic dominance, confidence labels, per-rule FP tracking, auto-demotion, one-click dismiss with reason |
| Advisory tool gets ignored | One real gate — start with regression-blocking, which nobody can reasonably argue against |
| Blocking too early creates an obstacle reputation | Advisory → regression-gate → threshold-gate, staged over phases |
| Score gaming | Deltas and Risk Units alongside score; trivial-code coverage detection; waiver audit trail |
| Catalogue rot | Owners, review dates, staleness flags, incident-to-rule pathway |
| Remediation prompts cause damage | Explicit non-goals in every prompt, single-concern scoping, collateral-rate tracking |
| Scan too slow for CI | Incremental by default, aggressive caching, LLM budget caps |
| Demoralised teams on legacy repos | Baseline + delta framing, quick-wins bundle, "no new findings" as the first bar |
| Missing registries degrade `AIC` | Graceful degradation to advisory; make the registry gap itself visible as a platform finding |

---

## 12. Open Decisions

1. **Where is the single chokepoint?** If there is no universal promotion gate, the CI plugin becomes N integrations and the sequencing changes materially.
2. **Which coding assistant is standard?** Determines whether Phase 6 context injection is available at all, and in what format.
3. **Does a service catalog with tier and ownership exist?** If not, `DOC-001`, `DOC-002`, tier escalation, and most of `AIC` degrade to advisory — and building a minimal registry may need to be in scope.
4. **Who owns the rule catalogue?** Platform engineering, security, or a federated model with per-category owners. This is an organisational decision that determines whether the catalogue stays alive.
5. **Is the standards corpus machine-readable?** The remediation moat depends on retrieving the specific clause being violated, not gesturing at a wiki.
