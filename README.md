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

> **This repo is a living document.** Sections are added as sessions are completed. Not everything is here yet — and that is intentional.

---

## What This Repo Is

Hands-on field notes built from real practice sessions on a local Elasticsearch + Kibana cluster. Every query result is real. Every mapping output is real. Every insight came from actually running the code, hitting the errors, and understanding why.

This is not a copy of the official docs. It is the notes you wish existed when you started.

---

## Current Status

```
ELASTICSEARCH_LEARNING.../
│
├── README.md                          ← you are here
├── stack_overview.html                ← interactive Elastic Stack tool summary
│
└── Notebooks/
    ├── ESQL_Complete_Guide.ipynb      ✅ ES|QL full reference + real query outputs
    ├── shards_in_Es.ipynb             ✅ Shards, Lucene, shard pruning, node roles
    ├── esql_query_cookbook.html       ✅ 15 queries — interactive HTML version
    └── esql_query_cookbook.pdf        ✅ 15 queries — print-friendly PDF version
```

> **U** = untracked (new files not yet staged) · **M** = modified — commit when ready.

---

## What Is Covered Today

### ✅ Notebooks/shards_in_Es.ipynb

The foundation. Everything else in Elasticsearch builds on this.

**Key concept from this section:**

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

Full ES|QL reference built from live queries against `kibana_sample_data_logs`.

**Real data facts from the practice session** (`kibana_sample_data_logs`):

| Metric | Value |
|---|---|
| Total documents | 14,074 |
| Response breakdown | 200 → 12,832 / 404 → 801 / 503 → 441 |
| Overall error rate | 3.13% |
| Top host | artifacts.elastic.co (46% of traffic) |
| Top destination corridor | US:CN (2,621 requests) |
| All traffic source | US only |
| ⚠️ Known mapping issue | `response` is `text` not `integer` — use `TO_INTEGER(response.keyword)` for range queries |

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

## Dataset Used

All notebooks and query outputs in this repo were built against the **Kibana Sample Web Logs** dataset — a built-in sample dataset shipped with Kibana.

### How to load it

1. Open Kibana → **Home** or **Add data**
2. Click the **Sample data** tab
3. Find **Sample web logs** → click **Install data**

That's it. No CSV upload, no external source, no credentials needed.

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

### Key facts to know before querying

- **All traffic originates from US** — `geo.src` returns only `"US"` for all 14,074 docs. Do not use this dataset to test source-country filtering.
- **No response exceeds ~20KB** — `WHERE bytes > 50000` returns zero rows. Check `MAX(bytes)` before writing range filters.
- **All 441 errors are 503** — there are no 500 errors in this dataset. Response breakdown: `200 → 12,832` / `404 → 801` / `503 → 441`.
- **`response` is text, not integer** — the single biggest mapping gotcha. Range queries like `response >= 500` throw a `verification_exception`. Workaround: `EVAL is_error = CASE(TO_INTEGER(response.keyword) >= 500, 1, 0)`.

---

## What Is Coming Next

### ✅ esql_query_cookbook.html + esql_query_cookbook.pdf

15 ready-to-run ES|QL queries grouped by concept with annotated real outputs. The HTML version is interactive — collapsible sections per concept group. The PDF is print-friendly for desk reference.

---

## The Stack

An interactive overview of all tools is in [`stack_overview.html`](./stack_overview.html) — open it in a browser for the full visual card with descriptions.

| Layer | Tool | Role | Status in this repo |
|---|---|---|---|
| UI | **Kibana** | Visualise, query, manage | Used throughout — Dev Tools and Discover |
| Query | **ES\|QL** | Pipeline-based query language | ✅ Full reference complete |
| Core | **Elasticsearch** | Distributed storage + search | ✅ Shards covered · Index build coming |
| Pipeline | **Logstash** | Parse, enrich, route events | 🔜 Coming in 02_index_build |
| Agent | **Filebeat** | Lightweight log collector | 🔜 Coming in 02_index_build |

---

## How to Use This Repo Right Now

1. **Read `01_core_concepts/shards_explained.md`** — understanding shards is the prerequisite for everything else. The shard pruning concept directly explains why every query guide tells you to set the time picker first.

2. **Open `03_esql_reference/esql_complete_reference.md`** — use this as your daily query reference. The decision tree at the bottom tells you which command to reach for in any situation.

3. **Run the queries in `03_esql_reference/esql_query_cookbook.md`** — paste them into Kibana Dev Tools against your own `kibana_sample_data_logs` dataset. All 15 queries are tested and return real results.

4. **Watch this repo** — index build and hands-on session notes are the next sections to be added.

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

All concepts in this repo are grounded in the official Elastic documentation. Use these links to go deeper on any topic.

### Elasticsearch Core

| Topic | Link |
|---|---|
| Clusters, nodes, and shards | https://www.elastic.co/docs/deploy-manage/distributed-architecture/clusters-nodes-shards |
| Index lifecycle management | https://www.elastic.co/guide/en/elasticsearch/reference/current/index-lifecycle-management.html |
| ILM phases and actions | https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management/index-lifecycle |
| Configure a lifecycle policy | https://www.elastic.co/docs/manage-data/lifecycle/index-lifecycle-management/configure-lifecycle-policy |
| Explicit mapping | https://www.elastic.co/docs/manage-data/data-store/mapping/explicit-mapping |
| Dynamic field mapping | https://www.elastic.co/guide/en/elasticsearch/reference/current/dynamic-field-mapping.html |
| Mapping types overview | https://www.elastic.co/guide/en/elasticsearch/reference/current/mapping.html |
| How many shards should I have | https://www.elastic.co/blog/how-many-shards-should-i-have-in-my-elasticsearch-cluster |
| Shards and replicas guide | https://www.elastic.co/search-labs/blog/elasticsearch-shards-and-replicas-guide |

### ES|QL

| Topic | Link |
|---|---|
| ES\|QL overview | https://www.elastic.co/guide/en/elasticsearch/reference/current/esql.html |
| ES\|QL commands reference | https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-commands.html |
| ES\|QL functions reference | https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-functions-operators.html |
| ES\|QL async query API | https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-async-query-api.html |
| ES\|QL query parameters | https://www.elastic.co/guide/en/elasticsearch/reference/current/esql-rest.html |

### Kibana

| Topic | Link |
|---|---|
| Kibana Discover | https://www.elastic.co/guide/en/kibana/current/discover.html |
| Index Management UI | https://www.elastic.co/guide/en/kibana/current/index-management.html |
| Sample data | https://www.elastic.co/guide/en/kibana/current/get-started.html |

### Logstash & Beats

| Topic | Link |
|---|---|
| Logstash Elasticsearch output plugin | https://www.elastic.co/guide/en/logstash/current/plugins-outputs-elasticsearch.html |
| Logstash grok filter | https://www.elastic.co/guide/en/logstash/current/plugins-filters-grok.html |
| Filebeat overview | https://www.elastic.co/guide/en/beats/filebeat/current/filebeat-overview.html |

---

*Built from hands-on Elasticsearch practice sessions — started March 2026*
