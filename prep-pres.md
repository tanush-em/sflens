# SnowLens Screen and KPI Presentation Guide

**Purpose:** A practical guide for explaining the working SnowLens product to a senior stakeholder. It covers every live UI screen, what a user can do there, every KPI visibly shown, how each metric is measured, its Snowflake source, and why the number matters to the business.

**One-sentence description:** SnowLens is an advisory Snowflake FinOps application that finds avoidable warehouse, query, and storage cost, presents safe recommendations, and later verifies whether people actually saved money.

---

## The story to tell first

Snowflake bills for running warehouse time, not directly for individual queries. A warehouse consumes credits for every second it is running, including idle time, and has a 60-second minimum charge each time it resumes. SnowLens therefore asks: *what caused warehouse seconds to exist, which of those seconds can be removed safely, and did the account actually save after a change?*

The product is strictly advisory. It reads Snowflake telemetry and configuration, calculates and ranks opportunities, and records human actions; it never changes Snowflake configuration itself.

The normal user flow is:

1. Connect Snowflake and run an extraction.
2. Review the Home page for account-level spend, opportunity, and data-quality signals.
3. Investigate warehouses, consolidation candidates, storage waste, or cost incidents.
4. Use Coverage to understand which data access limits the analysis.
5. Track analysis provenance in Runs and proved savings in Ledger.

## Important language for a presentation

| Say this | Do not say this |
|---|---|
| "estimated addressable savings" | "guaranteed savings" |
| "p10-p90 range; p50 is the midpoint" | "this will save exactly $X" |
| "verified after a human change" | "saved" when no change has been observed |
| "idle cost is money spent while a warehouse is awake without query work" | "idle queries" |
| "the portfolio total is de-duplicated" | "sum of every recommendation card" |

Credits are the native unit. Dollar figures appear only when SnowLens has an organization billing rate or a configured manual rate card.

---

## Screen Inventory

| Screen | Route | Main question it answers |
|---|---|---|
| Connection gate | Before the app opens | Can SnowLens connect and create a live analysis run? |
| Home | `/` | How much are we spending, how much may be addressable, and where should we look first? |
| Warehouses | `/warehouses` | Which warehouses consume the estate's credits and show idle behavior? |
| Warehouse detail | `/warehouses/:name` | What is happening inside one warehouse and what actions relate to it? |
| Consolidation | `/consolidation` | Can two warehouses be combined without unsafe queueing or workload conflict? |
| Storage | `/storage` | Which data objects appear redundant or reclaimable? |
| Coverage | `/coverage` | What data can the service role access, and which analyses are enabled? |
| Incidents | `/incidents` | Did spend spike abnormally or shift upward persistently? |
| Ledger | `/ledger` | How do estimated opportunities compare with verified savings? |
| KPI Glossary | `/glossary` | What does a KPI code mean, how is it calculated, and what access does it need? |
| Runs | `/runs` | Which analysis runs exist, what did each use, and how can data be shared safely? |

**Current implementation note:** `/backlog/*` redirects to Home. A richer opportunity backlog is in the product design, but is not currently a separate live screen.

---

## 1. Connection Gate

### What is on the screen

- SnowLens identity and a clear statement that no sample or local data is displayed.
- Missing environment-variable and configuration errors when connection setup is incomplete.
- **Connect Snowflake and analyze usage** button.
- Progress while extraction runs, including failure feedback.

### Feature

The button starts a Snowflake extraction, polls the analysis run until it completes, then opens the product only when a completed live run exists. This prevents dashboards from showing invented or stale demonstration data.

### Business significance

This is the trust boundary: every number in the UI should trace to a real extraction run, not a mocked dashboard.

---

## 2. Home: Cost and Opportunity Overview

### What is on the screen

- A first-run explanation and **Pull compute + storage data** action when no analysis exists.
- A narrative summary of spend, estimated addressable value, verified savings, and open recommendations.
- Six headline KPI tiles.
- Daily compute-spend chart, split into idle and attributed spend.
- Idle-versus-attributed doughnut chart, labelled "Where the money goes."
- Top five quick-win recommendations.
- Up to four cost anomalies or regressions that need attention.
- A portfolio-honesty notice explaining that overlapping opportunities are not added twice.

### Headline KPIs

| KPI | What the user sees | Measurement and source | Business significance |
|---|---|---|---|
| A1, Compute credits | Total compute spend in dollars for the lookback period | Sum `CREDITS_USED` from `SNOWFLAKE.ACCOUNT_USAGE.METERING_HISTORY`; dollars = credits x effective credit rate | Invoice anchor. If this cannot reconcile to the bill, downstream recommendations are not credible. |
| J1, Addressable | De-duplicated estimated savings at p50 | Union of the winning opportunity claims after arbitration; bootstrap daily observations to derive p10/p50/p90 | The defensible portfolio opportunity. It avoids counting the same warehouse seconds in both auto-suspend and consolidation recommendations. |
| J8, Verified saved | Savings that have been proved, not predicted | Difference-in-differences between a changed warehouse and unchanged comparable controls, using daily compute credits | Separates a recommendation from a real financial outcome and controls for seasonality or broad demand changes. |
| J6, Open opps | Number of recommended actions, split between quick wins and structural work | Count of active opportunities that passed safety gates and arbitration | Shows the actionable work queue and the mix of easy changes versus larger engineering initiatives. |
| A8, Attribution coverage | Percentage of compute SnowLens can explain through query attribution | `SUM(CREDITS_ATTRIBUTED_COMPUTE_QUERIES) / SUM(CREDITS_USED_COMPUTE)` from `WAREHOUSE_METERING_HISTORY` | Honesty metric. Low coverage means less of the bill can be assigned to query work; it is not treated as a data error. |
| J5, Ignored cost | Estimated value still left behind by old unacted-on recommendations | Sum of p50 estimates for recommendations older than the ignore threshold that remain unobserved | Quantifies the cost of inaction and gives FinOps a basis for escalation. |

### Charts and supporting metrics

| Component | Measurement and source | Why it matters |
|---|---|---|
| Daily compute spend | Daily metering split into used compute and the attribution gap, primarily `WAREHOUSE_METERING_HISTORY` | Shows whether cost is stable, rising, and how much is productive query work versus idle uptime. |
| Idle uptime, B1/B2 | `B1 = CREDITS_USED_COMPUTE - CREDITS_ATTRIBUTED_COMPUTE_QUERIES`; `B2 = B1 / CREDITS_USED_COMPUTE` | Idle cost is money paid for empty seconds. A material idle share often reveals an auto-suspend, scheduling, or consolidation opportunity. |
| Attributed queries, A9 | Attributed query credits plus a pro-rata allocation of idle cost to the queries that kept a warehouse active | Establishes a fairer team or workload cost view than attributed credits alone. |
| Quick wins | Recommended, `quick_win`-bucket opportunities ranked by priority | Focuses attention on changes that are high-confidence, low-effort, and low-risk. |
| Incidents | Latest anomaly and regression detections | Avoidable cost may be an operational issue today, not merely an optimization backlog item. |

### Suggested talk track

"This is the executive starting point. We begin with the bill anchor, then show a conservative, de-duplicated estimate of what may be removable. The charts distinguish useful query work from paid idle time, and the lower panels tell us what can be acted on quickly versus what needs investigation."

---

## 3. Warehouses: Estate Browser

### What is on the screen

- A table of every analyzed warehouse.
- Columns for warehouse name, spend in credits, idle share, uptime hours, resume count, open opportunities, and current size.
- A colored idle-share bar and row selection to open the detail page.

### KPIs

| KPI | Measurement and source | Business significance |
|---|---|---|
| A4, Spend by warehouse | Sum `CREDITS_USED` grouped by warehouse from `WAREHOUSE_METERING_HISTORY` | Identifies where the account's credit spend is concentrated. |
| B2, Idle share | Idle credits divided by used compute credits | Quickly reveals warehouses paying heavily for empty running time. It is assessed alongside absolute spend, not in isolation. |
| B7, Uptime hours | Union of `RESUME_WAREHOUSE` to `SUSPEND_WAREHOUSE` intervals from `WAREHOUSE_EVENTS_HISTORY` | The denominator for warehouse-cost analysis. It represents the actual billed-running time being managed. |
| B8, Resume count | Count `RESUME_WAREHOUSE` events | Frequent resumes may expose 60-second-minimum billing waste or overly aggressive auto-suspend. |
| Open opportunities | Count of live recommendations associated with the warehouse | Shows where the product already has a specific action, not just an interesting metric. |
| Warehouse size | Point-in-time `SHOW WAREHOUSES` snapshot | Allows a user to interpret credits and possible right-sizing actions in the correct rate context. |

### Feature

This screen is the estate triage view. It allows a FinOps owner to find the largest spend, largest idle share, or most recommendation-rich warehouse and drill directly into its supporting evidence.

---

## 4. Warehouse Detail: One Warehouse Investigation

### What is on the screen

- Back navigation to the estate table.
- Warehouse name and idle-share badge.
- Configuration and KPI tiles: size/scaling policy, total credits, idle credits and share, auto-suspend setting, uptime, and resumes.
- 24-hour representative load-profile chart.
- Related open opportunities with confidence and estimated credits.
- Latest 20 warehouse events, especially resume and suspend events.
- Top query shapes with run count and consumed credits.

### KPIs and evidence

| KPI / evidence | Measurement and source | Business significance |
|---|---|---|
| Size and scaling policy | `SHOW WAREHOUSES` snapshot | Configuration is necessary to judge the credit rate, auto-suspend baseline, and multi-cluster capacity. It is not available from ordinary Account Usage history. |
| Total credits | Warehouse-level sum from `WAREHOUSE_METERING_HISTORY` | Quantifies the warehouse's financial importance. |
| B1/B2, Idle credits and share | Metering used compute less attributed query compute; then divide by used compute | Finds paid runtime where no attributable work is occurring. |
| Auto-suspend and B13 candidate | Current setting from `SHOW WAREHOUSES`; candidate comes from minimizing the observed inter-query gap cost over timeout values | Finds a setting that balances idle time against resume-minimum charges instead of blindly recommending a very low timeout. |
| B7/B8, Uptime and resumes | Resume/suspend events | Explains how the warehouse's operating pattern creates cost. |
| Load profile, B3-B6 | Five-minute `WAREHOUSE_LOAD_HISTORY`: `AVG_RUNNING`, `AVG_QUEUED_LOAD`, `AVG_QUEUED_PROVISIONING`, `AVG_BLOCKED` | Separates busy workload, queue pressure, cold-start pressure, and lock contention. `AVG_RUNNING` is a load ratio, not CPU utilization. |
| Events | `WAREHOUSE_EVENTS_HISTORY` | Connects cost changes to resizes, resumes, suspends, and cluster changes. |
| Top query shapes, E1/E2 | Native `QUERY_PARAMETERIZED_HASH`, execution counts from aggregate/query history, and attributed credits by hash | Reveals recurring patterns that dominate spend or keep a warehouse awake. |

### Feature

This is the evidence page behind an action. A reviewer can see both the financial impact and the operational risk signals before asking a platform or data team to change a warehouse.

---

## 5. Consolidation: Warehouse Merge Explorer

### What is on the screen

- One card per recommended warehouse pair or group.
- Warehouse names, uptime-overlap percentage, estimated p50 dollar value, and redundant uptime hours removed.
- "Savings arithmetic" showing separate uptime, merged union uptime, and the difference.
- Safety-gate results.
- An empty state when no safe candidate meets the criteria.

### KPIs

| KPI | Measurement and source | Business significance |
|---|---|---|
| C2, Uptime intervals | Reconstructed `RESUME_WAREHOUSE` to `SUSPEND_WAREHOUSE` intervals from `WAREHOUSE_EVENTS_HISTORY` | Establishes the real running windows for each warehouse. |
| C3, Temporal overlap | Busy-time Jaccard: $|busy(A) \cap busy(B)| / |busy(A) \cup busy(B)|$, using `WAREHOUSE_LOAD_HISTORY` buckets where `AVG_RUNNING > 0` | Low busy overlap is favorable: the workloads need compute at different times, so one shared warehouse can often serve both without new contention. |
| C4, Consolidation saving | $(\Sigma uptime_i - |\cup uptime_i|) \times credits\_per\_hour(target)$ | Uses observed interval mathematics rather than an elasticity model. It is the headline reason consolidation can be a defensible saving. |
| C5, Combined peak concurrency | 99th percentile of aligned member `AVG_RUNNING` totals | Estimates the concurrent load the merged warehouse must absorb. |
| C6, Capacity feasibility | C5 must be less than or equal to configured target capacity | Hard safety gate. A potential saving is suppressed if the merged target could create queueing. |
| C7-C10, affinity and compatibility gates | Table overlap from `ACCESS_HISTORY` when available, role overlap from `QUERY_HISTORY`, workload class, and size/config compatibility | Protects against harmful merges that combine incompatible security models, SLAs, data access patterns, or warehouse sizes. |

### Feature

Consolidation candidates are not automatically executed. The page provides arithmetic and gates for a human architecture decision. It is especially useful because it converts a vague idea, "can we merge these?", into observable uptime and capacity evidence.

### Suggested talk track

"The saving is not the sum of idle percentages. It is the redundant warehouse-up time we can remove after checking that the combined workload fits safely. The visual compares today's separate uptime with the union of the same intervals after consolidation."

---

## 6. Storage: Redundancy and Reclaimability

### What is on the screen

- Guidance and an action to run the heavier object-level storage extraction after a compute run.
- Clear unavailable states when there is no completed run or storage analysis has not run.
- Four storage KPI tiles.
- A table of top redundant object families, including sample members, kind, member count, retirement candidates, and reclaimable bytes.
- Extraction timestamp and object count.

### KPIs

| KPI | Measurement and source | Business significance |
|---|---|---|
| Retirement candidates | Deterministic object-level analysis of access history, dependencies, and metadata | A review list of objects that may be retired only after human validation. |
| H1, Reclaimable bytes | `max(0, active + time_travel + failsafe - retained_for_clone)` using `TABLE_STORAGE_METRICS` | Avoids overstating storage savings by accounting for bytes that cannot yet be reclaimed because of clone retention. |
| Redundant families | Object families detected from metadata, columns, dependencies, and access patterns | Identifies duplicated or near-duplicated data structures that create ongoing storage and maintenance overhead. |
| Dead but maintained | Objects not meaningfully read but still generating maintenance cost | Targets automatic clustering, search optimization, or retention that creates cost without serving users. |
| Top redundant families | Ranked by reclaimable bytes | Helps teams direct review effort toward the largest concrete storage opportunity first. |

### Data sources and access

The analysis uses `ACCESS_HISTORY`, object metadata, dependencies, and storage metrics. Strong cold-data recommendations need governance access (`GOVERNANCE_VIEWER`, T3); metadata-only signals can work at lower access levels but are weaker.

### Business significance

Storage can be meaningful, but it is usually cheaper than continuously running oversized warehouses. This page helps prioritize real storage cleanup without letting it distract from high-credit compute waste.

---

## 7. Coverage: Access Tiers and Data Availability

### What is on the screen

- Introductory explanation of access tiers T0 through T4.
- Current access tier and attribution-coverage percentage.
- Pass/fail capability list for data sources.
- Opportunity types enabled at the current tier.
- Opportunity types disabled and the minimum tier required to unlock each.

### KPI

| KPI | Measurement and source | Business significance |
|---|---|---|
| A8, Attribution coverage | Attributed compute divided by used compute from `WAREHOUSE_METERING_HISTORY` | Indicates how well SnowLens can explain compute spend. Below 80% is a cue to investigate grants or adaptive warehouse behavior, not a reason to fabricate attribution. |

### Access tiers

| Tier | What it unlocks |
|---|---|
| T0 | No usable access; analysis cannot proceed. |
| T1 | Core compute FinOps: metering, idle, auto-suspend, consolidation, query shapes, and incidents. Requires `USAGE_VIEWER`; config-dependent features also need `SHOW WAREHOUSES`. |
| T2 | Objects and dependency metadata, tag coverage, and stronger storage/feature signals. Requires `OBJECT_VIEWER`. |
| T3 | Object access lineage, cold-data detection, and stronger consolidation affinity. Requires `GOVERNANCE_VIEWER` and an eligible Snowflake edition. |
| T4 | Organization-level cost, contract rates, and multi-account rollups. Requires organization usage/billing access. |

### Feature

Coverage makes degradation explicit. Rather than silently issuing weak conclusions, SnowLens explains which opportunities are available, which are disabled, and what incremental grant would improve the analysis.

---

## 8. Incidents: Anomaly and Regression Monitor

### What is on the screen

- Four headline tiles: active anomalies, regressions, total incidents, and open incidents.
- One card per incident with warehouse name, type, detection time, and a plain-language explanation.
- Empty state when the latest run found no incidents.

### KPIs

| KPI | Measurement and source | Business significance |
|---|---|---|
| G1, Daily cost anomaly | Robust z-score on daily warehouse credits, typically flagging $|z| > 3.5$, from `WAREHOUSE_METERING_HISTORY` | Detects a daily spend level that is unusually high relative to its normal history without letting outliers distort the baseline. |
| Cost regression | Persistent step increase: recent daily-credit average versus the earlier observation window; the UI calls out a change over 20% | Finds cost that rose and stayed high, often due to a new scheduled job, changed workload, or warehouse configuration change. |
| Open incidents | Count whose status is `open` | Shows the unresolved operational risk, distinct from a long-term optimization opportunity. |

### Why robust statistics are used

Spend is seasonal and right-skewed. A robust median/MAD method is less fooled by the unusual spikes it is trying to detect than a simple average and standard deviation. The product presents these as incidents to investigate, not as automatic proof of waste.

---

## 9. Ledger: Estimated Versus Verified Savings

### What is on the screen

- Four summary tiles: verified saved credits, estimated credits across changes, realization ratio, and verified-change count.
- Change ledger showing warehouse(s), opportunity type, estimated p50, verified point estimate, and status.
- Realization-by-type bars.
- Explanation of the verification standard.

### KPIs

| KPI | Measurement and source | Business significance |
|---|---|---|
| J8, Verified savings | Difference-in-differences: $(treated_{post} - treated_{pre}) - (control_{post} - control_{pre})$; savings are positive when treated spend falls more than comparable unchanged controls | The product's strongest financial claim. It avoids crediting SnowLens for a quiet month or organization-wide demand decline. |
| Estimated savings | Sum of p50 estimates shown across ledger entries | Planning context only. It must not be confused with the de-duplicated portfolio headline or with cash actually saved. |
| J7, Realization rate | `sum(verified savings) / sum(estimated p50)` for verified opportunities of a type | Shows whether the model's estimates become reality and supports calibration of future recommendations. |
| Verified changes | Ledger entries with an available verification point | Measures adoption and evidence maturity, not just the size of the recommendation pipeline. |

### Verification process

SnowLens passively detects a human-applied change through later `SHOW WAREHOUSES` snapshots, warehouse events, query cadence, or storage signals. It waits for enough post-change observations, selects unchanged control warehouses or shapes with similar workload and diurnal behavior, then computes the difference-in-differences result. If the confidence interval spans zero, the ledger records zero rather than claiming a win.

### Business significance

The Ledger lets FinOps report two separate numbers honestly: estimated portfolio potential and verified realized savings. That separation is essential for executive trust.

---

## 10. KPI Glossary: Metric Dictionary

### What is on the screen

- A visual explanation that a letter identifies the KPI family and a number identifies the metric.
- Family filter buttons for Foundation, Warehouse efficiency, Consolidation, Query efficiency, Recurring workload, Workload class, Anomalies, Storage, Feature ROI, and Opportunity results.
- Search by KPI code or name.
- A detailed definition panel for the selected KPI: availability, plain-English meaning, formula, source, and reference.

### Feature

This is the product's built-in audit and education screen. It gives a reviewer access requirements and documented calculation logic instead of forcing them to trust a label on a dashboard tile.

### KPI families

| Family | What it measures | Why it matters |
|---|---|---|
| A | Foundation and spend attribution | Reconciles the bill and describes where credits go. |
| B | Warehouse efficiency | Finds paid uptime that is not becoming useful work. |
| C | Consolidation | Tests whether workloads can share compute safely. |
| D | Query efficiency | Identifies scans, spills, queues, failures, and costly behavior. |
| E | Recurring workload | Finds repeated patterns whose cadence or uptime impact makes them valuable to fix. |
| F | Workload classification | Provides safety context: BI, ETL, batch, and other workloads tolerate different changes. |
| G | Anomalies and regression | Detects unusual or persistent cost growth. |
| H | Storage | Finds reclaimable, cold, redundant, or over-maintained data. |
| I | Feature ROI | Compares Snowflake feature maintenance cost to delivered benefit. |
| J | Opportunity and verification | Measures prioritized potential, adoption, and proven outcomes. |

---

## 11. Runs: Analysis History and Provenance

### What is on the screen

- **Pull new data** action to start a new live compute-and-storage extraction.
- **Export snapshot** action to download a portable analysis database.
- **Import snapshot** action to load a shared `.db` snapshot without direct Snowflake credentials.
- A run-history table with run ID, creation/completion time, status, access tier, opportunity count, warehouse count, and capability count.
- Clear import and extraction errors.

### KPIs / operational metrics

| Field | Measurement | Business significance |
|---|---|---|
| Run ID and timestamps | Application-created provenance for each extraction and analysis cycle | Allows every displayed number to be associated with a reproducible input window. |
| Status | Running, complete, or failed | Prevents incomplete data from being represented as finished analysis. |
| Tier and capabilities | Access-probe result captured at run time | Explains why a past run may have fewer opportunities or weaker evidence than a later one. |
| Opportunity and warehouse counts | Counts produced by that run | Quick health check for data completeness and changes in analytical output. |

### Business significance

Runs support auditability and collaboration. A central team can export a snapshot and let another reviewer inspect the same analysis without sharing live Snowflake credentials.

---

## Common Questions and Direct Answers

### Are the dollar amounts real?

The credits are measured from Snowflake telemetry. Dollar conversion is real only when SnowLens can obtain an effective credit rate from `ORGANIZATION_USAGE.USAGE_IN_CURRENCY_DAILY` and `RATE_SHEET_DAILY`, or when an approved manual rate exists in configuration. Otherwise, the product should show credits only.

### Why do query-attributed credits not equal the warehouse bill?

Query attribution deliberately excludes idle time, very short queries, cloud services, and some adaptive-warehouse cases. The gap is expected. SnowLens exposes it as idle cost and attribution coverage instead of hiding it.

### How is "addressable" measured?

Each recommendation claims a specific kind of avoidable cost, often an interval of billed warehouse uptime. SnowLens resolves conflicts so only one recommendation can claim the same uptime seconds, then bootstraps the remaining claims into a p10-p90 range. The displayed p50 is a sorting and planning midpoint, not a promise.

### How is "verified saved" measured?

After detecting that a person applied a recommendation, SnowLens compares the changed workload's before/after credit change with unchanged comparable controls. This difference-in-differences design removes much of the seasonal and organization-wide variation that makes simple before/after comparisons misleading.

### Does SnowLens make changes in Snowflake?

No. It has no execution feature and is designed not to need `MODIFY` or `OPERATE` privileges. Humans review and implement recommendations; SnowLens detects and measures the outcome afterward.

---

## Source and Trust Boundaries

| Source category | Examples | Used for |
|---|---|---|
| Billing and warehouse telemetry | `METERING_HISTORY`, `WAREHOUSE_METERING_HISTORY`, `WAREHOUSE_LOAD_HISTORY`, `WAREHOUSE_EVENTS_HISTORY` | Bill reconciliation, idle cost, uptime, queues, consolidation, and anomaly detection. |
| Query telemetry | `QUERY_HISTORY`, `AGGREGATE_QUERY_HISTORY`, `QUERY_ATTRIBUTION_HISTORY`, `QUERY_METERING_HISTORY` | Query shapes, gaps, spill, retries, recurring patterns, and attribution. |
| Live configuration | `SHOW WAREHOUSES` plus stored snapshots | Auto-suspend, size, cluster limits, scaling policy, and configuration-change verification. |
| Object and governance telemetry | `TABLE_STORAGE_METRICS`, `TABLES`, `COLUMNS`, `OBJECT_DEPENDENCIES`, `TAG_REFERENCES`, `ACCESS_HISTORY` | Storage, object redundancy, coldness, tags, and consolidation affinity. |
| Organization billing | `USAGE_IN_CURRENCY_DAILY`, `RATE_SHEET_DAILY` | Dollar conversion and contract-aware rate calculations. |
| SnowLens application data | Opportunities, claims, run records, verification ledger | Arbitration, ranking, status, ranges, and verified outcomes. |

Snowflake Account Usage data is not instantaneous. Typical latency ranges from roughly 45 minutes for query history to several hours for metering, warehouse load/events, and attribution; organization billing can lag about 24 hours. The product therefore reports the extraction run and observation window rather than claiming real-time monitoring.

---

## Five-Minute Presentation Outline

1. **Open Home:** "SnowLens explains our Snowflake bill in terms of warehouse uptime and turns that into a prioritized, conservative action list."
2. **Point to A1, J1, and J8:** "This is actual spend, this is de-duplicated estimated potential, and this is savings already verified after change."
3. **Open Warehouses and one detail page:** "We can drill from account spend into the exact warehouse configuration, idle pattern, events, and query shapes driving a recommendation."
4. **Open Consolidation:** "This is a differentiator: it uses observed time intervals and capacity gates to decide whether separate warehouses can safely share compute."
5. **Open Coverage:** "The product is transparent about access. Missing grants disable or weaken only the analyses that depend on them."
6. **Open Ledger and Runs:** "We preserve provenance for every analysis and keep estimated and verified savings separate."

For the full data contract, use [10-kpi-implementation-spec.md](10-kpi-implementation-spec.md), [16-kpi-source-access-matrix.md](16-kpi-source-access-matrix.md), and [07-verification.md](07-verification.md).
