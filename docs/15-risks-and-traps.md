# 15 — Risks & Traps

**Purpose:** Consolidate catalog traps with product-level failure modes. Encode as tests, gates, or UI copy.

---

## From the KPI catalog (must encode)

| Trap | Reality | Product response |
|------|---------|------------------|
| `AVG_RUNNING` is utilization % | Load ratio; can exceed 1 | Evidence copy; never drive RIGHTSIZE alone |
| Bytes scanned = cost | Cost is warehouse-seconds × size × clusters | Addressability × uptime in ranking |
| Σ attributed = bill | Excludes idle, short queries, CS, serverless; NULL adaptive | Publish A8; idle as feature |
| Lower auto_suspend always better | 60s minimums + cache loss | B13 simulator; can recommend “keep” |
| 0 reads = delete | Latency, retention, seasonal jobs | Advisory; H13; never delete |
| High cache % is a lever | Symptom | Evidence only |
| Cloud services always waste | Free below 10% compute | A3 gate |
| Fail-safe frees on DROP | 7d permanent | Storage copy |
| Bigger WH ∝ faster | Only some workloads | B14 hierarchy |
| Consolidation always saves | Only if busy windows don’t fully overlap | C3 + C6 |
| Regression on raw credits | Confounded by resize | Size-adjusted seconds |
| Scan growth = regression | May be data growth | Normalize G7 |
| Storage is where money is | Usually minority | Lead compute |
| Tags will exist | Often poor coverage | TAG_COVERAGE_GAP |
| One SHOW snapshot enough | No config history in AU | Snapshot every run |

---

## Product-level traps

### Double-counting savings

**Risk:** Sum of cards on exec slide.  
**Mitigation:** Uptime budget + UI “portfolio vs raw sum” ([05](05-savings-accounting.md)).

### Consolidation vs isolation

**Risk:** Merge the warehouse that needed split.  
**Mitigation:** Arbitration prefers ISOLATE when F2 severe ([05](05-savings-accounting.md)).

### Goodhart on idle share

**Risk:** Teams inflate useless queries to look “utilized.”  
**Mitigation:** Rank on credits and uptime anchors, not idle % alone; E7 detects heartbeats.

### Control contamination (verification)

**Risk:** Controls also resized; DiD biased.  
**Mitigation:** Strict control filters; verified_weak fallback; CI spanning 0 → verified_zero.

### Claiming credit for unrelated changes

**Risk:** Someone resized for performance; SnowLens marks verified.  
**Mitigation:** Match `expected_change` fingerprint; else `observed_unrelated`.

### Survivorship / cold tables

**Risk:** ACCESS_HISTORY window hides quarterly reads.  
**Mitigation:** Probabilistic language; write-recency guard; advisory only.

### Adaptive warehouse blind spot

**Risk:** Idle formula NULL.  
**Mitigation:** capability flag; alternate metering path; don’t fake A8.

### Extract warehouse cost

**Risk:** SnowLens burns credits scanning QUERY_HISTORY.  
**Mitigation:** Push-down agg; XS warehouse; spend floors before gap extracts.

### Security review failure

**Risk:** Asking for ACCOUNTADMIN / raw SQL to OpenAI.  
**Mitigation:** [12](12-security-and-privacy.md); LLM off; probe-first.

### Overbuilding detection vs pricing

**Risk:** Reimplement QUERY_INSIGHTS.  
**Mitigation:** Consume D11; differentiate on price/rank/verify.

### Solo scope creep

**Risk:** CI/CD + autonomy + org multi-account before ledger works.  
**Mitigation:** Roadmap phase exits ([14](14-roadmap.md)).

### Chargeback politics

**Risk:** Consolidation kills finance’s warehouse-name allocation.  
**Mitigation:** Surface as prerequisite; TAG_COVERAGE_GAP; don’t surprise finance.

### Latency promises

**Risk:** “+8% P95” without evidence class.  
**Mitigation:** Only with natural experiment / clear bound; else qualitative risk.

---

## Suggested automated guards

| Guard | Assert |
|-------|--------|
| Accounting | `portfolio_p50 ≤ sum_cards_p50 + ε` |
| Idle identity | \|used − attributed − idle\| small when attributed non-NULL |
| Autosuspend | Optimal T can equal current (no-op allowed) |
| Consolidation | No open card with C6 false |
| Verification | No ledger row without observed_at |
| Privacy | No opportunity evidence blob contains raw `query_text` by default |

---

## Operational risks

| Risk | Mitigation |
|------|------------|
| Account Usage latency → “realtime” expectation | Doc + UI copy |
| DBA delay | Synthetic + T1-minimal detectors |
| Radar contract drift | Version field on import artifacts |
| Postgres growth | Staging TTL; aggregate facts |

---

## Credibility checklist before any external demo

- [ ] A8 shown  
- [ ] Ranges not fake precision  
- [ ] Portfolio ≠ sum explained  
- [ ] Advisory language  
- [ ] Access tier honest  
- [ ] At least one idle + one consolidation story  
- [ ] No LLM required for the narrative  
