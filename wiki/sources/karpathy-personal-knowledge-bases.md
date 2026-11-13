---
title: "A pattern for building personal knowledge bases using LLMs"
type: source
status: active
created: 2026-11-13
updated: 2026-11-13
sources:
  - "https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f"
tags:
  - knowledge-base
  - llm
---

# A pattern for building personal knowledge bases using LLMs

Source: [Karpathy's gist](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)

## Summary

The document proposes replacing query-time-only RAG with a persistent, LLM-maintained wiki. Each new source is read once, synthesized into durable pages, cross-linked with existing knowledge, and recorded in a chronological log.

## Core model

- **Raw sources:** immutable articles, papers, notes, images, and data.
- **Wiki:** LLM-generated Markdown pages containing summaries, entities, concepts, comparisons, and evolving synthesis.
- **Schema:** agent instructions that define page structure, citations, ingest, query, and maintenance workflows.

## Operating loop

1. **Ingest:** read one or more sources, discuss important takeaways, create or update pages, update the index, and append to the log.
2. **Query:** search the index and linked pages, synthesize an answer with citations, and optionally file durable answers back into the wiki.
3. **Lint:** find contradictions, stale claims, orphan pages, missing links, unsupported concepts, and useful follow-up sources.

## Important design choices

- `index.md` is a content-oriented catalog used for navigation.
- `log.md` is chronological and append-only.
- Obsidian is a convenient editor and graph viewer, but the vault is just Markdown files.
- Git can provide history, branching, and collaboration.
- Search tooling such as qmd becomes useful only as the wiki grows; it is not required initially.
- Human responsibility is source curation, direction, review, and interpretation. The agent handles filing and maintenance.

## Limitations

This is a pattern rather than a turnkey implementation. Folder names, metadata, review rules, and tooling should be adapted to the user's domain. Generated synthesis must remain traceable to immutable sources and should not be treated as automatically correct.

