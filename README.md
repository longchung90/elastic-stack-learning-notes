# elastic-stack-field-notes
### Hands-on notes, reference guides, and real query results from a practitioner learning the Elastic Stack from the ground up

> Shards · Index Mapping · ILM · ES|QL · Logstash · Kibana · Production Patterns

---

## What This Repo Is

This repository documents a hands-on learning journey through the Elastic Stack — from core Elasticsearch concepts to production-grade index design, lifecycle management, and ES|QL querying. Every guide was built from real practice sessions on a live local cluster using real data.

All query results are real. All mapping outputs are real. Nothing is fabricated.

---

## Who This Is For

- Engineers new to Elasticsearch who want structured, practical notes
- Anyone transitioning from SQL or Splunk to ES|QL
- Teams building monitoring or observability pipelines on the Elastic Stack
- Anyone preparing to work with Elasticsearch in a banking or financial services context

---

## Repository Structure

```
elasticsearch-learning/
│
├── README.md                          ← you are here
│
├── 01_core_concepts/
│   ├── shards_explained.md            ← what a shard is, Lucene, shard pruning
│   ├── nodes_and_roles.md             ← data node vs coordinating node, shard limits
│   └── timestamp_and_pruning.md       ← @timestamp, shard metadata, cheapest filter
│
├── 02_index_build/
│   ├── index_complete_guide.md        ← auto-creation, mapping, ILM, registry, alias
│   ├── ilm_policy_reference.md        ← hot/warm/cold/frozen/delete phases
│   └── logstash_alias_relationship.md ← Logstash pipeline + alias end-to-end
│
├── 03_esql_reference/
│   ├── esql_complete_reference.md     ← full command reference with real results
│   └── esql_query_cookbook.md         ← 15 ready-to-run queries by use case
│
├── 04_hands_on/
│   ├── sample_data_session.md         ← session notes from kibana_sample_data_logs
│   ├── mapping_audit.md               ← field-by-field mapping analysis
│   └── query_results/
│       ├── response_codes.json
│       ├── traffic_by_hour.json
│       ├── geo_corridors.json
│       └── bandwidth_by_extension.json
│
└── 05_banking_context/
    └── ilm_banking_phases.md          ← 5-phase ILM for regulatory retention
```

---

## Guides in This Repo

### 01 — Core Concepts

| File | What it covers |
|---|---|
| `shards_explained.md` | What a shard is in plain terms and technical terms. The Lucene instance explanation. What "shard" means in other contexts (databases, Redis, blockchain). |
| `nodes_and_roles.md` | The difference between coordinating, data, and master nodes. The 1,000 shard per data node limit and why it exists. |
| `timestamp_and_pruning.md` | Why `@timestamp` is not the index. How Lucene stores shard metadata. Why the time filter is the cheapest filter. The full shard pruning flow with an example. |

---

### 02 — Index Build

| File | What it covers |
|---|---|
| `index_complete_guide.md` | Auto-creation vs explicit mapping. Every field type with plain-language explanations. Dynamic vs strict mapping. The full 5-step setup order: ILM policy → template → bootstrap → alias → write. |
| `ilm_policy_reference.md` | All five lifecycle phases. Rollover explained. Who decides the phase (ILM engine, every 10 minutes). Banking institution phase design with 7–10 year retention. Key CLI commands for checking ILM status. |
| `logstash_alias_relationship.md` | Full Logstash pipeline with annotated config. The `manage_template: false` trap. The end-to-end 13-step document flow from app server to Lucene segment. What breaks if you hardcode the index name instead of the alias. |

---

### 03 — ES|QL Reference

| File | What it covers |
|---|---|
| `esql_complete_reference.md` | Every command with syntax, real results, and annotated insights. The three Kibana layers. Common mistakes table. Async queries. Named parameters. Quick decision tree. Key numbers from the sample dataset. |
| `esql_query_cookbook.md` | 15 ready-to-run queries grouped by concept: response codes, traffic patterns, bandwidth, EVAL patterns, WHERE filters, multi-pipe chains. |

---

### 04 — Hands-On Session Notes

| File | What it covers |
|---|---|
| `sample_data_session.md` | Full session log from the `kibana_sample_data_logs` practice run. Every query run, every output received, every insight drawn. |
| `mapping_audit.md` | Field-by-field analysis of the sample data mapping. Scorecard: 8 good, 3 fix needed, 5 wasteful, 4 advanced. The `response` as text problem and workaround. |
| `query_results/` | Raw JSON outputs from every query run during the session for reference. |

---

### 05 — Banking Context

| File | What it covers |
|---|---|
| `ilm_banking_phases.md` | Why banks use all 5 phases. The frozen phase for 5-year audit access. Regulatory retention requirements (Basel III, SOX, PCI-DSS). Snapshot policy vs ILM. Cost efficiency breakdown per phase. |

---

## Key Concepts at a Glance

### The shard pruning chain

```
Time picker sets a window
        ↓
Elasticsearch checks each shard's @timestamp min/max in metadata
        ↓
Shards outside the window → SKIPPED (never read)
        ↓
Shards inside the window → SEARCHED in parallel
        ↓
Results merged and returned
```

### The ILM lifecycle

```
Hot (0–7d)    → active writes, fast SSD, 3 primary shards
     ↓ rollover at 50GB or 7 days
Warm (7–30d)  → read only, shrink to 1 shard, force merge, HDD
     ↓ min_age 30d
Cold (30–90d) → cheap storage, slow queries OK
     ↓ min_age 90d
Delete        → permanent removal
```

### The alias write path

```
Logstash → writes to "monitoring-logs" (alias, never changes)
                    ↓
           ES resolves alias → monitoring-logs-000047 (current hot index)
                    ↓
           ILM rolls over → creates monitoring-logs-000048
                    ↓
           Alias updated → now points to 000048
                    ↓
           Logstash next write → still "monitoring-logs" → now goes to 000048
           Zero config change. Zero downtime.
```

### ES|QL pipeline anatomy

```
FROM source
| EVAL   → compute new columns
| WHERE  → filter rows (on computed or raw fields)
| STATS  → aggregate and group
| SORT   → order results
| LIMIT  → cap output
```

---

## Real Data Facts — kibana_sample_data_logs

Quick reference for the practice dataset so you do not re-run exploratory queries.

| Fact | Value |
|---|---|
| Total documents | 14,074 |
| Index type | Data stream backing index (`.ds-` prefix) |
| Date range | ~30 days of synthetic data |
| All traffic source | US only (`geo.src = "US"` for every doc) |
| Top destination corridor | US:CN (2,621 requests) |
| Response code breakdown | 200 → 12,832 / 404 → 801 / 503 → 441 |
| Overall error rate | 3.13% (all errors are 503) |
| Max response size | ~20KB (no request exceeds 100KB) |
| Most common size band | Medium 1–10KB (85% of requests) |
| Top host | artifacts.elastic.co (6,488 requests, 46%) |
| `response` field type | ⚠️ text+keyword — mapped wrong, use `TO_INTEGER(response.keyword)` for range queries |
| `memory` / `phpmemory` | ⚠️ null in every document — wasted mapping slots |

---

## Environment

| Component | Version / Setup |
|---|---|
| Elasticsearch | Local single-node instance |
| Kibana | Co-located with ES |
| Sample data | `kibana_sample_data_logs` (installed via Kibana → Add data) |
| Dev Tools | Used for all raw API queries |
| Python | Jupyter notebook for session documentation |

---

## How to Use This Repo

1. **Start with `01_core_concepts/`** — understand shards, nodes, and `@timestamp` before writing any queries. These concepts are referenced everywhere else.

2. **Read `02_index_build/`** — understand how indices, mappings, ILM, and aliases connect before building anything in production.

3. **Use `03_esql_reference/esql_complete_reference.md`** as your daily query reference. The decision tree at the bottom tells you which command to reach for.

4. **Run the queries in `03_esql_reference/esql_query_cookbook.md`** against your own `kibana_sample_data_logs` dataset to verify and build intuition.

5. **Check `04_hands_on/mapping_audit.md`** to understand what a mapping analysis looks like and how to spot problems in your own indices.

---

## Notation Used in This Repo

| Symbol | Meaning |
|---|---|
| ⚠️ | Known problem or production risk |
| ✅ | Correct / recommended approach |
| 💡 | Insight from real query result |
| `keyword` | Use exact match — filter bar, aggregations, sorting |
| `text` | Use full-text search — match queries, word-level search |
| `date` | Use for time ranges — enables shard pruning |
| `ip` | Use for IP addresses — enables CIDR queries |

---

*Built from a hands-on Elasticsearch learning session — March 2026*
# elastic-stack-learning-notes
