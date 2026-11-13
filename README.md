# Personal Knowledge Base

This folder is an Obsidian-compatible, LLM-maintained knowledge base.

## Structure

- `raw/` — immutable source material: clipped articles, papers, meeting notes, and imported documents.
- `wiki/` — synthesized pages maintained by the LLM.
- `index.md` — the content-oriented catalog of wiki pages.
- `log.md` — the chronological record of ingest, query, and lint operations.
- `AGENTS.md` — the operating schema for an LLM agent working in this vault.

## First-time setup

1. Open this `knowledge-base` folder as a separate vault in Obsidian.
2. Install Obsidian Web Clipper if you want to collect web pages.
3. Configure the clipper destination to `raw/sources/`.
4. Configure Obsidian attachment storage to `raw/assets/`.
5. Open the workspace in your LLM coding agent and point it at `knowledge-base/AGENTS.md`.
6. Add one source at a time to `raw/` and ask the agent to ingest it.
7. Review the generated wiki changes, then commit them if this folder is version-controlled.

## Standard commands to give the agent

### Ingest

> Ingest `raw/sources/<file>`. Read the complete source, create or update the relevant wiki pages, update `index.md`, and append a dated entry to `log.md`. Show me the proposed changes and unresolved contradictions before finalizing.

### Query

> Answer this question using the wiki: `<question>`. Start with `index.md`, follow relevant links, cite the source pages, identify uncertainty, and ask before creating a new permanent page.

### File an answer

> Answer the question and file the durable result as `wiki/<category>/<slug>.md`, then update `index.md` and `log.md`.

### Lint

> Lint the knowledge base. Find stale claims, contradictions, orphan pages, missing links, unsupported claims, and concepts that deserve their own page. Do not silently rewrite uncertain content; report proposed repairs.

## Suggested Obsidian plugins

Obsidian core graph view is enough to start. Add Dataview only if frontmatter-based indexes become useful, and add Marp only if you want to generate presentations from wiki pages.

