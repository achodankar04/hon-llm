---
title: "VoiceCheck microservices design (2026-11-20)"
type: source
status: active
created: 2026-11-21
updated: 2026-11-21
sources:
  - "[[sources/voicecheck-codebase-analysis-2026-11-20]]"
tags:
  - voicecheck
  - architecture
---

# VoiceCheck microservices design (2026-11-20)

## Origin

The Copilot agent wrote this design proposal on 2026-11-20 at the user's request. Its
title is "VoiceCheck: monolith → microservices target architecture", proposal v1.

- The full document is kept outside this vault, in the `Workspace_VoiceCheck`
  workspace at `docs/voicecheck-microservices-architecture.md`.
- It is based on [[sources/voicecheck-codebase-analysis-2026-11-20]] and on the
  Catalyst Fabric conventions of the `Workspace_VCNG7` services.

## Contents

- Seven services, the platform services they reuse, the design principles, and the
  boundaries deliberately kept together.
- PostgreSQL DDL per service and a mapping of all 88 legacy tables.
- The event catalogue, key flows and performance fixes.
- Data migration, a roadmap and open questions.

## Validation

- The DDL (52 tables, 45 indexes, 2 sequences) parses with pglast 7.24, which wraps
  PostgreSQL's own parser.
- It was not executed on a database, so semantic errors such as foreign-key or
  partition-key rule violations are not ruled out.
- The Mermaid diagrams were not rendered.
- The capacity targets are assumptions, not measured requirements.

## Limitations

- This is a proposal: nothing has been agreed or built.
- The wiki pages leave out the document's section on legacy security findings,
  because this repository is public.
