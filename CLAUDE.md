# System Design Wiki — Schema

This is the schema for an LLM-maintained personal wiki on system design and distributed systems engineering. You (the LLM) own the `wiki/` layer entirely. The human owns `raw/`. You never modify files in `raw/`.

## Directory layout

```
system-design/
├── CLAUDE.md              ← this file; the schema
├── raw/
│   └── articles/          ← clipped markdown articles (immutable; human adds these)
└── wiki/
    ├── index.md           ← catalog of all wiki pages; update on every ingest
    ├── log.md             ← append-only chronological log; update on every operation
    ├── overview.md        ← high-level synthesis of the whole wiki; update periodically
    ├── concepts/          ← architecture patterns, distributed systems concepts
    ├── systems/           ← specific systems and technologies (Kafka, Cassandra, etc.)
    ├── trade-offs/        ← comparison and trade-off pages
    ├── case-studies/      ← how specific companies solved specific problems
    └── sources/           ← one summary page per source in raw/
```

## Page conventions

**Filenames**: lowercase, hyphenated. E.g. `event-sourcing.md`, `apache-kafka.md`.

**Frontmatter** (YAML, at top of every wiki page):
```yaml
---
title: "Human-readable title"
category: concept | system | trade-off | case-study | source
tags: [tag1, tag2]
sources: [source-slug, ...]   # which raw articles informed this page
updated: YYYY-MM-DD
---
```

**Cross-references**: use Obsidian wiki-links: `[[page-name]]`. Link liberally — every mention of a named system or pattern that has its own page should be linked.

**Section structure** for concept pages:
- What it is (1-3 sentences)
- How it works
- When to use / when not to use
- Trade-offs
- Real-world examples
- See also

**Section structure** for system pages:
- What it is / primary use case
- Architecture overview
- Key design decisions
- Strengths and weaknesses
- Comparison with alternatives (links to trade-off pages)
- See also

**Section structure** for case-study pages:
- Company and context
- Problem they faced
- Solution and architecture
- Key design decisions and why
- Outcomes / lessons
- See also

**Section structure** for source pages:
- Source metadata (title, author, URL if known, date)
- Key takeaways (3-7 bullet points)
- Concepts introduced or reinforced
- Contradictions or updates to existing wiki pages
- Pages updated during ingest

## Operations

### Ingest

When the human says "ingest [filename]" or drops a source and asks you to process it:

1. Read the source file from `raw/articles/`.
2. Discuss key takeaways with the human briefly — confirm what matters most before writing.
3. Create `wiki/sources/<slug>.md` with the source summary.
4. Identify which concept, system, trade-off, and case-study pages need updating. Create new pages as needed; update existing ones.
5. For every page touched, update its `sources:` frontmatter and `updated:` date.
6. Update `wiki/index.md` — add new pages, update summaries of changed pages.
7. Append to `wiki/log.md`:
   ```
   ## [YYYY-MM-DD] ingest | <Source Title>
   - Source: `raw/articles/<filename>`
   - Pages created: ...
   - Pages updated: ...
   - Notes: ...
   ```

A single ingest typically touches 5-15 wiki pages. Don't shortcut this — the value of the wiki comes from the cross-references being maintained.

### Query

When the human asks a question:

1. Read `wiki/index.md` to identify relevant pages.
2. Read those pages (and any they link to that are relevant).
3. Synthesize an answer, citing specific wiki pages.
4. If the answer is non-trivial (a comparison, an analysis, a synthesis that doesn't exist yet), offer to file it as a new wiki page. Good answers compound the knowledge base.
5. Append to `wiki/log.md`:
   ```
   ## [YYYY-MM-DD] query | <Short question>
   - Pages consulted: ...
   - Notes: ...
   ```

### Lint

When the human asks you to "lint" or "health-check" the wiki:

1. Read `wiki/index.md` and scan all pages.
2. Report:
   - **Contradictions**: claims on different pages that conflict.
   - **Stale pages**: pages whose `sources` list contains articles that have since been superseded by newer ingests.
   - **Orphans**: pages with no inbound links from other wiki pages.
   - **Gaps**: important concepts mentioned in passing but lacking their own page.
   - **Missing cross-references**: mentions of named entities that have pages but aren't linked.
3. Suggest new sources to look for based on gaps.
4. Append a lint entry to `wiki/log.md`.

## Domain-specific guidance

This wiki covers **distributed systems and system design**. The most valuable things to track:

- **Patterns**: CQRS, Event Sourcing, Saga, Circuit Breaker, Sidecar, Bulkhead, Backpressure, Consistent Hashing, Quorum, Two-Phase Commit, etc.
- **Trade-off axes**: consistency vs. availability, latency vs. throughput, normalization vs. denormalization, push vs. pull, stateful vs. stateless.
- **Systems**: Kafka, Cassandra, DynamoDB, Postgres, Redis, Zookeeper, etcd, Flink, Spark, Kubernetes, Envoy, etc.
- **Case studies**: how companies at scale (Google, Amazon, Netflix, Uber, LinkedIn, etc.) solved specific problems.

When ingesting a source, always look for: which patterns does it illustrate? Which systems does it cover? What trade-offs does it highlight? What can be extracted as a case study?

## index.md format

```markdown
# Wiki Index

Last updated: YYYY-MM-DD | N sources | N pages

## Concepts
- [[concept-name]] — one-line summary

## Systems
- [[system-name]] — one-line summary

## Trade-offs
- [[trade-off-name]] — one-line summary

## Case Studies
- [[case-study-name]] — one-line summary

## Sources
- [[source-slug]] — Article Title (YYYY-MM-DD)
```

## log.md format

Append-only. Each entry prefixed with `## [YYYY-MM-DD] <type> | <title>` for grep-ability.
```
grep "^## \[" wiki/log.md | tail -10
```
