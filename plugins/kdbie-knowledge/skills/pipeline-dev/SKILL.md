---
name: pipeline-dev
description: >
  Write kdb Stream Processor (SP) pipelines in q or Python. Use this skill whenever the user
  asks to build, create, or write a pipeline, connect readers/writers/operators, set up streaming
  ingestion, configure Kafka/S3/database connectors, add windowing or ML operators, or asks about
  .qsp.* / sp.* API usage. Also triggers for questions like "how do I read from Kafka into SP",
  "write a pipeline that decodes Avro", "tumbling window pipeline", or any task involving
  .qsp.run / sp.run.
---

# SP Pipeline Writer

A pipeline is a DAG: readers → operators → writers, chained in q with bracket notation or in
Python with `|`.

---

## Step 1 — Clarify (required before generating)

Before writing any code, confirm all four of these. Ask together in a single message for anything
not already stated:

| # | What to confirm | Examples |
|---|-----------------|---------|
| 1 | **Source** | Kafka, S3, HTTP, callback, database, expression |
| 2 | **Sink** | stream, database, Kafka, console, variable |
| 3 | **Language** | q, Python, or both |
| 4 | **Transform** | filter, map, window, join, ML, schema rename, none |

Do not generate a pipeline until all four are clear. If the user's request answers some but not
all, ask only about the gaps. Skip optional details (schema registry, auth) unless the user raises
them.

---

## Step 2 — Look up operator signatures

Don't guess API signatures — fetch them before generating code.

**Primary: kx-docs MCP** (use `mcp__kx-docs__search_kx_knowledge_sources` if available):

```
"SP readers fromKafka fromAmazonS3"
"SP writers toDatabase toStream toKafka"
"SP decode encode avro json csv"
"SP operators map filter apply merge split"
"SP window tumbling sliding"
"SP transform renameColumns replaceNull"
"SP ml minMaxScaler"
```

**Fallback: public docs** (if MCP unavailable — switch silently, only report if both fail):

Base URL: `https://code.kx.com/insights/api/stream-processor/`

| Need | Page |
|------|------|
| Read connectors | `readers.html` |
| Write connectors | `writers.html` |
| Decoders / Encoders | `decoders.html` / `encoders.html` |
| map / filter / apply / merge / split | `operators.html` |
| Window operators | `windows.html` |
| Transform utilities | `transform.html` |
| ML operators | `ml.html` |
| `.qsp.use` / operator config | `configuring-operators.html` |

---

## Step 3 — Generate the pipeline

**Hard rules:**
- `.qsp.run` / `sp.run(..)` called **exactly once**
- Readers have no upstream nodes; writers have no downstream nodes
- In q: always bracket notation (`.qsp.map[fn]`) — never prefix form
- Operator config via `.qsp.use` in q; keyword args in Python
- Prefer `v2` namespace when available (`.qsp.v2.read.fromAmazonS3`, `api_version=2` in Python)

**Output:** Show only the language the user specified in Step 1. If they said "both", use tab headers:

````
=== "q"
```q
.qsp.run
    .qsp.read.fromKafka[`trades; ...]
    ...
```

=== "Python"
```python
sp.run(
    sp.read.from_kafka('trades', ...)
    ...
)
```
````

**`.qsp.use` config patterns:**
```q
// Multi-key
.qsp.use (!) . flip (
    (`registry ; `$"http://schema-registry:8081");
    (`schemaType; "AVRO"))

// Single key
.qsp.use enlist[`exclude]!enlist excludedCols

// Two keys inline
.qsp.use `directWrite`database!(1b; "mydb")
```

**Topology patterns:**

| Pattern | Approach |
|---------|----------|
| Multiple sources | Pass a list to `.qsp.run`: `(trade; quote)` |
| Split (one source → multiple sinks) | Assign shared node to a variable or use `.qsp.split[]` |
| Merge streams as-is | `.qsp.union` |
| Join with a function | `.qsp.merge` |
| Metadata-aware transform | `.qsp.apply[{[op;md;data] ...}]` instead of `.qsp.map` |

For concrete code for any of these, read `references/patterns.md`.

---

## Step 4 — After generating

- Flag any parameters the user needs to fill in (topic names, brokers, table names, schemas).
- If the pipeline has branches or splits, briefly explain the topology.
- Offer to add windowing, error handling via `.qsp.apply`, or schema-registry config if relevant.
