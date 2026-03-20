# Cost Analysis — DDU Reduction Breakdown

> **The goal of this document is not to give you a number. It is to give you a model — so you can apply it to your own environment and make an informed architectural decision.**

---

## The DDU Cost Model for Log Alerting

The optional `scanLimitGBytes` parameter controls the amount of uncompressed data to be read by the fetch stage. The **default value is 500GB** when the parameter is omitted. If set to `-1`, all data available in the query time range is analysed with no ceiling.

```
Daily DDU Cost ≈ Log Volume per Scan (GB) × Evaluation Cycles per Day × DDU Rate

Where:
  Evaluation Cycles per Day = 1,440   (at 1-minute evaluation frequency)
                            = 288     (at 5-minute frequency)
                            = 96      (at 15-minute frequency)

  Effective Log Volume per Scan = Raw Volume per Minute × Lookback Window (minutes)

  scanLimitGBytes:-1  → overrides the 500GB default cap entirely
  scanLimitGBytes:1   → hard cap at 1GB per cycle — recommended for scoped alerts
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

The original query compounds this with three additional multipliers:

```
Base DDU Cost
  × 1.0   (baseline)
  × 1.20  (matchesPhrase × 3 fields — estimated +20% CPU/IO overhead)
  × 1.25  (lookup MZ join — estimated +25% cross-store overhead)
  × 5.0   (repeated-scan multiplier at 1-min eval / 5-min lookback)
  ─────────────────────────────────────────────────────────────────
  × 7.5×  effective cost vs optimal pattern
```

This means the original query is consuming roughly **7× more DDU than a well-architected equivalent**.

---

## Reduction Per Optimisation

### Phase 1 — Query Optimisation (query_v2.dql)

| Change | Mechanism | Estimated Reduction |
|--------|-----------|-------------------|
| `matchesPhrase` → `==` | Index lookup vs full-text scan | 15–25% per cycle |
| Remove `lookup` MZ join | Eliminates cross-store join | 20–30% per cycle |
| `scanLimitGBytes` cap | Hard budget ceiling | 10–20% on high-volume streams |
| **Phase 1 total** | Compound effect on per-cycle cost | **~35–45% reduction** |

> Phase 1 makes each scan cheaper. It does not reduce the number of scans.

### Phase 2 — Architecture Shift (Log Metric)

| Change | Mechanism | Estimated Reduction |
|--------|-----------|-------------------|
| Log Metric at ingest | Each record evaluated once vs 5× | **60–80% of total log alert DDU** |
| **Phase 2 total** | Eliminates repeated-scan model | **60–80% additional reduction** |

> Phase 2 removes the root cause. The scan loop is eliminated, not optimised.

### Combined

| Optimisation | Total DDU Reduction vs Original |
|---|---|
| Phase 1 only (query fix) | ~35–45% |
| Phase 2 only (Log Metric) | ~60–80% |
| Phase 1 + Phase 2 | **65–75% (realistic band)** |

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

### DDU Reduction

| Scenario | Estimated DDU Reduction |
|----------|------------------------|
| Conservative | ~50–60% |
| **Realistic** | **65–75%** |
| Enterprise | ~75–85% |

---

## Fleet-Wide FinOps Calculation

This is where the real value is — not on a single alert, but as a platform pattern.

### The Multiplier Effect

```
Single alert saving:        65–75% DDU reduction
Number of similar alerts:   N  (container error rate alerts in your estate)
Fleet saving:               N × (per-alert DDU saving)
```

For a managed or multi-tenant Dynatrace environment with many container-based workloads:

| Fleet Size | Alerts Using DQL Log Pattern | Fleet DDU Reduction (at 70% per alert) |
|---|---|---|
| 10 similar alerts | 10 | 10 × 70% = effectively 70% of that alert pool |
| 50 similar alerts | 50 | Proportional fleet-wide saving |
| 260+ client estate | Hundreds | **Structural DDU reduction at estate level** |

> **FinOps Principle: Pattern-Level Decisions Outperform Alert-Level Tuning.**
> Optimising one alert saves DDUs once. Standardising the Log Metric pattern as the default for all container error rate alerts saves DDUs continuously, across every deployment, at scale.

---

## Cost vs Complexity Trade-off

This analysis would be incomplete without acknowledging the trade-off:

| Approach | DDU Cost | Implementation Effort | Operational Complexity |
|---|---|---|---|
| Original DQL alert | Baseline (high) | Low — one query | Low |
| Optimised DQL (v2) | ~35–45% lower | Low — refactor existing query | Low |
| Log Metric + Metric Event | ~65–75% lower | Medium — new metric + alert | Low (once set up) |

**The Log Metric approach has higher initial setup effort and zero additional operational complexity once deployed.** The metric rule runs passively at ingest — there is nothing to maintain.

For production alerting at scale, the setup cost is paid once. The DDU saving is continuous.

---

## What This Analysis Does Not Cover

| Item | Note |
|------|------|
| Exact DDU rates | Vary by Dynatrace contract — use your own rates in the formula above |
| Log ingest DDU | Fixed cost regardless of alert architecture — not reduced by this change |
| Metric storage DDU | Small, fixed cost per metric key per day — negligible vs scan DDUs |
| OneAgent infrastructure DDU | Not related to log alerting — outside scope |
| Alert DQL query rules | `from:`, `to:`, `sort`, `limit` are prohibited in Anomaly Detection DQL — see `log_metric_config.md` |

---

## How to Apply This to Your Environment

1. Find your log volume for the target namespace: `Grail → Log Management → Storage Overview`
2. Find your current alert evaluation frequency: `Alerting → Metric Events → [alert] → Evaluation`
3. Find your lookback window: same alert definition
4. Plug into the formula: `Volume × Frequency × Lookback multiplier`
5. Apply the 65–75% reduction estimate to get your expected saving

---

*Part of the [DQL Cookbook](../../README.md)*
