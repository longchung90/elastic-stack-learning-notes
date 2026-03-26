# elastic-stack-field-notes
### A practitioner's growing field notes on the Elastic Stack — built from real sessions, real data, real outputs

![Elasticsearch](https://img.shields.io/badge/Elasticsearch-8.x-005571?style=flat&logo=elasticsearch&logoColor=white)
![Kibana](https://img.shields.io/badge/Kibana-8.x-005571?style=flat&logo=kibana&logoColor=white)
![Logstash](https://img.shields.io/badge/Logstash-8.x-005571?style=flat&logo=logstash&logoColor=white)
![Filebeat](https://img.shields.io/badge/Filebeat-8.x-005571?style=flat&logo=beats&logoColor=white)
![ES|QL](https://img.shields.io/badge/ES%7CQL-pipeline--based-0077CC?style=flat)
![ILM](https://img.shields.io/badge/ILM-hot%20warm%20cold%20frozen-E8A838?style=flat)
![Index Mapping](https://img.shields.io/badge/Index%20Mapping-explicit-3DBE8C?style=flat)
![Status](https://img.shields.io/badge/status-in%20progress-orange?style=flat)
![Last Updated](https://img.shields.io/badge/last%20updated-March%202026-lightgrey?style=flat)

[![Stack Overview](https://img.shields.io/badge/view-Stack%20Overview-005571?style=flat)](https://longchung90.github.io/elastic-stack-learning-notes/stack_overview.html)
[![Query Cookbook](https://img.shields.io/badge/view-Query%20Cookbook-0077CC?style=flat)](https://longchung90.github.io/elastic-stack-learning-notes/esql_query_cookbook.html)

> **This repo is a living document.** Sections are added as sessions are completed. Not everything is here yet — and that is intentional.

---

## What This Repo Is

Hands-on field notes built from real practice sessions on a local Elasticsearch + Kibana cluster. Every query result is real. Every mapping output is real. Every insight came from actually running the code, hitting the errors, and understanding why.

This is not a copy of the official docs. It is the notes you wish existed when you started.

---

## Current Status

```
elastic-stack-field-notes/
│
├── README.md                          ← you are here
├── stack_overview.html                ← interactive Elastic Stack tool summary
│
└── Notebooks/
    ├── ESQL_Complete_Guide.ipynb      ✅ ES|QL full reference + real query outputs
    ├── shards_in_Es.ipynb             ✅ Shards, Lucene, shard pruning, node roles
    └── esql_query_cookbook.html       ✅ 15 queries — interactive HTML version
```

---

## What Is Covered Today

### ✅ Notebooks/shards_in_Es.ipynb

The foundation. Everything else in Elasticsearch builds on this. Covers what a shard is in plain and technical terms, the Lucene index instance, node roles and the 1,000 shard limit, and why `@timestamp` enables shard pruning.

**Key concept:**

```
Time filter arrives at coordinating node
        ↓
ES checks each shard's @timestamp min/max in metadata
— no documents read yet —
        ↓
Shards outside the window → SKIPPED entirely
Shards inside the window → SEARCHED in parallel
        ↓
Results merged and returned
```

This is called **shard pruning** — and it is why the time picker is always the first filter you set.

---

### ✅ Notebooks/ESQL_Complete_Guide.ipynb

Full ES|QL reference built from live queries against `kibana_sample_data_logs`. Every command is shown with real outputs and annotated with the insight it reveals.

**The ES|QL pipeline model:**

```
FROM source
| EVAL   → compute new columns first
| WHERE  → filter on raw or computed fields
| STATS  → aggregate and group
| SORT   → order results
| LIMIT  → cap output
```

Every `|` passes its output as the input to the next stage. No subqueries. No nesting.

---

### ✅ esql_query_cookbook.html

15 ready-to-run ES|QL queries grouped by concept — response codes, traffic patterns, bandwidth, EVAL logic, WHERE filters, and multi-pipe chains. Interactive and live at the link above.

---

## Dataset Used

All notebooks and query outputs in this repo were built against the **Kibana Sample Web Logs** dataset — a built-in sample dataset shipped with Kibana.

### How to load it

1. Open Kibana → **Home** or **Add data**
2. Click the **Sample data** tab
3. Find **Sample web logs** → click **Install data**

No CSV upload, no external source, no credentials needed.

### What it contains

```
Index:      .ds-kibana_sample_data_logs-<date>-000001
Documents:  14,074
Type:       Data stream backing index
```

| Field | Type | Notes |
|---|---|---|
| `@timestamp` | `date` | Shard pruning key |
| `clientip` / `ip` | `ip` | CIDR-queryable |
| `geo.coordinates` | `geo_point` | Powers Kibana map view |
| `geo.src` / `dest` / `srcdest` | `keyword` | All traffic source = US only |
| `response` | `text + keyword` | ⚠️ Mapped as text — use `TO_INTEGER(response.keyword)` for range queries |
| `bytes` | `long` | Max ~20KB per response |
| `bytes_counter` | `long + counter` | TSDB ever-increasing total |
| `bytes_gauge` | `long + gauge` | TSDB point-in-time snapshot |
| `host` | `text + keyword` | 4 distinct hosts |
| `extension` | `text + keyword` | zip, deb, rpm, gz, css |
| `memory` / `phpmemory` | `double / long` | ⚠️ Null in every document — wasted mapping slots |

### Key facts before querying

- **All traffic originates from US** — `geo.src` returns only `"US"` for all 14,074 docs. Do not use this dataset to test source-country filtering.
- **No response exceeds ~20KB** — `WHERE bytes > 50000` returns zero rows. Check `MAX(bytes)` before writing range filters.
- **All 441 errors are 503** — there are no 500 errors. Response breakdown: `200 → 12,832` / `404 → 801` / `503 → 441`.
- **`response` is text, not integer** — range queries like `response >= 500` throw a `verification_exception`. Workaround: `EVAL is_error = CASE(TO_INTEGER(response.keyword) >= 500, 1, 0)`.

---

## What Is Coming Next

### 🔜 Index Build notebook
Explicit mapping, ILM policy, index templates, alias bootstrap, and Logstash integration. Covers why dynamic mapping breaks in production, all five ILM lifecycle phases, and the full Logstash → alias → rollover write path.

### 🔜 Hands-on session notebook
Full `kibana_sample_data_logs` session log — mapping audit, field type analysis, every query run and every error resolved. Includes the mapping audit visual and field type guide rendered inline.

### 🔜 Banking context notebook
ILM design for financial institutions — 7–10 year retention, regulatory compliance (Basel III, SOX, PCI-DSS), frozen phase for audit access, and snapshot policy vs ILM.

---

## The Stack

An interactive overview of all tools and how they connect is in [`stack_overview.html`](https://longchung90.github.io/elastic-stack-learning-notes/stack_overview.html).

| Layer | Tool | Role | Status in this repo |
|---|---|---|---|
| UI | **Kibana** | Visualise, query, manage | Used throughout — Dev Tools and Discover |
| Query | **ES\|QL** | Pipeline-based query language | ✅ Full reference complete |
| Core | **Elasticsearch** | Distributed storage + search | ✅ Shards covered · Index build coming |
| Pipeline | **Logstash** | Parse, enrich, route events | 🔜 Coming in index build notebook |
| Agent | **Filebeat** | Lightweight log collector | 🔜 Coming in index build notebook |

---

## How to Use This Repo Right Now

1. **Open `Notebooks/shards_in_Es.ipynb`** — start here. Shards are the prerequisite concept for everything else in Elasticsearch. The shard pruning section directly explains why the time picker is always set first.

2. **Open `Notebooks/ESQL_Complete_Guide.ipynb`** — full ES|QL command reference with real outputs. The decision tree at the end tells you which command to reach for in any situation.

3. **Open [`esql_query_cookbook.html`](https://longchung90.github.io/elastic-stack-learning-notes/esql_query_cookbook.html)** — 15 interactive queries, live in the browser.

4. **Open [`stack_overview.html`](https://longchung90.github.io/elastic-stack-learning-notes/stack_overview.html)** for a visual summary of every tool in the Elastic Stack and how they connect.

---

## Notation Used in This Repo

| Symbol | Meaning |
|---|---|
| ✅ | Section complete |
| 🔜 | Coming in the next session |
| ⚠️ | Known problem or production risk |
| 💡 | Insight drawn from a real query result |
| `keyword` | Exact match — filter bar, aggregations, sorting |
| `text` | Full-text search — word-level search only |
| `date` | Time fields — enables shard pruning |
| `ip` | IP addresses — enables CIDR range queries |

---

## Official References

All concepts in this repo are grounded in the official Elastic documentation.

### Elasticsearch Core

| Topic | Link |
|---|---|
| Clusters, nodes, and shards | [elastic.co/docs](https://www.elastic.co/docs/deploy-manage/distributed-architecture/clusters-nodes-shards) |
| Index lifecycle management | [elastic.co/guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html) |
| ILM phases and actions | [elastic.co/docs](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management/index-lifecycle) |
| Configure a lifecycle policy | [elastic.co/docs](https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management/configure-lifecycle-policy) |
| Explicit mapping | [elastic.co/docs](https://www.elastic.co/docs/manage-data/data-store/mapping/explicit-mapping) |
| Dynamic field mapping | [elastic.co/guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html) |
| Mapping types overview | [elastic.co/guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html) |
| How many shards should I have | [elastic.co/blog](https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster) |
| Shards and replicas guide | [elastic.co/search-labs](https://www.elastic.co/search-labs/blog/elasticsearch-shards-and-replicas-guide) |

### ES|QL

| Topic | Link |
|---|---|
| ES\|QL overview | [elastic.co/guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql.html) |
| ES\|QL commands reference | [elastic.co/guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html) |
| ES\|QL functions and operators | [elastic.co/guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html) |
| ES\|QL async query API | [elastic.co/guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-async-query-api.html) |
| ES\|QL query parameters | [elastic.co/guide](https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-rest.html) |

### Kibana

| Topic | Link |
|---|---|
| Kibana Discover | [elastic.co/guide](https://www.elastic.co/guide/en/kibana/current/discover.html) |
| Index Management UI | [elastic.co/guide](https://www.elastic.co/guide/en/kibana/current/index-management.html) |
| Sample data | [elastic.co/guide](https://www.elastic.co/guide/en/kibana/current/get-started.html) |

### Logstash & Beats

| Topic | Link |
|---|---|
| Logstash Elasticsearch output plugin | [elastic.co/guide](https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html) |
| Logstash grok filter | [elastic.co/guide](https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html) |
| Filebeat overview | [elastic.co/guide](https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-overview.html) |

---

*Built from hands-on Elasticsearch practice sessions — started March 2026*
*New sections added as sessions are completed*
