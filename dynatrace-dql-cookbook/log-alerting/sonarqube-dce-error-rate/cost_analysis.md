# Cost Analysis — Query Cost Reduction Breakdown

> **The goal of this document is not to give you a number. It is to give you a model — so you can apply it to your own environment and make an informed architectural decision.**

---

## Dynatrace Pricing Models for Log Alerting

Before running the numbers, identify which pricing model your environment is on. Both are affected by this case study's patterns.

### Model A — Log Monitoring Classic (DDU-based)
Each **log record ingested** consumes `0.0005 DDU`. This is an ingest cost — paid once per record regardless of query frequency. DQL alert queries add additional query overhead on top of ingest cost.
- Source: [docs.dynatrace.com — DDUs for Log Monitoring Classic](https://docs.dynatrace.com/docs/manage/subscriptions-and-licensing/monitoring-consumption-classic/davis-data-units/log-monitoring-consumption)

### Model B — Log Management and Analytics / Grail (DPS Query-based)
Query cost = **GiB of uncompressed data scanned per execution × rate card**.

| Metric | Value | Source |
|--------|-------|--------|
| Published query rate | **$0.0035 per GiB scanned** | Dynatrace blog, Dec 2024 |
| DDU equivalent weight | **1.70 DDU per GB read** | docs.dynatrace.com — DDUs for LMA |
| `fetch logs \| makeTimeseries` billing | **Billed as Query (GiB scanned)** | docs.dynatrace.com — Metrics powered by Grail |
| Log Metric Query billing | **Not billed — Ingest & Process only** | docs.dynatrace.com — Metrics powered by Grail |

> **This document models the Grail DPS Query cost** because that is where DQL alert evaluation patterns have the largest and most direct cost impact. The formula and multipliers apply to both models — substitute your own rate.

---

## The Query Cost Formula

The `scanLimitGBytes` parameter controls the amount of uncompressed data read by the fetch stage. The **default value is 500GB** when the parameter is omitted. If set to `-1`, all data in the query time range is scanned with no ceiling.
- Source: [docs.dynatrace.com — DQL data source commands](https://docs.dynatrace.com/docs/discover-dynatrace/platform/grail/dynatrace-query-language/commands/data-source-commands)

```
Daily Query Cost ≈ GiB Scanned per Execution × Evaluation Cycles per Day × Rate

Where:
  Evaluation Cycles per Day = 1,440   (at 1-minute evaluation frequency)
                            = 288     (at 5-minute frequency)
                            = 96      (at 15-minute frequency)

  GiB Scanned per Execution = Raw Log Volume per Minute × Lookback Window (minutes)

  scanLimitGBytes:-1  → overrides the 500GB default cap — unbounded scan cost
  scanLimitGBytes:1   → hard 1GB cap per cycle — recommended for scoped namespace alerts

Published rate example:
  100 GiB scanned/day × $0.0035/GiB = $0.35/day per alert
  With 5× repeated-scan multiplier:  effectively $1.75/day per alert
  At 1,440 cycles:                   each GiB/min of log volume costs ~$1.84/day
```

---

## The Repeated-Scan Multiplier

This is the most important number in the analysis and the one most teams miss.

```
If evaluation frequency = 1 minute
And lookback window     = 5 minutes

Then every log record generated at time T is scanned:
  → At T+1 min (cycle 1)
  → At T+2 min (cycle 2)
  → At T+3 min (cycle 3)
  → At T+4 min (cycle 4)
  → At T+5 min (cycle 5)

Effective scan multiplier: 5×
```

**A 100MB/minute log stream costs as if it is a 500MB/minute stream** when evaluated at 1-minute frequency with a 5-minute lookback.

---

## Anti-Pattern Cost Stacking

The original query compounds the base scan cost with three additional multipliers:

```
Base Query Cost (GiB scanned × $0.0035/GiB)
  × 1.0   (baseline)
  × 1.20  (matchesPhrase × 3 structured fields — est. +20% data touched per cycle)
  × 1.25  (lookup MZ join — est. +25% cross-store overhead per cycle)
  × 5.0   (repeated-scan multiplier at 1-min eval / 5-min lookback)
  ────────────────────────────────────────────────────────────────
  × 7.5×  effective cost vs the Log Metric architecture
```

This means the original query is paying roughly **7.5× more per day** than a well-architected Log Metric equivalent — which is not billed for Query at all.

---

## Reduction Per Optimisation

### Phase 1 — Query Optimisation (query_v2.dql)

| Change | Mechanism | Estimated Reduction |
|--------|-----------|-------------------|
| `matchesPhrase` → `==` | Index lookup vs full-text scan — less GiB touched per cycle | 15–25% per cycle |
| Remove `lookup` MZ join | Eliminates cross-store join — reduces scan overhead | 20–30% per cycle |
| `scanLimitGBytes` cap | Hard 1GB budget ceiling — prevents unbounded scans | 10–20% on high-volume streams |
| **Phase 1 total** | Compound effect on per-cycle GiB scanned | **~35–45% per-cycle reduction** |

> Phase 1 makes each scan cheaper. It does not remove the scan. You still pay per GiB × 1,440 cycles/day.

### Phase 2 — Architecture Shift (Log Metric)

| Change | Mechanism | Cost Reduction |
|--------|-----------|----------------|
| Log Metric at ingest | Evaluated once per record — Query not billed | **100% of query scan cost eliminated** |
| **Phase 2 effective saving** | Log Metric billed for Ingest & Process only | **Query cost = $0 vs DQL alert** |

> **Verified:** *"Log metrics are regular metrics that are billed for Ingest & Process (and Retain above 15 months), but not for Query."*
> Source: [docs.dynatrace.com — Metrics powered by Grail - Query](https://docs.dynatrace.com/docs/license/capabilities/metrics/dps-metrics-query)

> Phase 2 removes the root cause entirely. The query scan loop is eliminated — not optimised. The 65–75% figure in the combined estimate accounts for the residual Ingest & Process cost that remains even with Log Metrics.

### Combined

| Optimisation | Total Cost Reduction vs Original |
|---|---|
| Phase 1 only (query fix) | ~35–45% per-cycle cost reduction |
| Phase 2 only (Log Metric) | ~65–75% total (query cost eliminated; Ingest & Process remains) |
| Phase 1 + Phase 2 | **65–75% realistic band** (dominated by Phase 2) |

---

## Scenario Modelling

### Inputs

| Variable | Conservative | Realistic | Enterprise |
|----------|-------------|-----------|-----------|
| Log volume (namespace) | 50 MB/min | 200 MB/min | 600 MB/min |
| Evaluation frequency | 5 minutes | 1 minute | 1 minute |
| Lookback window | 5 minutes | 5 minutes | 10 minutes |
| Repeated-scan multiplier | 1× | 5× | 10× |

### Effective Daily Scan Volume

| Scenario | Raw Volume/Day | Effective Volume (with multiplier) |
|----------|---------------|----------------------------------|
| Conservative | 72 GB | 72 GB (no repeated scan at 5-min freq) |
| Realistic | 288 GB | **1,440 GB** |
| Enterprise | 864 GB | **8,640 GB** |

### Query Cost Reduction Estimate

| Scenario | Estimated Total Cost Reduction |
|----------|-------------------------------|
| Conservative | ~50–60% |
| **Realistic** | **65–75%** |
| Enterprise | ~75–85% |

---

## Fleet-Wide FinOps Calculation

This is where the real value is — not on a single alert, but as a platform pattern.

### The Multiplier Effect

```
Single alert saving (Grail DPS model):
  DQL log alert:  pays GiB scanned × rate × 1,440 cycles/day
  Log Metric:     pays Ingest & Process once — Query = $0
  Saving per alert: essentially 100% of query cost eliminated

Number of similar container error rate alerts: N
Fleet query cost saving:  N × (per-alert daily scan cost)
```

For a managed or multi-tenant Dynatrace environment with many container-based workloads:

| Fleet Size | Alerts Converted to Log Metrics | Query Cost Eliminated |
|---|---|---|
| 10 similar alerts | 10 | 10 × full per-alert daily scan cost |
| 50 similar alerts | 50 | Proportional — compounds with log volume |
| 260+ client estate | Hundreds | **Structural query cost reduction at estate level** |

> **FinOps Principle: Pattern-Level Decisions Outperform Alert-Level Tuning.**
> Converting one alert to a Log Metric eliminates that alert's query scan cost entirely. Standardising Log Metrics as the default for all container error rate alerts eliminates scan cost continuously, across every deployment, at scale. The saving is not marginal — query billing ceases to exist for those alert patterns.

---

## Cost vs Complexity Trade-off

This analysis would be incomplete without acknowledging the trade-off:

| Approach | Query Cost (Grail DPS) | Implementation Effort | Operational Complexity |
|---|---|---|---|
| Original DQL alert | GiB scanned × rate × 1,440 cycles/day | Low — one query | Low |
| Optimised DQL (v2) | ~35–45% lower per cycle | Low — refactor existing query | Low |
| Log Metric + Metric Event | **Zero query cost** — Ingest & Process only | Medium — new metric + alert config | Low (passive once deployed) |

The Log Metric approach has higher initial setup effort and zero additional operational complexity once deployed. The metric rule runs passively at ingest with no ongoing maintenance. For production alerting at scale, the setup cost is paid once. The query cost saving is permanent.

---

## What This Analysis Does Not Cover

| Item | Note |
|------|------|
| Exact rate card pricing | Published DPS rate: $0.0035/GiB scanned. Your contract rate may differ — use Account Management → Subscription → Usage for actuals |
| Log ingest cost (Ingest & Process) | Fixed cost per GB ingested regardless of alert architecture — not reduced by this change |
| Log Metric Ingest & Process cost | Small, fixed cost per GB ingested into the metric pipeline — negligible vs repeated scan query cost |
| Retain cost | 0.30 DDU/GB/day for stored data — separate from query cost, not affected by alert architecture |
| Classic Log Monitoring DDU model | Per-record weight: 0.0005 DDU/record. Source: [docs.dynatrace.com — DDUs for Log Monitoring Classic](https://docs.dynatrace.com/docs/manage/subscriptions-and-licensing/monitoring-consumption-classic/davis-data-units/log-monitoring-consumption) |
| Alert DQL prohibited patterns | `from:`, `to:`, `sort`, `limit` — prohibited in Anomaly Detection DQL. See `log_metric_config.md` |
| Grail scan optimisations | Dynatrace applies query optimisations that can discount irrelevant data by up to 98%. Actual scanned GiB may be lower than raw volume estimates. Source: [docs.dynatrace.com — LMA Query](https://docs.dynatrace.com/docs/license/capabilities/log-analytics/dps-log-query) |

---

## How to Apply This to Your Environment

**Step 1 — Identify your pricing model**
- Classic DDU: `Account Management → License → Davis Data Units → Log Monitoring pool`
- Grail DPS: `Account Management → Subscription → Cost and Usage Details → Log Management and Analytics – Query`

**Step 2 — Find your log volume for the target namespace**
```
fetch logs
| filter k8s.namespace.name == "<YOUR-NAMESPACE>"
| summarize count(), totalGiB = sum(dt.system.data_size) / 1073741824
```
Or: `Log Management → Storage Overview → filter by namespace`

**Step 3 — Find your current alert evaluation frequency**
`Alerting → Metric Events → [your alert] → Evaluation window`

**Step 4 — Calculate your current daily scan volume**
```
Daily GiB Scanned = (Log Volume GiB/min) × (Lookback minutes) × (Eval cycles/day)

Example: 0.2 GiB/min × 5 min lookback × 1,440 cycles = 1,440 GiB/day
At $0.0035/GiB: $5.04/day per alert
After Log Metric conversion: ~$0 query cost/day
```

**Step 5 — Verify your actual query consumption**
```dql
fetch bizevents
| filter event.kind == "BILLING_USAGE_EVENT"
    AND event.type == "Log Management & Analytics - Query"
| makeTimeseries sum(billed_bytes) / 1073741824, interval:1d
```
Source: [docs.dynatrace.com — Manage DPS costs](https://docs.dynatrace.com/docs/manage/dynatrace-platform-subscription/cost-management)

---

*Part of the [DQL Cookbook](../../README.md)*
*Repository: [github.com/niladrimondal-obs/observability-toolkit/tree/main/dynatrace-dql-cookbook](https://github.com/niladrimondal-obs/observability-toolkit/tree/main/dynatrace-dql-cookbook)*
