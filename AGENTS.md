# Knowledge Base Operating Schema

You maintain the `knowledge-base/` vault as a persistent, source-grounded wiki.

## Non-negotiable boundaries

1. Treat `raw/` as immutable. Never edit, rename, or delete a source.
2. Treat `wiki/` as a synthesis layer, not a replacement for sources.
3. Every material claim in a wiki page must link to one or more source pages or source files.
4. Do not hide uncertainty. Mark disputed, incomplete, inferred, and stale claims explicitly.
5. Do not invent citations, dates, quotations, people, or relationships.
6. Ask for confirmation before recording private, sensitive, legally significant, medical, financial, or high-impact personal information.
7. Keep edits narrowly scoped to the source or question being processed.
8. After any completed operation, update both `index.md` and `log.md`.

## Page conventions

Use this frontmatter for generated wiki pages:

```yaml
---
title: "Page title"
type: topic
status: active
created: YYYY-MM-DD
updated: YYYY-MM-DD
sources:
  - "[[sources/source-slug]]"
tags:
  - knowledge
---
```

Allowed `type` values are `source`, `topic`, `entity`, `project`, `decision`, `comparison`, and `question`.

Use stable lowercase kebab-case filenames. Prefer one idea per page. Link related pages with standard Obsidian links such as `[[topic-name]]`.

## Ingest workflow

When asked to ingest a source:

1. Confirm the source path and read it completely. If it is an image or scanned document, say what could not be read.
2. Create a source page in `wiki/sources/` with the source title, origin, date, concise summary, key claims, and limitations.
3. Identify entities, topics, projects, decisions, and open questions introduced or changed by the source.
4. Create missing pages only when they represent durable concepts; otherwise update existing pages.
5. Add backlinks from every touched page to the source page.
6. Preserve prior content that remains valid. When claims conflict, record both positions with dates and source links instead of choosing silently.
7. Update `index.md` with every created or materially changed page.
8. Append one entry to `log.md` using the required format.
9. Present a short change summary, unresolved contradictions, and follow-up sources for human review.

## Query workflow

When asked a question:

1. Read `index.md` first.
2. Follow the most relevant wiki links and inspect their cited source pages.
3. Separate sourced facts, synthesis, assumptions, and unanswered questions.
4. Cite pages inline with Obsidian links.
5. Do not create a page unless the user asks to file the answer or the result is clearly durable.
6. If filing the answer, use `wiki/questions/` or another appropriate category, then update the index and log.

## Lint workflow

Check for:

- claims without source links;
- pages with no inbound or outbound links;
- contradictory claims that are not marked;
- stale pages whose `updated` date predates newer relevant sources;
- important concepts mentioned repeatedly without a dedicated page;
- broken wiki links and duplicate pages;
- index entries missing for wiki files;
- log entries missing for completed ingest, query, or lint operations.

Report findings before making broad repairs. Make only deterministic repairs automatically; ask before resolving ambiguity.

## Activity log format

Append entries to `log.md` in this form:

```markdown
## [YYYY-MM-DD] ingest | Short title
- Input: `raw/...`
- Pages created/updated: [[...]]
- Notes: ...
```

