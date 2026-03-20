# Your Alert Is Running. Your Budget Is Bleeding. Here's Why.

### A FinOps-Driven Teardown of a Kubernetes Log Alert That Was Costing 5× More Than It Needed To

---

## The One-Line Summary

A log-based alert on a Kubernetes workload was re-scanning the same log data **1,440 times per day** due to three compounding architectural anti-patterns. A structured FinOps review and query re-architecture reduced DDU consumption by **65–75%** — with zero reduction in alerting coverage or accuracy.

---

## Savings at a Glance

| Metric | Value |
|--------|-------|
| DDU cost reduction | **65–75%** |
| Repeated log scans eliminated | **~80% of total scan volume** |
| Daily evaluation cycles removed | **1,440 full re-scans/day** |
| Alert accuracy compromised | **None** |
| Davis AI correlation enabled | **Yes** (was absent before) |
| Maintenance window suppression | **Now works** (was ignored before) |

---

## Table of Contents

1. [Context & Business Stakes](#1-context--business-stakes)
2. [The FinOps Problem — What This Alert Actually Costs](#2-the-finops-problem--what-this-alert-actually-costs)
3. [The Original Query — Anti-Patterns Identified](#3-the-original-query--anti-patterns-identified)
4. [The Optimised Query — Short-Term Fix](#4-the-optimised-query--short-term-fix)
5. [The Architecture Fix — Log Metrics](#5-the-architecture-fix--log-metrics)
6. [Entity Binding — The Missing Link](#6-entity-binding--the-missing-link)
7. [Full Cost Analysis](#7-full-cost-analysis)
8. [Architecture Decision Record](#8-architecture-decision-record)
9. [Key Takeaways for Platform Architects](#9-key-takeaways-for-platform-architects)

---

## 1. Context & Business Stakes

### What Is Being Monitored

A **SonarQube Data Center Edition (DCE)** deployment running on **Kubernetes (AKS/EKS/GKE)** in a pre-production environment. SonarQube DCE is a code quality gate — it sits between feature branches and production releases. If it is unhealthy, code analysis results are unreliable and release decisions are compromised.

Two containers are in scope:

| Container | Role | Why It Matters |
|-----------|------|----------------|
| `sonarqube-dce` | Application compute node — runs the analysis engine | If this errors, the analysis pipeline is broken |
| `sonarqube-dce-search` | Elasticsearch search node — indexes analysis results | If this errors, results are incomplete or missing |

The monitoring requirement is simple: **alert when either container generates ERROR-level logs at an abnormal rate**.

### The Business Stakes

This is not just operational hygiene. In a pre-production environment, this alert is a **release gate signal**:

| If Missed | Business Impact |
|-----------|----------------|
| SonarQube errors during a release window | False quality gate result — bad code passes or good code is blocked |
| Search node degradation undetected | Incomplete analysis results delivered to developers |
| Alert fires to wrong team | Incident routed incorrectly, SLA clock missed |
| No Davis AI correlation | Root cause (node pressure, storage issue) not surfaced automatically |

Getting the alert right is not just technical correctness — it is **business correctness**.

---

## 2. The FinOps Problem — What This Alert Actually Costs

> **FinOps Principle: Cost Visibility Before Optimisation.**
> You cannot reduce what you have not measured. Before rewriting a single line of DQL, decompose the cost model.

### How Dynatrace Log Alerting Consumes DDUs

Dynatrace licensing for log-based alerting is denominated in **Davis Data Units (DDUs)**. Every time a DQL alert query executes, it consumes DDUs proportional to the **volume of log data scanned**.

```
Daily DDU Cost = Log Volume per Scan × Evaluation Cycles per Day

Example — 1-minute evaluation, 5-minute lookback window:
  → 1,440 evaluation cycles per day
  → Each cycle re-scans the last 5 minutes of logs
  → Same log record is read ~5 times per day on average
  → Effective DDU multiplier: 5× on raw log volume
```

### The Four Cost Drivers in the Original Query

| Driver | The Problem | DDU Impact |
|--------|-------------|-----------|
| `scanLimitGBytes:-1` | Overrides the 500GB default cap — scans ALL data with no ceiling | Maximum possible per cycle — no budget guardrail whatsoever |
| `matchesPhrase()` on structured fields | Phrase-tokenised search on indexed dimensional attributes | +15–25% CPU/IO overhead per scan |
| `lookup []` MZ join in query | Cross-store join: log store + entity topology, every cycle | +20–30% overhead per cycle |
| 1-minute evaluation loop | 1,440 complete re-scans per day | Multiplies all above costs ×1,440 |

> **Note on `scanLimitGBytes`:** The Dynatrace default (when the parameter is omitted entirely) is **500GB** — not unlimited. Setting `-1` explicitly overrides even that safety cap, removing all budget guardrails. This is the most dangerous anti-pattern because it silently allows unbounded DDU consumption on high-volume log streams.

---

## 3. The Original Query — Anti-Patterns Identified

See [`query_v1.dql`](./query_v1.dql) for the full annotated query.

```dql
// ❌ ANTI-PATTERN 1: Unbounded scan — no DDU guardrail
fetch logs, scanLimitGBytes:-1

// ❌ ANTI-PATTERN 2: matchesPhrase on structured fields
//    These are indexed dimensional attributes — not free-text log body content
//    Full-text tokenised scan is unnecessary and expensive here
| filter matchesPhrase(k8s.cluster.name, "my-kubernetes-cluster")
    AND matchesPhrase(status, "ERROR")
    AND (
        matchesPhrase(k8s.container.name, "sonarqube-dce")
        OR matchesPhrase(k8s.container.name, "sonarqube-dce-search")
    )

// ❌ ANTI-PATTERN 3: Cross-store lookup join at query time
//    Fetches the entire entity topology table and joins it on EVERY evaluation cycle
//    At 1-min frequency: 1,440 topology fetches per day
| lookup [
    fetch dt.entity.kubernetes_cluster
    | expand managementZones
  ], lookupField:id, sourceField:dt.entity.kubernetes_cluster, prefix:"mz."
| filter mz.managementZones == "MY_MANAGEMENT_ZONE"

// ❌ ANTI-PATTERN 4: No entity binding in makeTimeseries
//    Alert fires as an orphaned event — invisible to Davis AI,
//    maintenance windows ignored, ownership routing broken
| makeTimeseries count(),
    time:{timestamp},
    by:{k8s.cluster.name, k8s.container.name},
    interval:1m
```

### What Each Anti-Pattern Costs the Business

| Anti-Pattern | Technical Problem | Business Cost |
|---|---|---|
| `scanLimitGBytes:-1` | Scans entire log store every cycle | No DDU budget control — unbounded spend |
| `matchesPhrase` on structured fields | Tokenised search on indexed attributes | 15–25% wasted compute per cycle |
| Lookup MZ join in query | Topology join 1,440×/day | 20–30% overhead; slows evaluation latency |
| No entity binding | Alert is orphaned from the entity model | Davis blind, MZ enforcement broken, SLA risk |

---

## 4. The Optimised Query — Short-Term Fix

See [`query_v2.dql`](./query_v2.dql) for the full annotated version.

```dql
// ✅ FIX 1: Hard DDU cap — budget guardrail enforced
fetch logs, scanLimitGBytes:1

// ✅ FIX 2: Exact equality on structured fields
//    Hits the index directly — no tokenisation, no full-text traversal
| filter k8s.cluster.name == "my-kubernetes-cluster"
    AND status == "ERROR"
    AND (
        k8s.container.name == "sonarqube-dce"
        OR k8s.container.name == "sonarqube-dce-search"
    )

// ✅ FIX 3: Entity binding guard
//    Prevents null entity IDs from polluting the timeseries dimensions
| filter isNotNull(dt.entity.kubernetes_cluster)

// ✅ FIX 4: Entity ID included in grouping
//    dt.entity.kubernetes_cluster dimension value = entity ID format (TYPE-HEXID)
//    Alert engine auto-promotes this to dt_source_entity
//    Result: Davis AI correlation enabled, maintenance windows respected
| makeTimeseries errorCount = count(),
    time:{timestamp},
    by:{k8s.cluster.name, k8s.container.name,
        k8s.namespace.name, dt.entity.kubernetes_cluster},
    interval:1m

// ✅ FIX 5: MZ filtering moved to alert definition level
//    Eliminates the lookup join entirely
//    MZ enforcement is native at the alert engine — zero cross-store cost
```

### What Changed and Why

| Change | Before | After | Saving |
|--------|--------|-------|--------|
| Scan limit | `-1` (unbounded) | `1` (1GB cap) | Budget guardrail — predictable spend |
| Filter method | `matchesPhrase()` | Exact `==` equality | 15–25% per cycle |
| MZ enforcement | Query-time `lookup` join | Alert definition level | 20–30% per cycle |
| Entity binding | Not present | `dt.entity.*` in `by:{}` | Davis AI + MZ + maintenance windows |

---

## 5. The Architecture Fix — Log Metrics

> **Query optimisation reduces cost. Architecture redesign eliminates the root cause.**

These are not the same thing. The optimised query still re-scans logs on every evaluation cycle. The architectural fix removes the scan loop entirely.

### The Core Problem with DQL Log Alerts

```
DQL Log Alert — execution model:

  Every evaluation cycle (e.g., every 1 minute):
    → Scan log store for lookback window (e.g., 5 minutes of logs)
    → Compute aggregation on raw records
    → Evaluate threshold
    → Discard intermediate results
    → Repeat next cycle

  1,440 cycles/day × 5-min lookback = same log record read ~5× per day
  You are paying to re-discover what you already knew.
```

### The Log Metric Architecture

```
Log Metric — execution model:

  At log ingest time (once, streaming):
    → Log record arrives in the pipeline
    → Metric rule evaluated ONCE against the record
    → Matching record increments the counter for its dimensions
    → Metric data point stored in metric store

  Alert engine evaluates the pre-aggregated metric — ZERO log scanning
  DDU cost = metric storage only (orders of magnitude cheaper)
```

### Log Metric Configuration

See [`log_metric_config.md`](./log_metric_config.md) for the full spec.

**Location in Dynatrace:** Log Management → Log Metrics → Add metric

| Parameter | Value | Reason |
|-----------|-------|--------|
| Metric key | `log.sonarqube_dce.error_count` | Follows `log.*` namespace convention |
| Filter: status | `== "ERROR"` | Exact equality — index, not scan |
| Filter: container | `sonarqube-dce` OR `sonarqube-dce-search` | Scoped to application only |
| Dimensions | `k8s.container.name`, `k8s.namespace.name`, `dt.entity.kubernetes_cluster` | Segmentation + entity binding |
| Measure | Occurrence (count) | Error rate per minute |
| MZ scope | Configured at **alert level** — not in metric rule | No join cost |

### Metric Event Alert Configuration

| Alert Parameter | Value |
|-----------------|-------|
| Metric | `log.sonarqube_dce.error_count` |
| Aggregation | Per-minute rate |
| Threshold | Static initially → anomaly detection after baseline |
| Dimension filter | `k8s.container.name` = target containers |
| Entity type | Kubernetes Cluster (auto-bound via `dt.entity.kubernetes_cluster`) |
| MZ filter | Target management zone — at alert config level |

---

## 6. Entity Binding — The Missing Link

This is the mechanism most DQL alert implementations get wrong. It is also the one that creates the most silent operational failures.

### What `dt_source_entity` Actually Does

When Dynatrace evaluates a metric alert, it needs to answer two questions:
1. **What is the signal?** → the `count()` timeseries value
2. **Who owns this signal?** → the entity the problem card attaches to

The `dt_source_entity` rule answers question 2.

**How the auto-promotion works:**

```
makeTimeseries produces a timeseries with named dimensions
    ↓
Alert engine scans dimension VALUES for entity ID format: TYPE-HEXID
    ↓
e.g., KUBERNETES_CLUSTER-A1B2C3D4 → valid entity ID format detected
    ↓
That dimension is promoted to dt_source_entity
    ↓
Problem card created → attached to KUBERNETES_CLUSTER-A1B2C3D4
```

You do not need to name a field `dt_source_entity`. You need the **value** to be a resolvable entity ID. Adding `dt.entity.kubernetes_cluster` to `by:{}` achieves this because the field carries the entity ID as its value.

### What Breaks Without Entity Binding

| Capability | With `dt.entity.*` in `by:{}` | Without |
|---|---|---|
| Problem card | Attached to entity | Orphaned — no owner |
| Management zone access | Enforced | Not enforced — wrong teams may see it |
| Maintenance windows | Respected — alert suppressed | **Ignored** — fires during maintenance |
| Davis AI correlation | Participates in root cause analysis | Isolated — Davis cannot connect it |
| Ownership routing | Inherited from entity | None — notification may not route |
| Entity event timeline | Alert visible on entity page | Not visible |

> See the [`/entity-binding`](../../entity-binding/) section of this cookbook for a deeper reference on entity binding patterns across alert types.

---

## 7. Full Cost Analysis

See [`cost_analysis.md`](./cost_analysis.md) for detailed modelling.

### DDU Reduction Per Change

| Change | Mechanism | Estimated DDU Reduction |
|--------|-----------|------------------------|
| `matchesPhrase` → exact `==` | Index lookup vs full-text scan | 15–25% per cycle |
| Remove `lookup` MZ join | Eliminates cross-store join | 20–30% per cycle |
| `scanLimitGBytes` cap | Hard 1GB guardrail | 10–20% on high-volume streams |
| Log Metric architecture | Ingest-time evaluation vs repeated scan | **60–80% of total log alert DDU** |
| **All changes combined** | Compound effect | **65–75% total reduction** |

### Scenario Modelling

| Scenario | Log Volume | Eval Frequency | DDU Reduction |
|----------|-----------|----------------|---------------|
| Conservative | Low | 5-min cycles | ~50–60% |
| Realistic | Medium | 1–2 min cycles | **65–75%** |
| High-volume enterprise | High | 1-min cycles | ~75–85% |

### Fleet-Wide Multiplier

In a managed or multi-tenant Dynatrace environment, this is not a one-alert saving. Every container error rate alert using the DQL log alert pattern instead of Log Metrics is incurring the same repeated-scan cost.

**The FinOps leverage point:** standardise on Log Metrics for all container error rate alerts across the estate. The DDU saving is not additive — it compounds across every similar alert pattern deployed at scale.

---

## 8. Architecture Decision Record

| Decision | Chosen | Rejected | Reason |
|----------|--------|----------|--------|
| Filter method | Exact `==` equality | `matchesPhrase()` | Structured fields have indexes — use them |
| MZ enforcement | Alert definition level | Query-time `lookup` join | Eliminates 1,440 cross-store joins/day |
| Alerting model | Log Metric + Metric Event | DQL log alert | Ingest-time evaluation vs repeated scan |
| Entity binding | `dt.entity.*` in `by:{}` | Unbound `makeTimeseries` | Davis correlation + MZ + maintenance windows |
| Scan limit | 1GB hard cap | Unlimited (`-1`) | Budget predictability and DDU guardrail |
| Segmentation | By container + namespace | Cluster-level aggregate | Independent signal per workload component |

---

## 9. Key Takeaways for Platform Architects

**1. Cost visibility is the first design step — not an afterthought.**
Before writing any alert query, model the DDU cost: volume × frequency × multiplier. Know what each DQL operator costs at scale.

**2. Know your filter operators — `matchesPhrase()` is not a general-purpose filter.**

Three operators exist for log filtering:

| Operator | Use case | Performance |
|---|---|---|
| `==` | Structured field, exact value, case-sensitive | Fastest — uses the index directly |
| `matchesValue()` | Structured field, case-insensitive or wildcard (`*`) needed | Good — index-aware with pattern support |
| `matchesPhrase()` | Unstructured `content` body — phrase/word-boundary search | Expensive — full tokenised scan; wrong for structured attributes |

Never use `matchesPhrase()` on structured dimensional attributes — cluster name, container name, status. It ignores the index.

**3. Entity binding is not optional in production alerts.**
Every production alert must include a `dt.entity.*` dimension in `makeTimeseries by:{}`. Without it: no Davis correlation, no maintenance window suppression, no MZ enforcement, no ownership routing.

**4. Follow the Anomaly Detection DQL rules.**
Per official Dynatrace documentation, alert DQL queries must not use `from:`, `to:`, `sort`, or `limit`. Do not override the timeframe — the alert engine manages its own sliding window. `interval: 1m` is required for Anomaly Detection alerts to function correctly.

**5. Log Metrics is the production alerting architecture. DQL log alerts are for exploration.**
The Log Metric pattern — evaluate once at ingest, alert on the pre-aggregated metric — eliminates the repeated-scan cost model structurally, not incrementally.

**6. Architecture decisions at the pattern level multiply across the fleet.**
A single query optimisation saves DDUs on one alert. A pattern decision — standardise on Log Metrics — saves DDUs across every similar alert in the estate. That is the FinOps leverage point.

---

## Files in This Case Study

| File | Contents |
|------|----------|
| `README.md` | This document — full case walkthrough |
| [`query_v1.dql`](./query_v1.dql) | Original query with anti-patterns annotated inline |
| [`query_v2.dql`](./query_v2.dql) | Optimised query with fix annotations inline |
| [`log_metric_config.md`](./log_metric_config.md) | Log Metric definition spec and alert configuration |
| [`cost_analysis.md`](./cost_analysis.md) | Full DDU cost breakdown and scenario modelling |

---

*Part of the [DQL Cookbook](../../README.md) — real-world Dynatrace Grail patterns from enterprise-scale operations.*
*Pattern generalised from enterprise deployments. No client-specific data included.*
