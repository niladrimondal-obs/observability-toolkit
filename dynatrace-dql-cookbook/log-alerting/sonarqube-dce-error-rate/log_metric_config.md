# Log Metric Configuration — Production Architecture Spec

> **This is Phase 2 — the architecture fix.**
> `query_v2.dql` reduces DDU cost by 35–45% by fixing the query.
> This document eliminates the root cause: the repeated-scan execution model.

---

## Why the Query Fix Is Not Enough

Even with all four anti-patterns fixed in `query_v2.dql`, the fundamental cost structure remains unchanged:

```
Every evaluation cycle (every 1 minute):
  → Scan log store for lookback window
  → Re-process log records you already processed last cycle
  → Discard results
  → Repeat

1,440 cycles/day × 5-minute lookback = each log record read ~5× per day
You are paying to re-discover what you already know.
```

The Log Metric architecture eliminates this loop entirely.

---

## The Architecture Shift

```
BEFORE — DQL Log Alert:

  Log record arrives
       ↓
  [stored in log store]
       ↓
  Alert evaluation cycle runs (every minute)
       ↓
  DQL scans log store ← YOU PAY HERE, 1,440× per day
       ↓
  Threshold evaluated
       ↓
  Alert fires or clears


AFTER — Log Metric + Metric Event:

  Log record arrives
       ↓
  Log Metric rule evaluated ONCE at ingest ← YOU PAY HERE, once per record
       ↓
  Matching record increments dimension counter
       ↓
  Metric data point stored in metric store
       ↓
  Alert engine evaluates pre-aggregated metric (no log scanning)
       ↓
  Alert fires or clears
```

**DDU cost for metric storage is orders of magnitude lower than repeated log scan DDUs.**

---

## Step 1 — Create the Log Metric

**Location:** Dynatrace → Log Management → Log Metrics → Add metric

### Metric Definition

```
# CONCEPTUAL SPEC — field names reflect UI concepts, not importable YAML keys
# Configure these settings in: Log Management → Log Metrics → Add metric

metric_key:   log.sonarqube_dce.error_count
              # Must start with "log." — enforced by Dynatrace

# Matcher (DQL Matcher syntax — evaluated once per log record at ingest)
# Log Metric matchers support: ==, matchesValue(), matchesPhrase(), isNotNull()
# For structured attributes, prefer == (fastest) or matchesValue() (case-insensitive)
# matchesPhrase() is valid but designed for unstructured content body — avoid on structured fields
matcher:
  status == "ERROR"
  AND (k8s.container.name == "sonarqube-dce"
    OR k8s.container.name == "sonarqube-dce-search")
  AND k8s.cluster.name == "<YOUR-CLUSTER-NAME>"

# Dimensions (select via "Add dimension" in the UI)
# Each dimension becomes a grouping axis on the resulting metric timeseries
dimensions:
  - k8s.container.name           # split by container → independent signal per workload
  - k8s.namespace.name           # split by namespace → environment-level segmentation
  - dt.entity.kubernetes_cluster # carries entity ID → enables dt_source_entity binding in alert

measure: Occurrence              # UI option: "Occurrence of log records" = count()
```

### Key Configuration Decisions

| Parameter | Value | Why |
|-----------|-------|-----|
| `metric_key` | `log.sonarqube_dce.error_count` | Must start with `log.` — Dynatrace enforces this prefix for all log metrics |
| Filter operators | `==` for structured fields | Fastest — exact index lookup. `matchesValue()` is valid if case-insensitive matching is needed |
| `matchesPhrase()` in filters | Avoid on structured fields | Valid syntax but designed for unstructured `content` body — unnecessary overhead on indexed attributes |
| `dt.entity.kubernetes_cluster` as dimension | Yes | Carries entity ID (format: `KUBERNETES_CLUSTER-XXXX`) into metric → auto-promoted to `dt_source_entity` in alert |
| MZ scope | **Not in metric rule** | Keep metric generic — apply MZ at alert level. One metric can then serve multiple environment-specific alerts |
| Measure | Occurrence | UI option: counts matching log records. No attribute value needed — we want event rate, not a numeric field sum |

> **Note:** Do not scope the metric rule to a specific management zone. Keep the metric generic and apply MZ scoping at the alert definition. This allows the same metric to be reused across multiple environment-specific alerts.

---

## Step 2 — Create the Metric Event Alert

**Location:** Dynatrace → Alerting → Metric Events → Add metric event

### Alert Configuration

```
# CONCEPTUAL SPEC — these are UI configuration concepts, not importable YAML
# Configure in: Settings → Anomaly Detection → Metric Events → Add metric event
# (or: Alerting → Metric Events depending on your Dynatrace version)

alert_name: "SonarQube DCE — Error Rate Anomaly"

# The metric key created in Step 1
metric_key: log.sonarqube_dce.error_count

# Aggregation applied over the evaluation window
aggregation: COUNT

# Sliding evaluation window — how far back each cycle looks
# 5 minutes handles normal log ingest latency gracefully
evaluation_window: 5m

# interval:1m is REQUIRED for Anomaly Detection alerts per Dynatrace docs
# Do NOT add from: or to: to the DQL query — causes evaluation failures
# Do NOT add sort or limit — prohibited in alert DQL

# Dimension filters — scope this alert to a specific deployment
# Leave k8s.namespace.name unset for fleet-wide coverage
dimension_filter_cluster:    "<YOUR-CLUSTER-NAME>"
dimension_filter_namespace:  "<YOUR-NAMESPACE>"   # optional

# Entity type — auto-resolved from dt.entity.kubernetes_cluster dimension value
# The alert engine detects KUBERNETES_CLUSTER-XXXX format and promotes it to dt_source_entity
entity_type: KUBERNETES_CLUSTER

# Management zone — configure here, NOT in the metric rule
# This enforces MZ access control without any query-time join cost
management_zone: "<YOUR-MANAGEMENT-ZONE>"

# Threshold strategy — start static, graduate to anomaly detection
threshold_type: STATIC         # or: ANOMALY_DETECTION after 2+ weeks of baseline data

# Static threshold settings (UI labels: "Alert condition")
# Raise alert when:
threshold_value:     10        # errors per minute
violating_window:    3         # 3 consecutive 1-min windows above threshold to open problem
dealerting_window:   5         # 5 consecutive windows below threshold to close problem

# Severity (UI: "Event severity")
severity: AVAILABILITY         # or PERFORMANCE — align to your classification policy
```

### Threshold Tuning Strategy

| Phase | Approach | When |
|-------|----------|------|
| Week 1–2 | Static threshold at observed P95 error rate | Establish baseline |
| Week 3–4 | Static at P99 — observe false positive rate | Refine |
| Month 2+ | Switch to anomaly detection | Baseline established |

---

## Step 3 — Validate Entity Binding

After the alert is created, trigger a test event and verify:

```
In Dynatrace:
  Problems → [find your test problem]
  → Check: "Affected Entity" shows KUBERNETES_CLUSTER-XXXX
  → Check: "Management Zone" shows the correct MZ
  → Check: Davis AI analysis section is populated (not "No context available")
```

If "Affected Entity" is empty, the `dt.entity.kubernetes_cluster` dimension is not resolving. Check:
1. OneAgent is deployed on the target cluster nodes
2. Log ingest enrichment is enabled for Kubernetes metadata
3. The cluster entity exists in the topology (Settings → Monitoring → Kubernetes)

---

## Critical Alert DQL Rules

These are enforced requirements from the [Dynatrace Anomaly Detection DQL guide](https://docs.dynatrace.com/docs/dynatrace-intelligence/anomaly-detection/anomaly-detection-app/davis-ad-dql-best-practice):

| Rule | Reason |
|------|--------|
| **`interval: 1m` is required** | Anomaly Detection evaluates every minute — other intervals may cause the alert to not work |
| **Do NOT use `from:` or `to:` in alert DQL** | Causes evaluation failures — the alert engine manages its own sliding window |
| **Do NOT use `sort`** | No impact on alerting logic, only reduces performance |
| **Do NOT use `limit`** | Causes dimensions to appear and disappear unpredictably, creating spurious alerts |
| **Use stable dimensions only** | Volatile dimensions (e.g., tag arrays) create new timeseries on every change, triggering false alerts |

The `query_v2.dql` file complies with all of these rules.

---

## Complete Architecture Comparison

| Dimension | DQL Log Alert | Log Metric + Metric Event |
|-----------|---------------|--------------------------|
| Scan on each evaluation | Yes — re-reads log store | **No** — metric already computed |
| DDU cost model | Volume × frequency × lookback | Metric storage only |
| Same record processed | ~5× per day (at 1-min/5-min) | **Once** — at ingest |
| Ingest lag handling | Needs wide lookback window | Handled natively by metric pipeline |
| Entity binding | Requires `dt.entity.*` in DQL `by:{}` | Configured as dimension on metric |
| MZ scoping | Query-time join (expensive) | Alert-level config (free) |
| Filter operators | Full DQL including `matchesPhrase()` | Exact equality only |
| Use case | Exploration, ad-hoc investigation | **Production alerting — always** |
| Baseline learning | Not available | Available (anomaly detection mode) |
| Estimated DDU vs DQL alert | Baseline | **65–75% lower** |

---

## Files Referenced

| File | Purpose |
|------|---------|
| [`query_v1.dql`](./query_v1.dql) | Original query — what we are replacing |
| [`query_v2.dql`](./query_v2.dql) | Optimised query — Phase 1 fix (use if Log Metrics not available) |
| [`cost_analysis.md`](./cost_analysis.md) | Full DDU modelling and scenario breakdown |

---

*Part of the [DQL Cookbook](../../README.md)*
