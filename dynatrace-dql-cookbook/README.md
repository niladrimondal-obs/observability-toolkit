# DQL Cookbook — Real-World Dynatrace Grail Patterns from Enterprise-Scale Operations

> **"Most observability teams write queries. Senior architects write queries that don't cost a fortune to run."**

This cookbook is a collection of production-grade DQL patterns, architectural decisions, and FinOps-driven optimisations drawn from operating one of the largest managed Dynatrace environments globally — **1M+ monitored nodes, 1.5B+ DDUs, 1,000+ enterprise customers across multiple regions**.

Each case study follows the same structure: the business problem, the cost problem, the anti-patterns, the fix, and the quantified savings. Not tutorials. Real patterns from real scale.

---

## Why This Cookbook Exists

Dynatrace Grail and DQL are powerful. But power without architectural discipline burns licensing budget silently.

The most common observability cost leaks are not from bad intentions — they are from **good intentions applied without FinOps thinking**:

- Alerts that re-scan log data they already processed
- Queries that use full-text search on structured fields
- Topology joins executed thousands of times per day
- Metrics alerts with no entity binding — orphaned from Davis AI

This cookbook documents each of these patterns: what they look like, why they happen, and how to fix them — at the query level and at the architecture level.

---

## Structure

```
dynatrace-dql-cookbook/
│
├── log-alerting/
│   └── sonarqube-dce-error-rate/        ← Case Study #01
│       ├── README.md                    ← Full case: problem → fix → savings
│       ├── query_v1.dql                 ← Original query with anti-patterns annotated
│       ├── query_v2.dql                 ← Optimised query
│       ├── log_metric_config.md         ← Log Metric definition spec
│       └── cost_analysis.md             ← Query cost breakdown and scenario modelling
│
├── metrics/                             ← Coming soon
├── entity-binding/                      ← Coming soon
└── automation/                          ← Coming soon
```

---

## Case Studies

| # | Case | Domain | Cost Saving | Key Pattern |
|---|------|--------|-------------|-------------|
| 01 | [SonarQube DCE Error Rate Alert](./log-alerting/sonarqube-dce-error-rate/README.md) | Log Alerting | 65–75% query cost | Log Metrics vs repeated log scans |

> **Pricing model note:** All cost analysis in this cookbook is verified against official Dynatrace documentation. For Grail / Log Management and Analytics (DPS model): query cost = GiB scanned × rate card (published: $0.0035/GiB). Log Metrics are billed for Ingest & Process only — Query is not billed. Sources linked inline in each case study.

---

## Who This Is For

- Dynatrace platform engineers managing multi-tenant or MSP environments
- Architects designing observability platforms at scale
- FinOps practitioners looking for cost levers in monitoring infrastructure
- Anyone preparing for Dynatrace Professional certification or Solutions Architect interviews

---

## Author

**Observability Architect & Platform Lead** — 16 years in observability and AIOps, spanning Dynatrace, Splunk, ELK, Prometheus, and OpsBridge platforms across 1,000+ enterprise customers globally.

*Pattern observations are sanitised and generalised from enterprise deployments. No client-specific data.*

---

> If this helped you reduce query spend or improve your alerting architecture, a ⭐ on the repo is appreciated.
>
> **Repository:** [github.com/niladrimondal-obs/observability-toolkit/tree/main/dynatrace-dql-cookbook](https://github.com/niladrimondal-obs/observability-toolkit/tree/main/dynatrace-dql-cookbook)
